---
status: shipped
collections: [announcements, artists, commissions, media, pages, products, queue, sprints, studios]
features: [admin-navigation, auth, pricing, queue, shop, studio-settings, studios]
---

# Page-Load Performance Plan — artpack (Next 16 + Payload on Vercel)

Status: plan / ready for implementation
Date: 2026-09-26
Author: session diagnosis (measured on the running system)
Scope: make storefront page loads fast on the Vercel pre-production deployment, without regressing correctness or the live-preview / multi-tenant features.

---

## 0. Goal & success criteria

| Metric | Today (pre-prod) | Target |
|---|---|---|
| Document `total` (curl, anonymity, warm edge) | 2.1–2.9 s | < 0.9 s |
| Document body time (`total − ttfb`) | 1.8–2.7 s | < 0.4 s |
| Document `ttfb` | 0.23–0.80 s | unchanged (≤ 0.3 s) |
| Storefront first-load JS (per route, `.next/diagnostics/route-bundle-stats.json`) | ~3.9 MB raw / 1.16 MB gzip (28 files) | < 1.0 MB raw |
| Local prod-build probe (`/`, `/prices`, `/shop`, `/queue`) | warm 4–11 ms, cold 87–238 ms | keep (guard against regressions) |

The user-visible symptom being fixed: page loads take seconds even though the HTML is small and the edge reports a cache HIT.

---

## 1. Measured evidence (reproducible)

### 1.1 Local production build + local Postgres (proves the app code is not slow)

Run: `node .perf-probe.mjs` (starts `next start` on :3211 against the already-built `.next`, measures cold + warm, saves HTML to `.perf-out/`).

- Studio pages via preview token: TTFB 1.2–6.5 ms, first content chunk 2.6–8.8 ms, **full document warm 6–11 ms, cold 87–238 ms**.
- Platform pages (`/platform-home`, `/studios`): warm 6–10 ms, cold 16–39 ms.
- Response headers: `x-nextjs-postponed: 1`, `Cache-Control: private, no-cache, no-store, max-age=0, must-revalidate`.

### 1.2 Vercel pre-prod (the actual problem)

HAR (`bribabibi.2monke.xyz/privacy-policy`, WebKit) parsed with `.har-doc.mjs` / `.har-analyze.mjs`:

- Document: `time 3145 ms` = TCP+TLS 354 ms + **wait (TTFB) 227 ms** + **receive 2698 ms**; body 72609 B uncompressed / 14934 B br.
- `x-vercel-cache: HIT`, `Age: 547` on the document — the prerendered shell is served from the edge cache, yet the tail still takes ~2.7 s.
- `x-vercel-id: arn1::iad1::…` → functions run in **iad1** (Washington DC).
- Reload: **all 32 JS chunks, 5 CSS, font = 0 ms / 0 KB** (browser cache). So on reloads the bundle is not the cost.
- RSC prefetch burst: 11–23 requests (`?_rsc=`, `next-router-segment-prefetch`), each ~180–380 ms wait, `x-vercel-cache: HIT`, only ~1 KB each. They start after the document finishes.

Two `curl`s from the reporter's machine (anonymous, no cookies, no JS):

```
connect=0.537 ttfb=0.796 total=2.900 size=71252
connect=0.045 ttfb=0.293 total=2.096 size=71252
```

→ ~1.8–2.1 s is spent after the first byte: the server producing the postponed (dynamic) part of the PPR response.

### 1.3 Cache infrastructure facts (verified in this repo / node_modules)

- `next.config.mjs` has `cacheComponents: true` and **no `cacheHandlers`**.
- Next 16 default `use cache` handler: `node_modules/next/dist/server/lib/cache-handlers/default.external.js` → `createDefaultCacheHandler(50 * 1024 * 1024)` = **50 MB in-memory LRU per process**. Cache dies with the instance; `revalidateTag` is process-local.
- Handler contract for a custom implementation: `node_modules/next/dist/server/lib/cache-handlers/types.d.ts` (`get(cacheKey, softTags)`, `set(cacheKey, pendingEntry)`, `refreshTags()`, `getExpiration(tags)`, `updateTags(tags, durations?)`); config shape `cacheHandlers: { default: '<module path>' }` (`config-schema.js:210`).
- Repo's own plan `docs/caching-strategy-and-scaling.md` already decided on Upstash + tiered handler; status "implementation pending".
- Env already present: `UPSTASH_REDIS_REST_URL` / `UPSTASH_REDIS_REST_TOKEN`; `@upstash/redis` is a dependency.
- Storefront slices use `cacheLife('minutes')` (`src/features/studios/cache/studios.cache.ts`, `src/features/queue/cache/queue.cache.ts`); page/pages slices `hours`.
- The tenant matrix (`src/lib/matrix/state.ts`) has L1 in-memory (30 s in prod) and **L2 = Upstash Redis over REST** — one more network hop per cold L1, whose region must be checked together with the DB region.

### 1.4 Payload bloat (present in every storefront document)

`siteSettings` and the page's `initialData.tenant` both embed the **depth-2 studio document**:

- `fetchStudio` → `getStudio(identity, { depth: 2 })` (`src/features/studios/cache/studios.cache.ts`).
- `fetchLayoutData` / `fetchSiteSettings` then return `{ ...studio, logo, logoMobile, … }` — the whole object.
- That object contains `owner` (artist user with **11 session IDs + timestamps — not token values** — plus consents and the owner's account email), `artists[].user` (same user again), `theme`, `commissionSettings` (tiers, checklist), `tenants`.

Consequence: two copies of a heavy, user-shaped object in the RSC payload of every page (visible in the captured HTML). This is payload hygiene *and* a privacy fix: the owner's `email` and their session list must never reach public HTML.

### 1.5 Per-request auth cost (logged-in visitors only)

`getCachedUser` (`src/features/auth/cache/auth.cache.ts`) → `getAuthenticatedUser` (`src/features/auth/actions/get-authenticated-user.action.ts`) → `payload.auth({ headers })`: JWT verify **plus user/session DB lookup**, once per request whenever a `payload-token` cookie is present. It is only React-`cache()`d (per request), not cross-request.

`getStudioVisibility` (`src/lib/tenant/studio-visibility-guard.ts`) fetches the user **unconditionally**, although `canViewStudio` (`src/features/studios/lib/studio-visibility.ts`) returns `true` for `profileStatus === 'published'` before consulting the user — i.e. an avoidable DB lookup on every visit to a published studio while logged in.

### 1.6 Client bundle composition (first visits)

From the served HTML + `.next/diagnostics/route-bundle-stats.json` (route `/queue`):

- 28 JS files, **4.0 MB raw / 1.16 MB gzip**; chunks include Payload **admin UI ~1 MB** (`RenderFields`, `Upload` strings; 1032 KB `2tf45z401xap1.js`), **Swiper 520 KB**, **emoji-picker + Lexical ~300 KB**. (Correction 2026-09-28: a 213 KB chunk was earlier attributed to Stripe — that was marker noise, the string `stripe` matched `stripeAccountId`; `grep -rn "@stripe/" src` finds no client-side Stripe SDK.)
- `/login` first-load = 3043 KB; `/_not-found` = 518 KB (root baseline). The storefront (frontend) route group shares one large chunk set.
- Verified import chains that put heavy code in that set:
  - `src/components/DynamicTheme.tsx` (server component in the root layout) imports `getEffectiveTheme` from the **barrel** `@/features/studio-settings`; that barrel `export *`s `components/PayoutsView` and `components/IntegrationsTab`, both `'use client'` and both importing `@payloadcms/ui`. Both modules appear in `/login`'s `page_client-reference-manifest.js` (verified).
  - `src/components/BlocksRenderer.tsx` (client) imports the `@/features/pricing|commissions|shop|queue` barrels, so content pages pull the whole catalogue even when they render one block.
  - `src/features/auth/index.ts` re-exports every auth form + `@/components/ui/Turnstile`.
  - `src/components/ui/Carousel.tsx` uses Swiper; imported by `HomeBentoGrid`, `ProductCarousel`, `CustomsMobileCarousel`, `ExtrasMobileCarousel`, `CommissionReviewsShowcase`, `LibraryContent`, `TourCardBody`.
  - `src/features/queue/components/CommentsSection.tsx` (line ~890) statically imports `RichTextEditor` (Lexical + `emoji-picker-react`).

### 1.7 Prefetch amplification

Next 16 segment prefetching: every `<Link>` fetches its route tree per segment (`_tree`, `_head`, `_index`, `(frontend)/(studio)/…`, `__PAGE__`). One privacy-policy view produced 11–23 RSC requests. `experimental.prefetchInlining` exists in Next 16.2.12 and is **off** (`config-shared.js:194`); when on, `scheduler.js:934` bundles all segments of a route into a single `/_inlined` response.

---

## 2. Root-cause ranking

1. **H1 (primary):** `use cache` entries live only in each function instance's memory. Pre-prod has ~no traffic → instances are cold/unrecycled → the dynamic tail re-runs the storefront data pass (studio depth-2, layout data, header pages, social settings, page depth-2) against a **remote** Postgres, partly sequential. Locally the same pass costs 87–238 ms; on Vercel it costs ~2 s because of DB RTTs (functions `iad1` ↔ DB `eu-west-1` per `TEST_DATABASE_URL`) times N queries.
2. **H2 (amplifier):** the pass is heavier than needed: depth-2 reads + embedding the full studio doc twice in every payload.
3. **H3 (first-visit only):** 1.16 MB gzip JS with admin UI/Swiper/emoji in the graph; barrel imports keep it there.
4. **H4 (amplifier):** 11–23 segment prefetches per view; each is a potential function invocation while caches are cold.
5. **H5 (logged-in only):** per-request `payload.auth` DB lookup; the guard's is avoidable for published studios.

Not causes: client JS on reload (0 ms in HAR); images; prefetch traffic for TTFB.

---

## 3. Phases

### Phase 0 — Baseline harness (0.5 day)

Keep these scripts available while iterating (they were created during the diagnosis; the originals were removed once the fixes landed — this section documents what they did):

- `.perf-probe.mjs` — starts `next start` on :3211 against `.next`, queries studios from Postgres, mints preview tokens (`?preview=true&previewToken=…` per `src/lib/theme/preview-access.ts`), measures TTFB / first chunk / total cold+warm per page, writes `.perf-out/results.json` + HTML.
- `.har-analyze.mjs`, `.har-doc.mjs` — parse WebKit HAR exports (document timings, cache headers, prefetch segment table).
- `.perf-analyze.mjs` — maps route client-reference-manifests to modules/chunks.
- Commands: `node .perf-probe.mjs`; `node .har-doc.mjs <paste.md>`; pre-prod curl:
  `curl -o /dev/null -s -w 'connect=%{time_connect} ttfb=%{time_starttransfer} total=%{time_total} size=%{size_download}\n' https://<studio-host>/privacy-policy`
- Bandwidth control test (rules the reporter's link in/out):
  `curl -o /dev/null -s -w 'ttfb=%{time_starttransfer} total=%{time_total} speed=%{speed_download}B/s\n' https://<host>/_next/static/media/c41ca59f1c34ba31-s.p.2y2uoi4t910qy.woff2`

Gate G0: baseline table filled (per page: local warm/cold, pre-prod ttfb/total, gzip JS, function region, DB region).

### Phase 1 — Kill the server-side document tail (P0)

**1.1 Co-locate function region with the DB (do first; ~10 min; no app code).**
- Confirm the pre-prod `DATABASE_URL` host/region (do not print secrets). The test DB is `aws-1-eu-west-1.pooler.supabase.com:6543` → AWS eu-west-1 (Ireland) → the matching Vercel function region is **`dub1`**.
- There is no `vercel.json` in the repo today; the project may instead pin regions in Vercel project settings. Pick one source of truth; if code, add `vercel.json` with `{ "regions": ["dub1"] }` (or the region matching the DB).
- Also check the **Upstash** database region (tenant-matrix L2) and move it near the functions if far.
- Re-measure with the curls. Acceptance: document body < 0.6 s, or a measured delta proving the tail is not purely DB latency.
- Caveat: keep functions near the DB *and* the audience; a US-east DB with an EU audience is a trade-off to state explicitly.

**1.2 Verify whether Vercel's runtime cache already serves `use cache` across instances/deploys (1 hour).**
Gate for the handler work: if active, a custom handler is likely unnecessary; if not, keep it on the roadmap but off the critical path once 1.1 + 1.3 land. (An external review asserts Vercel handles this natively; unverified from this repo — check before building anything.)

**1.3 Slim the cold pass and the payload (app code).**
- `src/features/studios/cache/studios.cache.ts`
  - `fetchStudio`: read the studio with a `select` and/or build an explicit shell DTO (name, subdomain, id, logo, logoMobile, profileStatus, theme, currency, socialLinks, nav flags). Never spread `...studio` into props.
  - `fetchLayoutData` / `fetchSiteSettings`: construct narrow objects field-by-field instead of `{ ...studio, … }`; strip `owner`, `artists`, `sessions`, `consents`, `commissionSettings` internals.
  - Consider splitting into slices (`shell`, `nav`, `pricing`) so a cold pass only pays for what the route renders.
- `src/features/studios/actions/page.action.ts`: `getPageBySlug(..., { depth: 2 })` → `depth: 0/1` + explicit `select` (blocks + minimal tenant) so pages stop embedding the full tenant tree.
- `src/lib/tenant/studio-visibility-guard.ts`: short-circuit before `getCachedUser()` when `studio.profileStatus === 'published'` (verdict is user-independent — see `canViewStudio`).
- `src/features/auth/cache/auth.cache.ts`: cross-request TTL cache for the token→user resolution (e.g. `use cache` + tag `auth-<tokenHash>`, 30–60 s), with `revalidateTag` in the logout action. Explicit trade-off: up to TTL of revocation latency; keep TTL ≤ 60 s.
- Update every consumer that depended on the removed fields (grep for `siteSettings.` usages, `StudioHeader`, `NavHeader`, `SiteFooter`, `DynamicTheme`, block components) — verify by `tsc --noEmit` + local probe.

**1.4 Optional warm-on-deploy** if 1.2 finds the platform cache is not active.

**1.5 Shared cache handler for `use cache` (deferred; only if 1.1+1.3+1.2 leave a gap).**
Options, in order:
- (a) Platform cache (1.2) — nothing to build.
- (b) Custom handler implementing `CacheHandler` from `node_modules/next/dist/server/lib/cache-handlers/types.d.ts`, backed by `@upstash/redis` (env already present). Requirements:
  - `set(cacheKey, pendingEntry)`: await the entry, buffer `value: ReadableStream<Uint8Array>`, store `{ value, tags, stale, timestamp, expire, revalidate }`; must satisfy "get waits for pending set".
  - `get(cacheKey, softTags)`: return `undefined` when any tag's revalidate timestamp is newer than the entry (or when soft tags are stale); otherwise the entry.
  - `updateTags(tags, durations?)`: write tag → timestamp (drives `revalidateTag` globally across instances).
  - `getExpiration(tags)`: return the max tag timestamp, or `Infinity` to delegate staleness to `get(softTags)`; `refreshTags()` may no-op.
  - Tiered L1 (in-process Map, small) + L2 Upstash REST; key prefix `use-cache:<VERCEL_ENV ?? 'dev'>:`; consider value-size cap (Upstash REST value limits) — buffer + gzip, skip entries beyond the cap.
  - Deployment-safety decision to make explicitly: prefix by env only (entries survive deploys, matches Next's data-cache behavior; requires warm-on-deploy, 1.4) vs. include deployment id (cold every deploy). Recommendation: env-only + 1.4.
- (c) `@neshca/cache-handler` v1.9.0 (repo's earlier plan) — **verify its API matches the Next 16 `CacheHandler` interface before adopting**; it predates `cacheComponents`.

Config: `next.config.mjs` → `cacheHandlers: { default: '<path>' }`. No new runtime dependency needed for (b) beyond `@upstash/redis`.

Acceptance: two requests landing on different instances share entries; `safeRevalidateTag` from one instance invalidates others; document body time < 0.5 s on pre-prod; key cardinality sane.

Expected outcome of Phase 1: document body 1.8–2.7 s → < 0.4 s; total < 0.9 s. If 1.1 shows the tail is not DB-bound, escalate 1.2/1.5 instead of 1.3.

### Phase 2 — Client bundle diet (P1, first-visit only; can run in parallel with Phase 1)

**2.1 Break the barrel leaks (highest value, smallest diffs).**
- `src/components/DynamicTheme.tsx`: `import { getEffectiveTheme } from '@/features/studio-settings'` → import the action module directly (`@/features/studio-settings/actions/get-effective-theme.action`). This removes `PayoutsView` + `IntegrationsTab` + `@payloadcms/ui` from every frontend route's client graph.
- `src/features/auth/index.ts`: audit what it re-exports; keep the barrel client-safe but stop pulling every form + `@/components/ui/Turnstile` into pages that need one component (import components directly at call sites). Respect the existing note about NOT re-exporting `auth.cache`.
- Audit client components importing `@/features/*` barrels (list in §1.6); switch to direct module imports (AGENTS.md rule for hooks/API/webhooks already says this).

**2.2 Dynamic-import heavy leaves:** `RichTextEditor` in `CommentsSection` (`next/dynamic`, only on interaction/open), `SecurePDFViewerClient`, Stripe (`BuyNowButton`, `TipButton` load on intent), chat widgets.

**2.3 Replace Swiper** in `src/components/ui/Carousel.tsx` (520 KB raw) with a light carousel (Embla ~15 KB, or CSS scroll-snap). Verify the 7 importers listed in §1.6.

**2.4 Prefetch tuning:** `experimental.prefetchInlining: true` (preview-test first — experimental), and `prefetch={false}` on low-value links (footer legal/social, Tip, Log in). Keep default prefetch on the main nav.

**2.5 Acceptance:** rebuild → `.next/diagnostics/route-bundle-stats.json` storefront first-load < 1.0 MB raw; `/login` client graph no longer contains `@payloadcms/ui`-importing modules, Swiper, or emoji/Lexical; re-run `.perf-probe.mjs` and the pre-prod curls.

### Phase 3 — Optional: client `UserContext` (sunfly-style) (P2, deferred)

Only after Phase 1+2. Motivation captured from `/Users/eugenepucko/Projects/sunfly` (`src/types/IUserContext.ts`, `src/app/api/user-context/route.ts`, `src/client/features/layout/providers/Auth/index.tsx`):

- Add `UserContextProvider` (client) in the root layout + `GET /api/user-context` returning a **minimal** shape `{ id, name, collection, avatar, tenants? }` — not the full document (sunfly returns the full user; do not copy that).
- Make `NavHeader`'s Dash/Logout/avatar and `ArtistAdminTopBar` consume the context instead of server-passed user props, so the storefront shell becomes user-agnostic.
- Cache client-side keyed by subdomain (multi-tenant: `payload-tenant-subdomain`); clear on login/logout/studio switch. Respect anonymous live-preview (`x-anonymous-preview`).
- Keep server-side auth for guards (`getStudioVisibility`, hidden-page preview in `getCachedPage`), dashboard, APIs.
- Costs: a neutral header until context resolves; one extra request per load (client-cached).
- Do not port sunfly's bare 5 s timestamp cache or its full-document response.

### Phase 4 — Regression guards (P1)

- Keep the local probe + a bundle-size check in CI: parse `.next/diagnostics/route-bundle-stats.json` and fail on a storefront budget (e.g. > 1.2 MB raw first-load).
- Watch post-deploy: Vercel function duration for the storefront routes, cache hit ratio, and `x-vercel-cache` on document requests.
- Optional: a scheduled curl matrix against pre-prod recording ttfb/total per studio page, alerting on regressions.

---

## 4. Ordering & dependencies

```
Phase 0 ─┬─ Phase 1.1 (regions) ─ Phase 1.2 (cache check) ─ Phase 1.3 (slim) ─ Phase 1.4 (warm, opt) ─┐
         ├─ Phase 1.5 (handler, only if needed) ─────────────────────────────────────────────────────┤
         └─ Phase 2 (independent) ────────────────────────────────────── Phase 3 (deferred) ── Phase 4
```
- 1.1 first: cheapest, largest expected win, and the decisive experiment.
- 1.2 gates 1.5 (don't build a handler if the platform already provides one).
- 1.3 is pure app-code and is worth landing regardless.
- Phase 2 is independent; do it first if the owner prioritizes first-visit UX over repeat-visit latency.

## 5. Risks & rollbacks

- **Custom cache handler:** stream buffering, tag semantics, cross-deploy staleness. Mitigate: env-scoped prefix, size cap, tag invalidation test, `MATRIX_SKIP_L2` untouched. Rollback = remove `cacheHandlers`.
- **Auth TTL cache:** revocation latency ≤ TTL; add tag invalidation on logout; keep TTL ≤ 60 s. Rollback = delete the cache wrapper.
- **Payload slimming:** components may rely on removed fields. Mitigate: `tsc --noEmit` + local probe + grep consumers before merging.
- **`prefetchInlining`:** experimental; preview-test, keep main-nav prefetch semantics.
- **Region move:** migration window/DB connection strings; do it in one change and re-measure. The edge stays global (nearest to the visitor); only the compute region moves — check audience geography before pinning, and re-check the Upstash region too.
- **Swiper replacement:** check the 7 importers for autoplay/pagination features before swapping.

## 6. Open questions for the implementer

1. Is Vercel's runtime cache for Next 16 `use cache` active for this project? (gates 1.2/1.5)
2. Which DB does pre-prod use, and in which region? (1.1)
3. In which region is the Upstash database used by the tenant matrix? (1.1)
4. Do pre-prod functions scale to zero between requests? (explains "2 s on every curl")
5. Auth revocation tolerance for the TTL cache in 1.3?
6. Bundle budget + which routes are in scope (studio storefront only, or dashboard too)?

## 7. File index (starting points)

| File | Why |
|---|---|
| `next.config.mjs` | `cacheHandlers` (1.2), `experimental.prefetchInlining` (2.4) |
| `src/features/studios/cache/studios.cache.ts` | depth-2 studio reads, `{...studio}` spreads, storefront cache lives here |
| `src/features/studios/actions/get-studio.action.ts` | `select`/depth options for the studio read |
| `src/features/studios/actions/page.action.ts` | page depth-2 read |
| `src/lib/tenant/studio-visibility-guard.ts` | unconditional user fetch (1.4) |
| `src/features/studios/lib/studio-visibility.ts` | `canViewStudio` — published short-circuit |
| `src/features/auth/cache/auth.cache.ts` | token→user resolution (1.4) |
| `src/features/auth/actions/get-authenticated-user.action.ts` | `payload.auth` call site |
| `src/lib/utils/revalidate.ts` | `safeRevalidateTag(tag, 'default')` semantics |
| `src/components/DynamicTheme.tsx` | barrel leak #1 (2.1) |
| `src/features/studio-settings/index.ts` | the barrel itself (2.1) |
| `src/features/auth/index.ts` | barrel leak #2 (2.1) |
| `src/components/BlocksRenderer.tsx` | barrel imports of 4 feature barrels (2.1) |
| `src/components/ui/Carousel.tsx` | Swiper (2.3) |
| `src/features/queue/components/CommentsSection.tsx` | static `RichTextEditor`/Lexical/emoji import (2.2) |
| `src/features/shop/components/SecurePDFViewerClient.tsx` | heavy leaf (2.2) |
| `.next/diagnostics/route-bundle-stats.json` | bundle measurement artifact (after `yarn build`) |
| diagnosis harness (`.perf-probe.mjs`, `.har-analyze.mjs`, `.har-doc.mjs`, `.perf-out/`) | used during the 2026-09-26 diagnosis; removed once the fixes landed |
| `docs/caching-strategy-and-scaling.md` | prior decisions for the shared cache handler |

## 8. Verification commands (copy-paste)

```bash
# local re-measure (harness scripts were removed with the fix work): rebuild into a
# scratch dist dir and curl a preview-token URL against a local `next start`,
# or use the pre-prod curls below
NEXT_DIST_DIR=.next-perf yarn build

# pre-prod latency (repeat 2–3×; watch ttfb vs total)
curl -o /dev/null -s -w 'connect=%{time_connect} ttfb=%{time_starttransfer} total=%{time_total} size=%{size_download}\n' \
  https://bribabibi.2monke.xyz/privacy-policy

# bandwidth control (should be < 150 ms if the link is fine)
curl -o /dev/null -s -w 'ttfb=%{time_starttransfer} total=%{time_total} speed=%{speed_download}B/s\n' \
  https://bribabibi.2monke.xyz/_next/static/media/c41ca59f1c34ba31-s.p.2y2uoi4t910qy.woff2

# bundle report after a build
node -e "const s=require('./.next/diagnostics/route-bundle-stats.json'); for (const r of s.filter(x=>['/','/prices','/queue','/shop'].includes(x.route))) console.log(r.route, (r.firstLoadUncompressedJsBytes/1024).toFixed(0)+' KB')"

# HAR inspection — the .har-*.mjs parser was removed with the harness; read
# WebKit's exported HAR directly (document entry timings + x-vercel-cache)
```

---

## 9. Verification log

**2026-09-26 — Phase 1.1 applied (Vercel functions `iad1` → `dub1`, redeployed).**

Before (functions `iad1`, DB eu-west-1): `total 2.10–2.90 s`, body (`total − ttfb`) `1.8–2.1 s`, `ttfb 0.22–0.80 s`.

After (three runs, same URL, anonymous):

| run | connect | ttfb | total | body |
|---|---|---|---|---|
| 1 | 0.052 | 0.291 | 0.854 | 0.563 |
| 2 | 0.055 | 0.222 | 0.708 | 0.486 |
| 3 | 0.063 | 0.218 | 0.414 | 0.196 |

Reading: the transatlantic DB RTT multiplier is gone. The third run (~196 ms body) is essentially the local cold-pass cost from §1.1 (87–238 ms) — the floor for a cold instance. Residual body ≈ cold-pass DB/Payload work + matrix L2 REST hops (Dublin→Frankfurt ~20–25 ms each, only when L1 is cold) + streaming. `ttfb` unchanged, as expected.

Implication for the remaining phases: the custom shared cache handler (1.5) is now the lowest-priority item; Phase 1.3 (slimming) still shrinks the residual tail and fixes the payload/privacy issue; Phase 2 (client bundle) is now the largest user-visible win for first visits.

**2026-09-26 — Phase 2.1 applied (barrel-graph cleanup). Verified with a fresh build.**

What was changed:

- `src/components/DynamicTheme.tsx` — direct import of `getEffectiveTheme` (was the `@/features/studio-settings` barrel).
- `src/lib/infra/cleanup.ts` — direct action imports (was 5 feature barrels). This module sits on the `NavHeader → auth-actions → performDeepCleanup` chain, so its barrels leaked into **every** page.
- `src/app/(frontend)/api/seed/route.ts` — direct action imports (was 8 feature barrels). A route handler inside the `(frontend)` group was pulling all eight catalogues into the group's shared client chunks.
- `src/features/studios/actions/get-studio-artist-profile.action.ts` — direct imports of the two admin-navigation helpers (was the `@/features/admin-navigation` barrel, which re-exported `IconNavLinks` → `@payloadcms/ui`). This was the chain that kept the ~1 MB Payload admin UI in the storefront graph after the first pass.
- `src/features/admin-navigation/index.ts` — stopped re-exporting client components (actions/types only; comment documents the direct-import convention).
- `src/components/NavHeader.tsx`, `src/components/LogoutButton.tsx`, `src/components/ConsentCheckboxes.tsx`, `src/components/SocialLogins.tsx` — direct imports instead of the auth/tipping barrels.
- `src/components/BlocksRenderer.tsx` — direct component imports (was 4 feature barrels).
- `src/features/queue/components/CommentsSection.tsx` — `RichTextEditor` (Lexical + emoji-picker) is now `next/dynamic` with `ssr: false`.
- Auth/legal/studio pages (`login`, `register`, `forgot-password`, `reset-password` ×2 groups, `social-confirm`, `tip`, `verify`, `onboarding`, `studios`, `prices`, platform `layout`/`platform-home`) — direct component/action imports instead of barrels.
- `src/lib/tenant/studio-visibility-guard.ts` — published studios short-circuit before `getCachedUser()`.

Measured (same machine, `.next-perf` build vs the pre-change `.next` build, `route-bundle-stats.json`):

| route | before (raw) | after (raw) | after (gzip) |
|---|---|---|---|
| `/` | 3891 KB | **1392 KB** | 424 KB |
| `/prices` | 3893 KB | 1395 KB | 424 KB |
| `/queue` | 3891 KB | **1458 KB** | 443 KB |
| `/shop` | 3891 KB | 1392 KB | 424 KB |
| `/login` | 3042 KB | **1204 KB** | 361 KB |
| `/_not-found` | 518 KB | 518 KB | 148 KB |

Marker scan of first-load chunks: `RenderFields`/`useAuth` (Payload admin UI) now absent from `/`, `/queue`, `/login`; the emoji-picker/Lexical editors are lazy (only a 28 KB residue remains); **Swiper is still 506 KB in every frontend route** — the new largest single item.

Other checks: `tsc --noEmit` clean; `next build` clean; local probe (`.perf-probe.mjs`, `.next-perf`) all pages HTTP 200 — warm totals 4–9 ms, cold 75–200 ms.

Remaining Phase 2 work: lazy-load or replace Swiper (`src/components/ui/Carousel.tsx`, 7 importers); audit the other component-exporting barrels (`commissions` 19, `queue` 9, `auth` 9, `pricing` 6, `shop` 4) for server-side importers; optionally lazy-load `SecurePDFViewerClient` and Stripe buttons.

**2026-09-26 — Phase 2.3 applied (Swiper lazy-loaded). Verified with a fresh build + probe.**

`src/components/ui/Carousel.tsx` is now a thin `'use client'` wrapper: Swiper's implementation moved to `src/components/ui/CarouselSwiper.tsx` and is loaded via `React.lazy` + `Suspense`. The fallback renders the slides server-side as a static scroll row, so carousel content stays in the initial HTML (and crawlable) until the Swiper chunk arrives. Public API unchanged (`Carousel`, default export, `CarouselProps`, `CarouselAutoplayOptions`).

Measured (`.next-perf` vs the pre-change `.next` baseline):

| route | before (raw) | after (raw) | after (gzip) |
|---|---|---|---|
| `/` | 3891 KB | **1293 KB** | 395 KB |
| `/prices` | 3893 KB | 1295 KB | 396 KB |
| `/queue` | 3891 KB | 1358 KB | 415 KB |
| `/shop` | 3891 KB | 1293 KB | 395 KB |
| `/login` | 3042 KB | **1104 KB** | 333 KB |

Marker scan: `RenderFields` / `useAuth` / `swiper` / `emoji-picker` are now absent from the first-load chunks of `/`, `/prices`, `/queue`, `/shop`, `/login`.

Cumulative so far: storefront first-load **3891 KB → 1293 KB raw (−67%)** and **~1160 KB → 395 KB gzip (−66%)**; `/login` 3042 → 1104 KB.

Runtime: local probe all 44 requests HTTP 200; warm totals 4.5–12.4 ms; the server-rendered carousel fallback was found in the home/prices HTML (content present before hydration).

Still open: further barrel audit (`commissions` 19 component exports, `queue` 9, `auth` 9, `pricing` 6, `shop` 4 — check for server-side importers in the `(frontend)` group), lazy `SecurePDFViewerClient` / Stripe buttons, and Phase 1.3 (studio DTO slimming — payload size + the owner-email/session-list exposure in public HTML).

**2026-09-27 — Phase 1.3 implemented (verification pending).**

- `src/features/studios/cache/studios.cache.ts`: added `StorefrontStudioSettings` + `toStorefrontStudioSettings()`, used by `fetchSiteSettings` and `fetchLayoutData` — no more `{ ...studio }` spread (drops `owner` (artist user with session rows + consents + email), `artists[].user`, stripe fields, `commissionSettings` internals from every page's payload). Added `toStorefrontPage()`, applied in `fetchPageBySlug` so the depth-2 page read keeps only `{ id, name, subdomain }` on its `tenant` field (the storefront only reads the tenant id — `extractTenantId` in `BlocksRenderer`).
- New shared helper `src/lib/tenant/storefront-payload.ts` (`stripTenantSubtrees`) replaces every populated `tenant` subtree with `{ id, name, subdomain }`, applied inside the cached functions (page read + `siteSettings.theme`) so the walk happens once per cache entry. Other storefront slices are already safe: products use a `select` without `tenant`, queue items likewise, sprints/announcements are read at `depth: 0`.
- Verified 2026-09-27: `tsc --noEmit` clean; fresh build clean; runtime check against the built server (studio home/prices/shop via preview token, all HTTP 200) shows `"sessions": 0`, `"owner": 0`, `stripeAccountId: 0`, `commissionSettings: 0` in the delivered HTML — the studio document no longer reaches public payloads. The three remaining `tenant` references are the slim `{ id, name, subdomain }` shape (the page doc and the theme). HTML sizes: home ~86 KB, prices ~65 KB, shop ~59 KB.
- Consumer audit performed before the change: `siteSettings` readers use `id`, `name`, `logo`, `logoMobile`, `theme`, `meta.*`, `artistBackground`, `mascotImage`, `shopDescription` — all present in the DTO; the payment gates read `stripeChargesEnabled` from `getStudioById`, not from `siteSettings`.

**2026-09-27 — Deployed verification (pre-prod, `dpl_8enCasXHU2WHaWx2bvc9TJtsbkEC`).**

`/privacy-policy` reload captured as a WebKit HAR (62 entries):

| metric | before (2026-09-26) | after |
|---|---|---|
| document total | 3145 ms | **501 ms** |
| wait (TTFB) | 227 ms | 132 ms |
| receive (streamed tail) | 2698 ms | **367 ms** |
| uncompressed HTML | 72.6 KB | 55.6 KB |
| over the wire | 14.9 KB | 13.3 KB |
| payload markers | `sessions` 11, `owner`, `stripeAccountId`, `commissionSettings` present | **all 0** (tenant refs are the slim `{id,name,subdomain}` shape) |

- JS on this reload: 22 chunks, **0 bytes transferred** (browser cache). Cold-visit first-load measured locally on the same code: ~395 KB gzip (was ~1160 KB). CSS/font: 0 bytes (memory cache).
- RSC prefetch burst: 23 requests, ~6 KB total, starting at +613 ms and ending at +1406 ms — background work after the document completes.
- The only non-cached third party observed: `vercel.live` feedback widget (~23 KB) — preview-deployment tooling, not app code; absent in production.

End-to-end so far: document **3145 → 501 ms** (−84%), streamed tail **2698 → 367 ms** (−86%), storefront first-load JS **~1160 → ~395 KB gzip** (−66%), and the studio document no longer appears in public payloads.

**2026-09-27 — Cold-visit verification (same deployment, anonymous, empty cookies).**

WebKit HAR captured from a fresh window (no `payload-token`, no cache): `pageTimings.onContentLoad` 892 ms, `onLoad` **1221 ms**.

- Document: 629 ms (blocked 268 / wait 101 / receive 260), 13.2 KB over the wire, `x-vercel-cache: HIT`, payload markers all 0.
- First load over the network: **JS 22 chunks / 410 KB**, CSS 32 KB (4 files; largest 29 KB — Tailwind), font 33 KB, document 13 KB. Matches the ~395 KB gzip measured locally on the same code (was ~1160 KB before the barrel/Swiper work).
- RSC prefetch burst: 26 requests / 35 KB, running +1018 → +2467 ms — after `load`, background only.
- Non-app noise in the capture: Vercel's preview toolbar (`vercel.live` feedback: ~23 KB JS + two ~34 KB HTML documents, ~91 KB total). Preview-only; disable the toolbar (or measure a production deployment) to keep cold-visit numbers clean.
- Largest app chunks on this route: 125 KB (`44-axwn_zzsip.js`) and 71 KB (`1qpsmg0fb_obl.js`) — framework/shared chunks, not feature code.
