---
status: shipped
collections: [commissions, media, pages, queue, studios]
features: [media, pricing, queue, shop, tipping]
---


# Remaining Work — Page-Load Performance (post-fix)

Status: main program complete and verified; this document covers only what is left.
Date: 2026-09-27
Companion: `docs/page-load-performance-plan.md` (full diagnosis, phases, verification log §9)

---

## 0. Where things stand (verified on pre-prod, `dpl_8enCasXHU2WHaWx2bvc9TJtsbkEC`)

| metric | before | after |
|---|---|---|
| document total (HAR) | 3145 ms | 501–629 ms |
| TTFB / streamed tail | 227 / 2698 ms | 101–132 / 260–367 ms |
| cold first-load JS | ~1160 KB gzip | 410 KB transferred (22 chunks) |
| `onLoad` (cold visit) | > 3 s (corrupted pageTimings; document alone 3.1 s) | **1221 ms** |
| studio owner/stripe/commission data in public HTML | present (incl. owner email + session list) | **absent** (markers all 0) |

Done and verified: Vercel functions `iad1 → dub1`, barrel-graph cleanup (DynamicTheme, cleanup.ts, seed route, studios/admin-navigation chains, auth/legal/studio pages), Swiper lazy-load, lazy `RichTextEditor`, published-studio short-circuit in the visibility guard, `StorefrontStudioSettings` DTO + recursive tenant strip.

Everything below is **optional** unless a trigger condition fires. Each item lists where it lives, why it matters, how to verify, and its risk.

---

## 1. Finish the barrel audit (mechanical, low risk)

**What remains** (verified counts, 2026-09-27): barrels still re-exporting client components — `commissions` 18 exports, `auth` 9, `queue` 8, `pricing` 5, `shop` 4, `dashboard` 4, `onboarding` 3, `tipping` 2, `studio-settings` 2, `admin-dashboard` 2. Barrel-root imports outside `src/features`: **121**, mostly under `src/app/(app)/dashboard/**` (a separate route group, so mostly harmless) plus a few in `(frontend)`/shared code.

**Why**: the same class of leak that cost ~1 MB (admin UI) and ~500 KB (Swiper) earlier. What matters is only whether a barrel is imported by something reachable from a **public route's client graph**. The dashboard group is authenticated and lower priority.

**Do** — verified target list (2026-09-28; the earlier "121 barrel-root imports" were mostly `(app)/dashboard` and dead shims):

*Storefront/shared call sites that actually matter (7):*
1. `src/app/(frontend)/(studio)/shop/[slug]/BundleContentsWrapper.tsx` — `ProductCarousel, getProductsByIds` from the shop barrel → `@/features/shop/components/ProductCarousel` + `@/features/shop/actions/get-product.action`.
2. `src/app/(frontend)/(studio)/shop/[slug]/page.tsx` — `getProductBySlug` → `@/features/shop/actions/get-product.action`.
3. `src/app/(frontend)/(studio)/queue/[id]/discussion/page.tsx` — `getQueueItemById` → `@/features/queue/actions/get-queue-item.action`.
4. `src/app/(frontend)/(studio)/queue/[id]/page.tsx` — `RoadmapSection, NotifyButton, getQueueItemById` → direct component/action paths.
5. `src/components/blocks/HeroBlockComponent.tsx` — `ApplySlotButton` → `@/features/pricing/components/ApplySlotButton`.
6. `src/components/ui/RichTextEditor.tsx` — `mediaService` → `@/features/media/services/media.service` (low impact: this editor is already lazy-loaded).
7. `src/components/TenantSelector.tsx` — imports `useAuth` from `@payloadcms/ui` and `studioService` from the studios barrel, and **nothing imports this component** — delete it (or, if kept, import the service directly).

*Dead shims — zero importers, safe to delete (verified):* `src/components/{SecurePDFViewerClient,TipButton,PriceList,ShopItem,ShopList,CharacterDrawingTiers,ApplySlotButton,CustomsCarousel,TipAmountSelector,ArtistOnboardingModal}.tsx`, `src/components/commission-form/{FormStatus,Typewriter,FileUpload,Mascot}.tsx`, `src/features/tipping/services/tipService.ts`. Each is an `export *` re-export of a feature barrel; deleting them removes a future footgun at no cost. (`ComplexityBadge.tsx` has one importer — check before touching.)

*Type-only import — free:* `src/components/platform/PlatformLandingPage.tsx` uses `import type`, erased at compile time; leave it.

*Server-side barrel imports (defense in depth, low priority):* `src/lib/{stripeConnectAuth,dashboard-guard,discord}.ts`, `src/lib/ai/{processAIEstimateInBackground,estimateCommission}.ts` import the `studios`/`commissions` barrels. The `studios` barrel is now leak-free (the admin-navigation chain was fixed), but `commissions` still re-exports 18 client components — switch these to direct action imports when convenient.

**Verify**: `npx tsc --noEmit`; rebuild (`NEXT_DIST_DIR=.next-perf yarn build`); compare `.next-perf/diagnostics/route-bundle-stats.json` against `.next`; re-run the marker scan (see §5) for `/`, `/queue`, `/login`.
**Acceptance**: no increase anywhere; storefront first-load ≤ current (1293–1458 KB raw).
**Risk**: low, mechanical. Budget: ~1–2 h for the useful subset.

## 2. Lazy-load the remaining heavy leaves

- `SecurePDFViewerClient` — the `src/features/shop/components/SecurePDFViewerClient.tsx` shim has **zero importers** (delete it); the real component lives in `@/features/shop/components/SecurePDFViewerClient` and is pulled by the dashboard course viewer through the shop barrel → switch that import to the direct path, and optionally wrap it in `next/dynamic`. Dashboard-only, low priority.
- **Stripe: closed — no action needed.** Verified 2026-09-28: `grep -rn "@stripe/" src` returns nothing; `TipButton` imports only React/Button/useRouter/lucide/cn, and there is no `BuyNowButton.tsx` at the expected path. The earlier "Stripe 213 KB chunk" was marker noise (the string `stripe` matched `stripeAccountId` in the studio payload). No Stripe SDK is bundled client-side; checkout is server-driven/hosted.
- Chat widgets — `/chat` only; low priority.

**Verify**: manual check that the course viewer still renders the PDF (dashboard-only); Stripe needs no verification — closed by inspection.
**Acceptance**: no Stripe code in first-load chunks for any route that does not open a checkout; buttons still work.

## 3. Prefetch tuning (one flag + a few attributes)

Context: cold visit shows **26 RSC prefetch requests / 35 KB**, running +1018 → +2467 ms (after `load`). Not blocking, but noisy on the wire and in function-invocation counts.

- `experimental.prefetchInlining: true` in `next.config.mjs`. Verified in Next 16.2.12 source (`node_modules/next/dist/client/components/segment-cache/scheduler.js:934`): with the flag on, "all segment data for this route is bundled into a single `/_inlined` response". Expected: ~26 → ~5 requests per view. **Experimental — preview-test first.**
- `prefetch={false}` on low-value links (NavHeader already does this in two places; audit the remaining CTAs, Tip, Log in, and any footer link that is a `<Link>`).
- Keep default prefetch on the primary nav (Home/Prices/Queue/Shop) — it is what makes navigation instant.

**Verify**: cold-visit HAR before/after — prefetch count and timing; click-through navigation still instant.
**Acceptance**: fewer requests, no regressions in soft navigation.

## 4. Deferred — shared `use cache` handler (trigger-based)

Design and rationale: `docs/page-load-performance-plan.md` §3 Phase 1.5 (custom `CacheHandler` over `@upstash/redis`, tiered L1/L2, `use-cache:<VERCEL_ENV>:` prefix, tag manifest for global `revalidateTag`).

**Trigger to implement**: document tail back above ~600 ms after the region + DTO work, or a real cross-instance staleness complaint (`revalidateTag` is still process-local with the default 50 MB in-memory handler — verified in `node_modules/next/dist/server/lib/cache-handlers/default.external.js`).
**Related, cheaper lever**: if residual tail analysis points at the tenant matrix, create the Upstash database in **eu-west-1 (Dublin)** instead of eu-central (Frankfurt, ~20–25 ms/hop) — the cache data is disposable.

## 5. Deferred — client `UserContext` (sunfly-style) (trigger-based)

Design and do-nots: main plan §3 Phase 3. Motivation was (a) removing per-request `payload.auth()` DB lookups from the document path and (b) making the shell fully tenant-static.
**Trigger**: wanting the storefront shell cached without per-user variance, or auth lookups becoming measurable again. Currently the auth lookup is same-region (~2–5 ms), so value is low.

---

## 6. Regression guards (recommended, cheap)

1. **Bundle budget in CI** — after `yarn build`:
   ```bash
   node -e "const s=require('./.next/diagnostics/route-bundle-stats.json');const r=['/','/prices','/queue','/shop'];for(const x of s)if(r.includes(x.route)){const kb=(x.firstLoadUncompressedJsBytes/1024)|0;console.log(x.route,kb+' KB');if(kb>1600)process.exit(1)}"
   ```
   Current 1293–1458 KB; pre-fix 3891–3893 KB. Suggested threshold: 1600 KB raw.
2. **Latency watch** — scheduled curls against pre-prod (per studio route): alert when `total > 1.5 s` or `ttfb > 0.5 s`:
   ```bash
   curl -o /dev/null -s -w 'connect=%{time_connect} ttfb=%{time_starttransfer} total=%{time_total}\n' https://<host>/privacy-policy
   ```
3. **Measurement hygiene** — Vercel's preview toolbar (`vercel.live` feedback: ~23 KB JS + two ~34 KB HTML documents, ~91 KB) pollutes cold-visit captures. Disable it for the test environment or measure a production deployment.
4. **Local probe (if needed again)** — recreate as: start `next start` with `NEXT_DIST_DIR=.next-perf`, query the `studios` table for id/subdomain, mint a preview token (`<expiresAt>.<base64url(HMAC-SHA256(`${studioId}.${expiresAt}`, PAYLOAD_SECRET))>`, see `src/lib/theme/preview-access.ts`), then measure cold/warm per page with the token (`?preview=true&previewToken=…`). This was the `.perf-probe.mjs` script removed after the fix work; the main plan §9 documents what it produced.

---

## 7. Open questions / decisions

1. Commit the remaining untracked artifacts? (`docs/page-load-performance-plan.md`, this file, `.github/`, `docs/vercel-error-sync-github-worker.md`.)
2. Version the function region in `vercel.json` (`{"regions": ["dub1"]}`) — currently dashboard-only, per the user's earlier "ignore" call.
3. Is the dashboard group worth the barrel audit at all, given it is authenticated traffic? (Recommendation: only if the dashboard feels slow; the public storefront is the priority and is done.)
4. Bundle budget threshold — 1600 KB raw proposed; adjust to taste.

---

## 8. Verification commands (copy-paste)

```bash
# bundle report (after a build)
node -e "const s=require('./.next/diagnostics/route-bundle-stats.json');for(const r of s.filter(x=>['/','/prices','/queue','/shop','/login'].includes(x.route)))console.log(r.route,((r.firstLoadUncompressedJsBytes/1024)|0)+' KB')"

# pre-prod latency (repeat 2–3×)
curl -o /dev/null -s -w 'connect=%{time_connect} ttfb=%{time_starttransfer} total=%{time_total} size=%{size_download}\n' \
  https://bribabibi.2monke.xyz/privacy-policy

# cold-visit check: private window + Preserve Log + (toolbar disabled), then read the HAR
# document timings (wait/receive), first-load JS transfer, and the payload markers:
#   "sessions" / "owner" / "stripeAccountId" / "commissionSettings" must all be 0
```

---

## 9. Cold-start measurement (2026-11-09)

"Cold start" is four layers, and a single TTFB number cannot tell them apart:

1. **Artifact** — what the deployment ships and must extract/load (`.nft.json` trace size).
2. **Process** — Node spawn + module graph evaluation until the listen socket opens.
3. **Warmup** — `instrumentation.register()`: tenant-matrix DB queries + Redis pipeline.
4. **Request** — the first render, with an empty per-instance `use cache`.

### Why Next's own "Ready in" is not the number

`start-server.js` logs `Ready in X` **before** the config is loaded and before
`prepare()` runs. In `base-server.js`, `handleRequest` awaits `prepare()`, which loads
`instrumentation` and awaits `register()`. So Next's ready log excludes exactly the
warmup that dominates a low-traffic (scale-to-zero) deployment — and that cost lands on
the **first request** instead.

### Local A/B with `yarn measure:cold`

Build two arms from the same env (a git worktree of the baseline keeps the checkout
clean):

```bash
# baseline
git worktree add /tmp/artpack-base <baseline-commit>
(cd /tmp/artpack-base && NEXT_DIST_DIR=.next-base yarn build)
cp -R /tmp/artpack-base/.next-base ./.next-base

# candidate (current tree)
NEXT_DIST_DIR=.next-branch yarn build

# measure: alternating A B B A, one fresh process per sample
yarn measure:cold --a .next-base --b .next-branch --samples 5 --path /platform-home
```

`--json` emits the raw samples. The harness fails closed on a dist dir without
`BUILD_ID`, refuses to start when the port is occupied, and never reuses a process —
`handleRequest` awaits `prepare()`, so the first request pays `register()`.

Read the report left to right:

- `listen` — spawn → listen socket (layer 2).
- `first byte` — listen → first byte of the single cold request (layers 3+4).
- `next-ready-in` vs `first byte` — the gap Next's log hides.
- `register` / `warmup` / `import` — the `[BOOT]` split from `DEBUG_BOOT_TIMING=1`
  (`src/instrumentation.ts`), medians across samples.

Percentiles are nearest-rank and medians are reported instead of a mean: cold starts
are bimodal (host page cache, CPU steal, pooler handshake) and a mean hides that.

### Test deployment protocol

1. Deploy the baseline commit and the candidate as two preview deployments — same
   region, env and database.
2. Confirm the request hit a function, not the edge: the response must carry
   `x-vercel-cache: MISS`. A `HIT` measures the CDN, not a cold instance.
3. One request per idle period. Test has no traffic, so wait for the instance to be
   recycled or redeploy; redeploys are the cheapest way to get independent samples.
4. Set `DEBUG_BOOT_TIMING=1` on the preview environment only and read the `[BOOT]` line
   in the function logs plus `[TENANT-MATRIX] Warmup complete: … (Xms)`.
5. Compare medians of the first request per deployment and the warmup `Xms` — not a
   single curl.

### What a given change can show

- `outputFileTracingExcludes` — artifact layer only: check the `.nft.json` files, not
  TTFB. `typescript`, `drizzle-kit` and `esbuild` were 46 MB of the 77 MB
  instrumentation trace on the 2026-11-06 build.
- `PAYLOAD_DB_PUSH` opt-in — **development only**. Production never ran the schema
  push (`push` was already gated on `NODE_ENV !== 'production'`), so a deployment A/B
  will correctly show zero.
- `cacheLife('hours')` — helps only inside one warm instance. On a scale-to-zero
  deployment most requests still start with an empty `use cache`; the shared
  `cacheHandlers.default` (§4) is the lever that changes that.
