---
status: shipped
collections: [artists, customers, pages, products, studios]
features: [studios]
---

# Studio Draft Visibility Guard

Status: shipped in PR #66 (`studio-visibility`), extended with preview capability tokens.
Owner surface: storefront rendering, robots.txt, sitemap.xml.

## The promise

A studio whose `profileStatus` is `draft` is **not publicly observable**. Its
subdomain renders a generic "This Studio Is Not Published Yet" placeholder for
anonymous visitors instead of the storefront, and search engines are told to
stay away. A studio becomes `published` when Stripe reports
`charges_enabled` (see `src/features/studios/lib/profile-status.ts`), so
publication is an outcome of connecting payouts, not a manual toggle.

## Where the gate lives

- **Pure rule** — `src/features/studios/lib/studio-visibility.ts`
  (`canViewStudio`). No I/O; the whole verdict matrix is unit-testable.
- **Request wrapper** — `src/lib/tenant/studio-visibility-guard.ts`
  (`getStudioVisibility`, memoized per request with React `cache`). Resolves the
  host studio, the session user, the preview verdict and the preview capability,
  then calls the rule.
- **Enforcement point** — `src/app/(frontend)/(studio)/layout.tsx`. Every route
  in the `(studio)` group renders through this layout, so placement is the
  mechanism: a new storefront page gets the guard by existing. When blocked, the
  layout renders `UnpublishedStudio` (no studio chrome, no studio data) and the
  layout's `generateMetadata` emits `noindex, nofollow`.
- **SEO surfaces** — `src/app/robots.ts` (draft host: `Disallow: /` even in
  production), `src/app/sitemap.ts` (draft host: empty sitemap). Both read the
  studio through `getCachedStudio` — the same cached read the
  storefront uses — so publication flips invalidate everything together.

## Verdict matrix (`canViewStudio`)

- Published studio → allowed for everyone.
- Unregistered subdomain → passes through (the pages themselves 404).
- Draft studio, anonymous, content route → **blocked**.
- Draft studio, owner artist or platform admin session (`userHasTenantAccess`)
  → allowed.
- Draft studio, live preview **carrying a valid capability token** → allowed.
- Draft studio, auth-flow route (`/login`, `/register`, `/forgot-password`,
  `/reset-password`, `/verify` and subpaths) → allowed, so an owner can always
  sign back in on their own host.
- Missing pathname → fail closed (no exemption).

## Preview capability tokens (`src/lib/theme/preview-access.ts`)

A bare `?preview=true` is NOT sufficient to render a draft studio. The proxy
mints `x-is-live-preview` from client-controlled input (a preview query param or
an admin-referred iframe — see `detectPreviewRequest`), so that verdict alone was
a public bypass: anyone could append `?preview=true` (or `?draft=`) to a draft
subdomain and read the storefront.

The fix layers a capability on top:

- **Minting** — the admin's preview URL builders (`getDynamicLivePreviewUrl`,
  `getThemeLivePreviewUrl` in `src/lib/theme/livePreviewUrl.ts`) sign
  `HMAC-SHA256(studioId.expiry)` with `PAYLOAD_SECRET` and append it as
  `?previewToken=`. Builders that cannot resolve the studio id mint nothing —
  fail closed.
- **Transport** — `src/proxy.ts` overwrites any client-supplied
  `x-preview-token` header with the query param, so the node-side reader has a
  single source of truth. The proxy never verifies (edge-safe code stays free of
  `node:crypto`).
- **Verification** — the guard verifies against the *resolved studio's* id
  before allowing a draft preview. Wrong studio, tampered signature, malformed
  token, missing secret and expiry all fail closed.
- **TTL** — 24 hours. There is no per-token revocation; rotating
  `PAYLOAD_SECRET` invalidates every outstanding token.

The token is a **bearer capability in the URL**: whoever holds a fresh preview
link can view that one studio's draft for the token's lifetime. That is the
intended "preview link" model, and it preserves the anonymous guest view — the
proxy still scrubs the session for storefront previews, so artists see what
customers see.

Why not "draft previews require an owner session instead"? Because the preview
pipeline deliberately strips the session on storefront paths (anonymous preview,
see `src/lib/theme/live-preview-headers.ts`); requiring a session would either kill the
guest view or force every preview entry point — including client-built fallback
URLs — to carry `auth=true`. The token keeps all four preview entry points
working through one server-side change.

## Stripe demotion trade-off

`profileStatus` is derived from Stripe. A studio that stops charging — flagged
account, disconnected payouts — falls back to `draft` and its storefront goes
dark for the public until Stripe reports `charges_enabled` again. That is
deliberate: the platform never routes money for a studio that cannot receive it.
Owners and admins keep full access throughout, and the placeholder tells the
owner to sign in.

## HTTP and SEO semantics

The placeholder renders with **HTTP 200** plus `noindex, nofollow` metadata, a
`Disallow: /` robots.txt on the host, and an empty sitemap. The status is 200 on
purpose: the page is a real, owner-actionable surface (it links to sign-in), and
the crawler-facing guarantees are carried by robots metadata rather than status
codes. If a future requirement is "drop draft URLs from indexes entirely", the
place to change is the layout's `notFound()` call — not the rule.

## Known boundaries (deliberate, documented)

- **The gate is a storefront/SEO gate, not a data-access gate.** The `(studio)`
  layout is the only enforcement point, so `/api/*` REST and GraphQL still serve
  draft-studio documents to anyone who asks with the right ids. Tenant ids are
  opaque prefixed ids (`std_…`) but are discoverable through public reads of
  tenant collections (e.g. anonymous `GET /api/products` returns tenant ids).
  `studios` itself is owner/admin-read only, so subdomain→id resolution is not
  anonymous. If the product ever needs draft documents to be unreadable at the
  data layer, the enforcement belongs in collection access rules
  (`src/access/ownTenantOnly.ts` and per-collection `read`), not here.
- **Client-built preview fallbacks cannot mint tokens.**
  `MobileLivePreviewBridge` constructs preview URLs as a last resort when no
  server-built preview link exists in the admin DOM; for draft studios that
  fallback shows the placeholder. The primary paths (preview button, external
  link, iframe src — all server-built) carry the token.
- **Onboarding lands artists on the current origin's `/admin`.** The
  cross-subdomain session-transfer bounce was removed because staging without
  wildcard DNS sent artists to production. On production this means a new artist
  starts in the platform-root admin (tenant selection included) instead of their
  studio subdomain; `/dashboard` still routes artists to their studio subdomain
  via `requireDashboardAccess`.

## Testing

- `tests/int/studio-visibility.int.spec.ts` — rule matrix, guard wrapper
  (bare preview blocked, valid token allowed, foreign/tampered/expired token
  blocked, spoofed embed blocked), robots/sitemap surfaces.
- `tests/int/preview-access.int.spec.ts` — token sign/verify semantics
  (binding, tamper, expiry, fail-closed).
- `tests/int/proxy-anonymous-preview.int.spec.ts` — token forwarding: query
  param wins, client-supplied header dropped.
- `tests/e2e/studio-visibility.e2e.spec.ts` — live server: placeholder for
  anonymous visitors, bare preview params still blocked, capability token
  unlocks, owner session renders, published studio unaffected.
