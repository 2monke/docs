# Custom Domains for Studios

Artists can connect a domain they own (e.g. `gloomcat.art`) to their studio so
visitors and customers reach the storefront on the artist's own address. The
platform never becomes a registrar: the artist buys the domain anywhere, types
it into Studio settings, and the platform attaches it to the platform's Vercel
project, which serves it with automatic TLS.

## How it actually works (and why "an IP address" is only half the story)

The app is deployed on Vercel. The platform does **not** own an IP for each
studio — Vercel owns the edge and the certificate. Attaching a domain means:

1. The platform adds the domain to **its** Vercel project (Domains API).
2. Vercel returns the records the artist must set at **their** registrar:
   - **Apex domain** (`gloomcat.art`): an `A` record to Vercel's anycast IP
     (DNS forbids a CNAME at the zone apex).
   - **Subdomain** (`shop.gloomcat.art`): a `CNAME` to `cname.vercel-dns.com`.
   - **Ownership TXT** (`_vercel.<domain>`): only when the domain is already
     claimed in another Vercel account.
3. Once DNS resolves correctly, Vercel verifies ownership, serves the domain
   and issues the TLS certificate automatically.

We never hardcode the IP/CNAME: the service stores and displays exactly what
Vercel's API returns for that domain (`recommendedIPv4` / `recommendedCNAME`),
so a Vercel-side IP change cannot break the instructions.

## What the app already did

Host resolution was built multi-tenant and already treats unknown registrable
domains as studio keys (`resolveStudioIdentityFromHost`), matches
`studios.customDomain` in `getStudio`, indexes custom domains in the tenant
matrix, reflects them in the Payload CORS policy, prefers them in
`getTenantUrl` (emails/links), and keeps auth cookies host-only on hosts
outside `COOKIE_DOMAIN` — so a customer can log in on the custom domain.

## How a custom host resolves to a tenant

`resolveStudioIdentityFromHost` returns a single label for platform hosts
(`bribabibi`) but the **full host** for a studio's own domain
(`bribabibi.shop`). Both shapes are handed to `getTenantByIdentity`, which
routes a dotted identifier to the `customDomain` index and never to the
subdomain index (studio handles are normalized to a dot-free label).

This is load-bearing for more than routing: the tenant id it returns is what
`cacheTags` namespaces every storefront read with. When a custom host failed to
resolve, `resolveTenant` echoed the host back as a tenant id
(`{ id: 'bribabibi.shop' }`), so page, header-nav, shop and queue queries all
filtered on a tenant that does not exist — the studio rendered its chrome but
no content, and no write could invalidate its cache. The studio lookup is a
single function: `getStudio(identity)` accepts an ID, a handle or a custom-
domain host, and its cached storefront twin is `getCachedStudio(identity?)`.
Callers never need to know which host shape they hold.

## Media uploads and serving

- **Buckets** — public media (`media/`) lives in `NEXT_PUBLIC_S3_BUCKET` and is
  served by CloudFront. Purchased downloads (`digital-assets/`) and commission
  deliverables (`raw-media/`) live in `S3_PRIVATE_BUCKET`, with no public
  policy and no public CDN behaviour.
- **Uploads** — commission reference sheets go browser→S3 with a presigned PUT
  (`src/features/commissions/services/commission.service.ts`); the signing route
  is same-origin, so studio hosts work unchanged. The **public** bucket's CORS
  uses `AllowedOrigins: ["*"]` for arbitrary studio domains. The **private**
  bucket also allows arbitrary origins for `PUT`/`POST` (artists upload
  deliverables from the queue on their custom domain), but **not `GET`**: no
  private presigned GET URL is ever minted, and app streaming is used instead,
  so an origin cannot read private content even if a request URL leaks. Do not
  append each custom domain to either bucket at registration time — presigned
  URLs are the upload authorization, and S3 CORS size limits make per-tenant
  edits brittle.
- **CORS is not object access** — `AllowedOrigins: ["*"]` lets a browser
  origin *read a response it is already authorized to request*. It does not list
  the bucket, does not grant unsigned PUTs, and cannot read a private object
  without a valid signed URL. Private assets stay private through the private
  bucket policy + `/api/files` / `/api/deliverables` (purchaser/tenant checks
  followed by a same-origin byte-range stream), not through CORS.
- **No presigned download URLs** — archive downloads stream through the app as
  well. A presigned GET would be a bearer capability that survives in browser
  history, referrers and logs for its TTL; the private routes never issue one.
  This costs server bandwidth for large files, but is the only way to keep a
  leaked URL from being reusable.
- **Fonts** — `@font-face` is CORS-protected. The public media CloudFront
  distribution's response-headers policy must return
  `Access-Control-Allow-Origin` for font files; the S3 bucket CORS alone is not
  enough unless CloudFront forwards `Origin` and passes S3's headers through.
  Without it, uploaded fonts fail on custom domains even though images load.
- **Serving** — `getCdnUrl` returns an absolute CloudFront/S3 URL for public
  media (relative `/api/media/file/...` without S3). Raw media and digital
  assets always return `/api/deliverables/<id>` or `/api/files/<id>`, so images
  and uploaded fonts render on a custom domain through the app or the public
  CDN. Public raw-media rows get a shared-cache header and a high public
  rate-limit ceiling; they still stream through the app, so use `media` when a
  public asset must sit behind CloudFront. CSP allows the CDN/S3 origins in
  `img-src`, `font-src` and `connect-src`; a non-CloudFront
  `NEXT_PUBLIC_CDN_URL` origin is added automatically by
  `src/lib/security/csp.js`.
- **Server Actions** — Next compares `Origin` with `x-forwarded-host`/`Host`,
  and the proxy pins that header, so custom domains pass without appearing in
  `experimental.serverActions.allowedOrigins`; that list only covers origins
  that differ from the host (dev tunnels / reverse proxies).

## Architecture added for the lifecycle

- **`studio-domains` collection** — one row per studio (unique `domain`, and `tenant` is unique too, so "one domain per studio" is enforced by the database, not just by the action pre-check), status `pending_dns | active | error`, the stored
  DNS snapshot, a verified flag and the last check time. Admin-only writes;
  artists read their own row through the UI action.
- **`Studios.customDomain`** — the **active only** denormalized domain. It is
  written by the domain flow after Vercel verifies ownership and DNS, and is
  `adminOnlyUpdate` at the field level. Routing, the tenant matrix, CORS and
  outbound links therefore never see a half-configured host.
- **Server actions** (`src/features/studio-domains/actions`) — request /
  refresh / remove. The studio is resolved from the session tenant (never a
  client-supplied id), the Vercel call happens server-side, writes use
  `overrideAccess`, and every action is rate-limited (`createSafeLimiter`).
- **Vercel service** (`src/features/studio-domains/services`) — attach, read,
  verify, remove, config. Typed failures; 404 means "not attached"; the token
  only ever travels in the `Authorization` header.
- **Validation** (`src/lib/tenant/custom-domain.ts`) — normalization (scheme,
  path, port, trailing dot, `www.`, IDN → punycode) and rejection of
  platform-owned hosts, IPs, reserved dev suffixes and malformed labels. Enforced
  as a Payload field validator, so every write path gets the same rules.
- **Lifecycle rules** (`src/features/studio-domains/lib/domain-status.ts`) —
  `active` only when `verified && !misconfigured`; the DNS record projection
  shown to the artist.
- **UI** — “Custom Domain” card in Studio settings → Studio Brand: enter the
  domain, copy the records, check status, remove. The card shows the server's
  state; it never guesses.
- **Canonical URLs** (`src/lib/tenant/canonical.ts`) — the canonical origin is
  the active custom domain when present, else the studio subdomain, else the
  platform root. `metadataBase` is per-request now; sitemap/robots already
  resolve per host.
- **Custom domain priority & takeover** (`src/lib/tenant/custom-domain.ts`) — when
  a studio has both a custom domain and a subdomain registered, the active custom
  domain takes priority over the platform subdomain. Incoming storefront requests
  arriving on the studio's platform subdomain (`gloomcat.2monke.com`) redirect
  (308, `permanentRedirect`) to the custom domain (`artist.com`), preserving
  pathname and search queries. Localhost development and preview requests (a
  live-preview iframe or a valid preview capability token) are exempt.
- **Admin host policy** — `/admin` on a custom domain 308s to the platform
  host’s admin (one admin origin keeps sessions/CSRF/CORS simple).
- **Studio deletion** — detaches the domain from Vercel and deletes the rows
  (best-effort; never blocks the delete).

## Environment

```bash
VERCEL_API_TOKEN=      # team-scoped token with project access
VERCEL_PROJECT_ID=     # project that serves the app (id or name)
VERCEL_TEAM_ID=        # optional, when the token/project are team-scoped
```

When these are absent the feature stays off: the card renders a short
“not enabled on this deployment yet” note and every action fails closed with a
clear message.

## Status model

- `pending_dns` — attached to the project, waiting for the artist's DNS (and/or
  the ownership TXT). The domain is **not** used in links or emails.
- `active` — ownership verified and DNS configured; mirrored onto
  `Studios.customDomain`; canonical origin switches to it.
- `error` — the domain was detached from the project outside our flow, or
  Vercel rejected it (e.g. claimed by another Vercel account). The artist sees
  an actionable message.

If an active domain stops resolving (DNS removed), the next status check
demotes it and clears the mirror so emails never point at a host that stopped
serving. A **transient failure to read Vercel's config is not a demotion**: the
last known status, mirror and DNS snapshot stay untouched (only `lastCheckedAt`
moves) and the artist gets a “status unchanged” message — only an explicit
`misconfigured: true` from Vercel demotes the domain.

## Migration & rollout

1. The migrations are checked in — `20261020_121752_add_studio_domains`
   (table) and `20261020_123306_studio_domains_tenant_unique` (one row per
   studio). Apply them where Postgres is reachable:
   ```bash
   yarn migrate
   ```
   (The local dev DB has already had both applied during this implementation; its
   two older push-managed migrations were recorded as applied so the runner has
   no pending work besides these.)
2. Set the Vercel env vars in production (and preview if you want to test).
3. Add one real test domain to a preview project, set DNS at a registrar you
   control, and watch the status reach `active`.
4. Confirm TLS issues automatically and that `/admin` on the custom host
   forwards to the platform admin.

## Local testing

`NEXT_PUBLIC_ROOT_DOMAIN`/platform roots are recognized; any other host
resolves as a custom-domain key. To simulate one locally, map a real-looking
domain to loopback (`/etc/hosts`: `127.0.0.1 gloomcat.art`) and visit
`http://gloomcat.art:3000` — the studio whose `studios.customDomain` is
`gloomcat.art` renders. The Vercel API is not configured locally, so the card
will report the feature as unavailable.

## Deliberately not in v1

- In-app domain purchase (registrar API, payments, registrant data, renewals).
  Artists buy elsewhere; the platform only attaches and verifies.
- More than one custom domain per studio.
- Attaching the `www` alias with a redirect (the apex is attached; `www` input
  is normalized to the apex).
- A scheduled reconciliation cron (status checks are on-demand from the card);
  a daily job can reuse `getProjectDomain`/`getDomainConfig` later.
- Expiring stale `pending_dns` claims. Until then, an artist who abandons a
  claim keeps the domain reserved; removing the row in the admin releases it.
