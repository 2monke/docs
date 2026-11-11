---
status: shipped
collections: [announcements, artists, media, orders, pages, products, queue, sprints, studios]
features: [queue, studio-settings, studios]
---


# Queue Route Latency Variance & Live-Preview Flakiness (Diagnosis + Implementation)

Status: **fixes implemented** (2026-09-18). Diagnosis kept below with corrections
marked inline where the original reading did not match the code.

Changed files:

- `src/collections/hooks/revalidatePagesCache.ts` + `src/lib/blocks/page-revalidation.ts` — no-op save gate
- `src/features/studios/cache/studios.cache.ts`, `src/features/queue/cache/queue.cache.ts` — `cacheLife('seconds')` → `minutes`
- `src/lib/matrix/state.ts` — dev L1 TTL + `MATRIX_SKIP_L2`
- `src/lib/tenant/subdomain.ts` — deduped resolution log
- `src/lib/tenant/cors-origins.ts` (new) + `src/app/(payload)/api/[...slug]/route.ts` — allowlisted CORS reflection
- `src/components/BlocksRenderer.tsx` — admin-origin resolution, admin-row block indexes
- `src/lib/blocks/effective-blocks.ts` — indexed block resolver
- `src/lib/tenant/tenant-tag.ts` — root-exempt revalidations for page/pages/queue/sprints/announcements; layout bundle registers `queue`
- Tests: `tests/int/effective-blocks.int.spec.ts`, `tests/int/cors-origins.int.spec.ts`, `tests/int/page-revalidation.int.spec.ts`, `tests/int/tenant-tag-revalidate.int.spec.ts`

## Part 1 — Why /queue latency sawtooths

The route is a server component layered over several caches with different lifetimes.
Every slow render is a "cold pass" through one or more of them.

### Cache layers in play (per request)

- **Page doc**: `fetchPageBySlug` in `src/features/studios/cache/studios.cache.ts`
  — `use cache` + `cacheLife('hours')`. Usually a hit.
- **Studio/artist**: `fetchStudio` (same file) — was `cacheLife('seconds')`,
  runs `getStudio(identity, { depth: 2 })`. This is the heavy
  query: populates owner, artists, theme, `commissionSettings` (pricing tiers →
  example images → media), logo uploads, etc. Expired within seconds.
- **Queue + sprints**: `src/features/queue/cache/queue.cache.ts` — was `cacheLife('seconds')`.
- **Layout**: `getEffectiveTheme()` (`src/features/studio-settings/actions/get-effective-theme.action.ts`)
  → `getCachedLayoutData` — was also `cacheLife('seconds')` and additionally runs
  announcements + a sprints query + a queue query on expiry, on top of the page's own needs.
- **Subdomain → tenant matrix**: L1 in-memory with `L1_TTL_MS = 30_000`
  (`src/lib/matrix/state.ts`). On L1 expiry, resolution falls to **L2 = Upstash Redis
  over REST** (configured via `UPSTASH_REDIS_REST_URL` in `.env` — remote, even in dev)
  before falling back to Postgres (L3). Every `cacheTags.*` call inside a cached
  function pays this on re-execution.

### Why it alternated

- Reload within the seconds window → warm entries → ~130–300ms.
- Reload after expiry → full cold pass (depth-2 studio fetch + queue + sprints +
  matrix lookups, some over the Upstash REST network) → 1.4–1.8s.

**Correction (verified in code):** the original "amplifier" — *“Payload draft
autosave (~every 1–2s while editing) fires afterChange on every change”* — does not
apply to this project. No collection declares `versions`, so no collection has
drafts or autosave and Pages has no `_status` field at all. The real revalidation
churn drivers were:

1. **Every save revalidated, even when nothing a visitor can see changed.**
   `revalidatePagesCache` called `revalidate.page(...)` unconditionally; a repeated
   save or an admin-only edit purged the page and (through the shared root tenant
   tag, see “Adjacent findings”) every other cached slice of that tenant.
2. **`cacheLife('seconds')` = 1s server revalidate**, so effectively every request
   after the first was a revalidation pass.

### Cosmetic: doubled `[STUDIO IDENTITY]` log

`getStudioIdentityFromRequest` (`src/lib/tenant/subdomain.ts`) logged once per render pass:
one from the root layout's theme resolution (`getEffectiveTheme`,
`src/app/(frontend)/layout.tsx` lines 80/162), one from the page content
(`QueueContent` → `loadPageData` wrappers). React `cache()` dedupe does not span
the two passes. Pure string parsing — microseconds, not the latency problem.

### Implemented fixes

1. **Page revalidation gated on real change** (`src/lib/blocks/page-revalidation.ts`).
   Updates only revalidate when the storefront signature changed — `slug`,
   `title`, `pageType`, `isHidden`, `showInHeader`, `navTitle`, `headerOrder`,
   `blocks`, and the owning `tenant` (normalized to its id). Creates always
   revalidate; deletes always revalidate (via `revalidateDeletedPageCache`).
   Admin-only field churn (updatedAt, relationship hydration differences) no
   longer purges caches. A page **moved between studios** purges both the old
   and the new tenant's page tags (review follow-up, 2026-09-18).
2. **`cacheLife('seconds')` → `cacheLife('minutes')`** for artist-by-subdomain,
   artist-by-id, pricing, site settings, layout data, active sprints, upcoming
   round, queue and sprints. `minutes` is 60s server revalidate with
   stale-while-revalidate, so a stale entry is still served while the refresh
   runs in the background — no request blocks on the cold path. Freshness on
   writes is tag-driven (collection hooks) and does not wait for the timer.
3. **Dev latency**: `L1_TTL_MS` now defaults to 30s in production (unchanged) and
   5 minutes in development, and is overridable with `MATRIX_L1_TTL_MS`. A new
   `MATRIX_SKIP_L2=true` opt-out bypasses the Upstash L2 tier entirely for
   single-process dev servers.
4. **Subdomain log deduped** per host (capped set) and suppressed in production
   unless `DEBUG_SUBDOMAIN_LOGS=true`.
5. **Root-tag fanout narrowed** for the write paths that nothing else reads:
   `page`/`pages`/`queue`/`sprints`/`announcements` revalidations no longer purge
   `tenant_<id>`, and the layout cache now registers the `queue` tag (see
   “Adjacent findings” for the audit and the remaining flip).

### Verification for Part 1

- Reload `/queue` twice within 2s (both fast), wait 10s (slow under the old
  `seconds` profile), then reload twice more — with `minutes` the second and
  later reloads should stay in the fast band; one background refresh may appear
  on the first reload after the 60s window, not as blocked latency.
- Save a page without changing anything visible → no `[PAGES-HOOK]` tag churn
  and no cold pass on the next `/queue` render. Change a block → revalidation
  happens immediately (logs with `DEBUG_CACHE=true`).
- In dev, watch for `[MATRIX-REDIS]` fetches: after the first resolution they
  should disappear for 5 minutes (or entirely with `MATRIX_SKIP_L2=true`).

### Adjacent findings

- **Root tenant tag fanout — partially removed (2026-09-18).** Every `cacheTags.*` call
  registers the root tag `tenant_<id>`, and the coarse revalidations purge it, so a single
  page save used to mark *every* cached slice of that tenant stale.
  `revalidate.page`, `pages`, `queue`, `sprints` and `announcements` are now
  **root-exempt** (`REVALIDATE_ROOT_EXEMPT_METHODS` in `src/lib/tenant/tenant-tag.ts`): those
  slices have no readers outside their own tag family, so their writes no longer touch
  the artist, layout, pricing, products or reviews caches. `api.layout` now registers the
  `queue` tag so the layout's `showQueue` flag still tracks queue writes without the root.
  Everything else (`products`, `pricing`, `meta`, `orders`, `reviews`, …) keeps the root
  because it is load-bearing for real cross-slice dependencies — verified by audit:
  - `pricing`/`meta` root coverage: commissionSettings changes feed the artist cache that
    `fetchPricing` reads; media replacements feed resolved example images in pricing.
  - `products` root coverage: page documents embed product cards (`featuredProducts`
    blocks) at `hours` lifetime.
  - `revalidate.all()` (studio-level writes) still purges everything by design.

  Remaining flip (needs precise wiring before narrowing): `products` → pages that carry
  product blocks (either pages register the products tag, or the products hook keeps
  root), `pricing`/`meta` → their embedded dependency refresh paths. Do it behind the
  e2e preview/queue flows, not blind.
- **Soft revalidation semantics**: `safeRevalidateTag` calls
  `revalidateTag(tag, 'default')`, which in Next 16 marks tags *stale*
  (serve-stale + background refresh). Writes are therefore visible on the
  request after next rather than instantly; this is intentional SWR behavior,
  but if artists expect immediate propagation, `updateTag`/`expire` semantics
  would be the lever.

## Part 2 — Queue page "sometimes" not previewing in live preview

Two independent preview paths:

- **Theme preview** (cookie/`draft` param → `x-draft-theme-data` header →
  `DynamicTheme` + `LivePreviewThemeOverride`): no REST calls, reliable.
- **Page content preview**: `src/components/BlocksRenderer.tsx` + `useLivePreview`
  from `@payloadcms/live-preview-react`. The server-rendered base is always the
  **published** cached page (`getCachedPage` is `cacheLife('hours')` and not
  draft-aware). Live updates arrive via admin `postMessage` → merge POST to
  `${adminOrigin}/api/pages/{id}` with `credentials: 'include'`.

### Findings (corrected where the original reading did not match the code)

1. **Origin gate drifts after in-iframe navigation — CONFIRMED.**
   `isLivePreviewEvent.js` requires `event.origin === serverURL`, and
   `BlocksRenderer` derived `serverURL` from `document.referrer`. Fresh iframe →
   referrer is the admin origin → works. Navigating inside the preview (e.g. a
   link to `/queue`) → referrer becomes the studio origin → admin postMessage
   updates silently discarded. The custom `payload-live-preview` listener in
   `BlocksRenderer` is not origin-gated, which is why this presented as
   “sometimes” rather than “never”.
2. **CORS — CORRECTION.** The merge POST was *not* blocked: this project does
   not rely on Payload's `cors: '*'` for the REST API. `src/app/(payload)/api/[...slug]/route.ts`
   wraps every REST handler with a CORS layer that already reflected
   `Access-Control-Allow-Origin: <request origin>` + `Access-Control-Allow-Credentials: true`.
   The real defect was that the reflection was **open to any origin** — any
   website could read authenticated API responses from a logged-in artist's
   browser (Payload's own `headersWithCors` would have been the safer `*`
   behavior, but the wrapper deliberately overrode it). The wrapper now reflects
   only trusted origins: localhost / `*.localhost` / `*.test`, the configured
   root domain and its subdomains, `CORS_ALLOWED_ORIGINS` extras (tunnels), and
   hosts that resolve to a known studio custom domain (matrix lookup, fail-closed).
3. **Auth cookie scoping**: unchanged and correct for the cross-origin merge —
   `payload-token` is host-only on the admin origin, and cookies attach to the
   merge POST by *target* origin, so credentials work without bridging.
4. **Queue-specific block-index mismatch — CONFIRMED (different mechanism).**
   The merge does not “apply updates to the wrong rows” (it returns the admin's
   posted form values; the renderer resolves the whole list). The real offset was
   the **admin hover/scroll bridge**: it targets `data-block-index` with the row
   index of the *stored* page document, while `/queue` and `/shop` render the
   route's composed list with a synthetic block pinned in front
   (`auto-queue-view` / `auto-shop-catalog`), shifting every stored block by one.
   Hovering a row scrolled to and highlighted the wrong section.

### Implemented fixes

1. **Allowlisted CORS reflection** (`src/lib/tenant/cors-origins.ts` +
   `src/app/(payload)/api/[...slug]/route.ts`) — credentialed reflection kept for
   trusted origins only; unknown origins get no CORS grant. Trusted roots come
   from `getPlatformRootDomains()` (the same source host resolution uses), so the
   trust set cannot drift from subdomain resolution; tunnel origins are
   configured via `CORS_ALLOWED_ORIGINS`. The custom-domain probe is wrapped in a
   bounded in-process cache (`createCachedTenantDomainProbe`, 60s / 500 hosts,
   fail-closed) because the `Origin` header is attacker-controlled — without it,
   every request with a random origin would pay a Redis + Postgres lookup.
2. **Admin origin resolved env-first** — `NEXT_PUBLIC_ADMIN_ORIGIN` → referrer
   *only if it points at the platform* (not a studio host) → `getServerUrl()`.
   In-iframe navigations can no longer poison `serverURL`, so the package's
   `isLivePreviewEvent` gate and its `ready()` handshake target stay correct.
3. **No-op revalidation gate** (same as Part 1 fix 1) — the preview's
   server-rendered base stays warm instead of being purged by unrelated saves.
4. **Admin-row block indexes** — `resolveEffectiveBlocksWithIndexes` carries each
   block's editor row index; the synthetic route-owned block maps to a stored
   row only when a legacy block of the same type exists, otherwise it is marked
   `-1` and renders without `data-block-index`. Hover/scroll targeting now
   matches the editor on `/queue` and `/shop`.

### Verification for Part 2

- Open the live-preview iframe's devtools console, edit the queue page: the
  merge POST to `/api/pages/...` should return 200 (no CORS error) and blocks
  should update without a reload.
- Navigate the iframe (click “Queue” in the studio nav) and keep editing — the
  preview must keep updating (previously dead after navigation).
- Request `/api/pages?limit=1` from a foreign origin (e.g. a scratch HTML file)
  → the response must have **no** `Access-Control-Allow-Origin`, and the browser
  must block reading it. From a studio origin → reflected with credentials.
- Hover each block row in the editor on `/queue` → the preview should scroll to
  and flash the matching section (previously off by one).

## Regression surface to keep in mind

- Cache/block tables: `tests/int/block-data-needs.int.spec.ts` — green.
- `tests/int/effective-blocks.int.spec.ts` — extended with index cases, green.
- `tests/int/cors-origins.int.spec.ts`, `tests/int/page-revalidation.int.spec.ts` — new, green.
- Admin UX invariants: `tests/e2e/admin-ux.e2e.spec.ts` (not run here — needs the
  app + DB; the CORS/reflection change is not on its path).
- Tour anchors: `tests/e2e/setup-tour-anchors.e2e.spec.ts` (12/12 expected; not
  run here — sanity-gate it when the dev DB is available).
- `tests/int/tenant-matrix.int.spec.ts` — cannot run in a network-isolated
  sandbox: its `beforeEach` calls `invalidateTenantMeta` against the real Upstash
  REST endpoint from `.env` (the suite does not mock `@/lib/upstash`), so hooks
  time out at 10s (verified 2026-09-18). Environmental; nothing in these changes
  alters that call path (`isRedisConfigured()` only short-circuits when
  `MATRIX_SKIP_L2=true`).
- `tests/int/submit-commission.int.spec.ts` — 15/18 green under isolation. The
  three failures are not caused by these changes:
  - `rejects unauthenticated caller when no session is present` and
    `rejects unauthenticated requests with 401` need Postgres and die on
    `connect EPERM 127.0.0.1:5432` (sandbox blocks the connection).
  - `accepts valid dev / test / local media keys` fails at HEAD: the validator
    (`validateReferenceImageKeys` → `resolveMediaKeyShape`) no longer accepts
    non-tenant keys, while the test still expects `dev/`, `test/` and bare
    `media/` keys to pass. Pre-existing test/implementation drift — flagged for
    a follow-up, not touched here.
- Live-preview docs: `docs/theme-live-preview.md` (theme path unchanged). The
  content-preview transport notes above are the authoritative reference for the
  REST merge path.

## Env reference (new)

| Variable | Purpose |
| --- | --- |
| `NEXT_PUBLIC_ADMIN_ORIGIN` | Origin of the Payload admin for live preview. Defaults to `NEXT_PUBLIC_SERVER_URL`; set for tunnels. |
| `CORS_ALLOWED_ORIGINS` | Comma-separated extra origins allowed credentialed CORS on `/api`. |
| `MATRIX_L1_TTL_MS` | Override the L1 matrix TTL (default 30s prod / 5min dev). |
| `MATRIX_SKIP_L2` | `true` bypasses the Upstash L2 tier (dev only). |
| `DEBUG_SUBDOMAIN_LOGS` | `true` keeps the subdomain resolution log in production. |
