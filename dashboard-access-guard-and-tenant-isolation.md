---
status: shipped
collections: [artists, commissions, orders]
features: []
---

# Centralized Dashboard Access Guard, Session Bridge & Multi-Tenant Isolation

How ArtPack protects `/dashboard` and its sub-routes with a centralized server-side authorization guard, eliminates magic query numbers, and provides seamless cross-subdomain session transfers across root platform and artist studio subdomains.

## Architecture

```
                                    ┌──────────────────────────────┐
                                    │    Incoming HTTP Request     │
                                    │      (/dashboard/*)          │
                                    └──────────────┬───────────────┘
                                                   │
                                                   ▼
                                    ┌──────────────────────────────┐
                                    │    requireDashboardAccess    │
                                    │  (src/lib/dashboard-guard)   │
                                    └──────────────┬───────────────┘
                                                   │
                   ┌───────────────────────────────┴───────────────────────────────┐
                   │                                                               │
                   ▼                                                               ▼
       [Platform Root Domain]                                          [Artist Subdomain]
    (artpack.me / localhost:3000)                               (e.g. gloomcat.artpack.me)
                   │                                                               │
         ┌─────────┴─────────┐                                           ┌─────────┴─────────┐
         │                   │                                           │                   │
         ▼                   ▼                                           ▼                   ▼
    [Is Artist]        [Is Customer]                                [Is Artist]        [Is Customer]
         │                   │                                           │                   │
  Has Studio?           Allow Access                               Owns Tenant?         Allow Access
  ├── Yes: Redirect    (Filter: customer = user.id)                ├── Yes: Filter      (Filter: tenant = currentArtist.id
  │   via Session                                                  │   tenant = ID       AND customer = user.id)
  │   Transfer Bridge                                              └── No: Redirect
  └── No: Redirect                                                     to own studio
      to Platform Home (/)
```

---

## Key Components

### 1. Cross-Subdomain Session Bridge (`src/app/api/auth/session-transfer/route.ts`)

Bridges authenticated sessions across subdomain boundaries in local development (`*.localhost:3000`) and production:

- Accepts `GET /api/auth/session-transfer?token=<jwt>&redirect=<path>`.
- Validates the token against Payload CMS (`payload.auth`).
- Sets `payload-token` and `payload-tenant-context` cookies on the target subdomain origin.
- Issues an HTTP 307 redirect to the destination path with full session state.

**Redirect hardening (2026-09-19).** The route used to carry its own
`sanitizeRedirectPath`, which rejected backslashes, `//` and `scheme:` prefixes
but not **control characters**. `searchParams.get('redirect')` decodes `%09` to
a literal tab, and the WHATWG URL parser strips tabs/newlines before parsing —
so `?redirect=/%09/evil.com` passed the guard and then resolved to
`//evil.com`: an open redirect on an endpoint that installs a session cookie,
exploitable with any valid transfer token (i.e. any self-registered account).

The route now uses `getSafeRedirectUrl` from `src/lib/security/redirect.ts` — the shared
sanitizer already backing login, register, reset/forgot password, verify,
social-confirm and `payload-sync` — which rejects `[\u0000-\u001F\u007F]`,
protocol-relative and backslash forms, `/_next/*` targets and over-long values.
The failure path no longer echoes the error message back through an
`x-transfer-error` response header (logged server-side only) so the endpoint
cannot be used as a token-probing oracle.

Regression coverage: `tests/int/safe-redirect.int.spec.ts` (sanitizer case
table, including the decoded `%09` payload) and
`tests/int/session-transfer-route.int.spec.ts` (the route never leaves the
origin, installs no cookie on failure, and leaks no error header).
`scripts/smoke-session-transfer.sh` exercises the live hop end to end.

**Customer "Visit Studio" links (2026-11-02).** The platform dashboard groups a
customer's purchases and requests by studio and offers a "Visit Studio" button
per group. Sessions are per-host cookies (and a custom domain cannot share one
at all), so a plain storefront link opened the studio as a guest — reported as
"I'm logged in on the platform, but the studio site says I'm not".

The button is now a static platform gateway link,
`/api/auth/visit-studio?tenant=<id>`
(`src/app/api/auth/visit-studio/route.ts`), and the gateway decides at **click
time** — after reading the platform `payload-token`:

- guest → the studio storefront, no token minted;
- signed in → a 120-second transfer token → the studio's
  `/api/auth/session-transfer`, which installs the real session cookie.

There is deliberately no “cookie already shared” shortcut: the request cannot
prove the domain scope of the `payload-token` it carries (the browser sends only
name/value), so a host-only cookie would silently reintroduce the guest bug.
Minting is one sign + one session lookup per click.

The studio row supplies the subdomain/custom domain (the platform base domain
comes from the request host); a caller-supplied `host` is never read (that would
turn the route into a token-exfiltration gadget). `to` goes through
`getSafeRedirectUrl`, which also refuses `/api/*` targets so a redirect cannot
chain into another credential-installing endpoint. Both redirect responses are
`Cache-Control: no-store` with `Referrer-Policy: no-referrer`.

This replaced `buildStudioVisitUrls`, which minted during render and baked a
15-minute token into every section header's `href`: it expired in an open
library tab, put a credential in the DOM for a click that might never happen,
and forced a `visitUrls` map through `PlatformLibraryView` /
`PlatformCommissionsView`. Those views now render a plain link; nothing
server-only is imported from the client-reachable barrels.

Regression coverage: `tests/int/platform-visit-studio.int.spec.ts` (per-host
destination, custom domain, host-injection and unsafe-`to` rejection, `COOKIE_DOMAIN`
still minting, mint-failure fallback, and the rendered href shape),
`tests/int/session-transfer-route.int.spec.ts` (the transfer-claim gate),
`tests/int/safe-redirect.int.spec.ts` (the `/api/*` rejection), and
`scripts/smoke-session-transfer.sh` (the live gateway → transfer hop, using a
minted transfer token rather than a session JWT).

**Transfer tokens are typed.** `mintSessionTransferToken` adds
`tokenType: 'session-transfer'`, and `/api/auth/session-transfer` rejects any
`token` without it before validating the session. A full 7-day session JWT in a
URL would otherwise install that account in the visitor's browser — a
login-CSRF/replay primitive. The `studios` hub now mints a transfer token for
its cross-host hops instead of forwarding the session token.

**Still open (deliberately deferred):** the transfer token is a bearer
credential — now minted with a 120-second TTL at click time — and is not
single-use, so a captured URL still yields a session until it expires; the
endpoint has no rate limiting; and `getRequestBaseUrl` trusts `x-forwarded-host`
unconditionally.

---

### 2. Centralized Access Guard (`src/lib/security/dashboard-guard.ts`)

The `requireDashboardAccess()` function is the single source of truth for dashboard authorization:

- **Authentication Validation**: Retrieves and parses session via `getCachedUser(token)`. Automatically issues a `redirect('/login')` if unauthenticated.
- **Domain Context Resolution**: Analyzes the hostname via `getStudioIdentityFromRequest()`.
- **Root Domain Routing**:
  - If an **Artist** accesses `/dashboard` on the root domain, it resolves their studio subdomain (via `user.tenants` or `artists.user = user.id`) and redirects them to their dedicated studio URL (`[subdomain].artpack.me/dashboard`) via the session transfer bridge.
  - If the artist has no studio created, it routes them to the platform home page (`/`).
  - If a **Customer** accesses `/dashboard` on root, they are granted access to view their orders and commissions across all artists (`scopedCustomerId = user.id`).
- **Studio Subdomain Isolation**:
  - If an **Artist** accesses a specific studio subdomain (e.g. `gloomcat`), it verifies ownership via `userHasTenantAccess(user, currentArtist.id)`. If unauthorized, they are safely redirected away.
  - If a **Customer** accesses an artist's studio dashboard, queries are strictly dual-scoped to that tenant and their customer ID (`scopedTenantId = currentArtist.id` AND `scopedCustomerId = user.id`).

```typescript
export interface DashboardAccessContext {
  user: AuthenticatedUser
  isArtist: boolean
  isRootDomain: boolean
  currentArtist: Artist | null
  scopedTenantId?: string | number
  scopedCustomerId?: string | number
}
```

---

### 3. Layout-Level Guard Execution (`src/app/(app)/dashboard/layout.tsx`)

Executes `requireDashboardAccess()` at the top of `DashboardLayout`:
- Any redirection (e.g. artist root to studio subdomain) occurs immediately at the layout level as an **HTTP 307 redirect**, completely eliminating the flash of un-themed platform templates.

---

### 4. Clean Commission Querying (`src/app/(app)/dashboard/DashboardMainContent.tsx`)

Replaced ad-hoc query conditions and magic numbers (`tenant: { equals: -999999 }`) with strongly-typed filters supplied directly by the access guard:

```typescript
const access = await requireDashboardAccess()
const { isArtist, scopedTenantId, scopedCustomerId } = access

const whereClause: Where = {
  ...(scopedTenantId ? { tenant: { equals: scopedTenantId } } : {}),
  ...(scopedCustomerId ? { customer: { equals: scopedCustomerId } } : {}),
  ...(status && status !== 'all' ? { status: { equals: status } } : {}),
  ...(type && type !== 'all' ? { commissionType: { equals: type } } : {}),
}
```

---

## Security & Isolation Matrix

| Persona | Location | Action / Result | Scoped Filter |
| :--- | :--- | :--- | :--- |
| **Unauthenticated** | Any `/dashboard` | Redirected to `/login` | `N/A` |
| **Artist (Owner)** | Own Subdomain `/dashboard` | Granted access | `tenant: currentArtist.id` |
| **Artist (Non-Owner)** | Foreign Subdomain `/dashboard` | Redirected to `/login` / own studio | `N/A` |
| **Artist** | Root Platform `/dashboard` | Redirected to studio dashboard via Session Bridge | `N/A` |
| **Customer** | Root Platform `/dashboard` | View all placed commissions | `customer: user.id` |
| **Customer** | Artist Subdomain `/dashboard` | View commissions for this artist only | `customer: user.id AND tenant: currentArtist.id` |
| **Admin** | Any Storefront `/dashboard` | Strictly scoped to current tenant (No global dump) | `tenant: currentArtist.id` |

---

## Debug & Troubleshooting Reference

### 1. Session Transfer Token Verification
- Accessing `/api/auth/session-transfer?token=...` directly verifies that `payload.auth` succeeds and sets the `payload-token` cookie for the active host.

### 2. Cookie Domain Configuration
- In production, ensure `COOKIE_DOMAIN=.artpack.me` is set so cookies naturally apply across wildcards.
- In local development, the `/api/auth/session-transfer` bridge handles the cross-origin cookie assignment automatically.

### 3. Testing Local Multi-Tenant Isolation
- Run the test suite:
  ```bash
  npx tsc --noEmit
  npm run test:int
  npx playwright test tests/e2e/artist-onboarding.e2e.spec.ts
  ```
