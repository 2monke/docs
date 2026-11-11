---
status: shipped
collections: [artists, commissions, customers, queue, sprints, studios]
features: []
---

# Platform "My Studios" Hub & Superadmin CMS Segregation

How ArtPack segregates artist creator workspace management from low-level Payload CMS database tables, providing a multi-tenant studio hub at `/studios`, smart subdomain routing, and strict superadmin-only access to `/admin`.

---

## Architecture

```
                                      ┌──────────────────────────────┐
                                      │    Incoming HTTP Request     │
                                      └──────────────┬───────────────┘
                                                     │
                     ┌───────────────────────────────┴───────────────────────────────┐
                     │                                                               │
                     ▼                                                               ▼
         [Platform Root Domain]                                          [Artist Subdomain]
      (artpack.me / localhost:3000)                               (e.g. gloomcat.artpack.me)
                     │                                                               │
         ┌───────────┼───────────┐                                                   │
         │           │           │                                                   ▼
         ▼           ▼           ▼                                       ┌───────────────────────┐
     GET /login  GET /studios GET /admin                                 │  Studio Dashboard     │
         │           │           │                                       │     (/dashboard)      │
         ▼           ▼           ▼                                       └───────────┬───────────┘
   loginAction   "My Studios"  Users.access.admin                                    │
         │           Hub      (roles = ['admin'])                                    ├── Commissions & Queue
         │           │           ├── Superadmin: Allow Payload CMS                   ├── Theme & Customizer
         │           │           └── Artist: Access Denied / Redirect                ├── Pricing & Sprints
         │           │                                                               └── Stripe Connect Payouts
         └───────────┴───────────────────────────┐
                                                 │
                                                 ▼
                              ┌─────────────────────────────────────┐
                              │     Session Transfer Bridge         │
                              │  (/api/auth/session-transfer?token) │
                              └──────────────────┬──────────────────┘
                                                 │
                                                 ▼
                                     [Artist Studio Dashboard]
                                 (https://[subdomain].artpack.me)
```

---

## Key Components

### 1. "My Studios" Creator Hub (`src/app/(frontend)/studios/page.tsx`)

The centralized platform portal where artists manage and switch between their creator storefronts:

- **Authentication & Role Check**: Validates `payload-token` cookie via `getCachedUser(token)`. Redirects non-artists or unauthenticated visitors.
- **Tenant Aggregation**: Queries all `artists` records where `user = user.id` OR `id IN user.tenants`.
- **Studio Cards**: Renders interactive cards featuring:
  - Studio Name, Initial Avatar, and Subdomain
  - Theme color swatches (`primaryColor`, `backgroundColor`)
  - "Open Studio Dashboard" button with automatic cross-subdomain session transfer
  - "View Public Storefront" link opening the live storefront in a new tab
  - "+ Launch a New Studio" card linking to the 5-step AI onboarding flow (`/onboarding`)

---

### 2. Multi-Studio Smart Routing (`src/lib/security/dashboard-guard.ts`)

The centralized authorization guard automatically differentiates between single-studio and multi-studio artists:

```typescript
// If artist owns multiple studios, send them to the Studios Hub
if (allowedTenants.length > 1) {
  redirect('/studios')
}

// Single-studio artist: directly forward to their studio subdomain
if (artistSubdomain) {
  const targetUrl = isLocal && token
    ? `${protocol}://${artistSubdomain}.localhost${port}/api/auth/session-transfer?token=${encodeURIComponent(token)}&redirect=/dashboard`
    : `${protocol}://${artistSubdomain}.${host.split('.').slice(-2).join('.')}${port}/dashboard`

  redirect(targetUrl)
}
```

---

### 3. Superadmin CMS Isolation (`src/collections/Users.ts`)

Prevents creators and customers from accessing low-level developer database tables:

```typescript
export const Users: CollectionConfig = {
  slug: 'users',
  access: {
    // Restrict Payload CMS Admin Panel strictly to superadmins
    admin: ({ req: { user } }) => Boolean(user?.collection === 'users' && user?.roles?.includes('admin')),
    read: adminOrSelf,
    create: isAdmin,
    update: adminOrSelf,
    delete: isAdmin,
  },
  ...
}
```

---

### 4. Cross-Subdomain Session Bridge (`src/app/api/auth/session-transfer/route.ts`)

Ensures seamless transition from the platform root (`artpack.me` or `localhost:3000`) to tenant subdomains without re-prompting for credentials:

- Accepts `GET /api/auth/session-transfer?token=<JWT>&redirect=<path>`.
- Validates token against `payload.auth()`.
- Sets `payload-token` and `payload-tenant-context` cookies directly on the destination origin.
- Performs an HTTP 307 redirect into the requested studio dashboard.

---

## Routing & Persona Access Matrix

| Route | Persona | Action / Outcome |
| :--- | :--- | :--- |
| **`artpack.me/`** | Any | Public Platform Landing Page (`Create Studio` CTA) |
| **`artpack.me/login`** | Artist | Logs in & redirects to `[subdomain].artpack.me/dashboard` |
| **`artpack.me/dashboard`** | Single Studio Artist | Auto-forwards to `[subdomain].artpack.me/dashboard` |
| **`artpack.me/dashboard`** | Multi-Studio Artist | Auto-forwards to `/studios` |
| **`artpack.me/studios`** | Artist | Renders "My Studios" Hub |
| **`artpack.me/admin`** | Superadmin (`roles: ['admin']`) | Accesses Payload CMS Collection Explorer |
| **`artpack.me/admin`** | Artist (`roles: ['artist']`) | Access Denied / Blocked from developer tables |
| **`[subdomain].artpack.me/dashboard`** | Tenant Owner | Full Creator Dashboard (Commissions, Queue, Shop, Theme) |

---

## Debug & Troubleshooting Reference

### 1. Artist lands on login instead of studio dashboard
- **Check**: Verify that the URL passed through `/api/auth/session-transfer?token=...`.
- **Check**: Verify that `COOKIE_DOMAIN` is set in production (`COOKIE_DOMAIN=.artpack.me`).

### 2. Multi-studio artist not seeing all their studios
- **Check**: Verify that `Users.tenants` array contains all assigned artist tenant IDs and that `saveToJWT: true` is enabled on the `tenants` field in [`Users.ts`](file:///Users/eugenepucko/Projects/artpack/src/collections/Users.ts).

### 3. Superadmin cannot log into `/admin`
- **Check**: Ensure the user record in `users` collection has `roles` containing `'admin'`.
