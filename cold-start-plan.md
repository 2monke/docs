# Cold-Start Improvement Plan

Status of the work, what is done, and what remains. Companion to
`docs/cold-start-measurement-runbook.md` (the copy-paste procedure) and
`docs/page-load-performance-remaining-plan.md` §9 (the four-layer rationale).

Last updated: 2026-11-09.

---

## Current state

| Branch | Base | Contents |
|---|---|---|
| `perf/cold-start-clean` | `origin/develop` @ `16213dc` | **Cold-start work only.** 6 commits (3 code, 3 docs). |
| `origin/develop` | `16213dc` | **Baseline for the A/B.** `perf/cold-start-clean` is cut from here. |
| `perf/cold-start-base` | `20af792` | Older baseline (escrow tip, 10 commits ahead of develop). Only if you deliberately want escrow in the comparison. |
| `perf/cold-start` | `20af792` | Original branch: cold-start **plus 10 escrow commits**. Kept for reference; not for the PR. |

Commits on `perf/cold-start-clean`:

```
6ecd064 docs(perf): point the runbook at the clean branch and its develop baseline
bc554ba docs(perf): note that the build needs network for next/font/google
bf76a6a docs(perf): write the cold-start plan to a file
6951a64 fix(cache): invalidate the landing slice from every write that can change a card
d43bf14 docs(perf): add the cold-start runbook and an artifact check to the harness
0af02ac perf(boot): cut cold-start work and add a cold-start measurement harness
```

Nothing is pushed. No PR is open.

---

## Phase 0 — Housekeeping — DONE

Stale build output (`.next-bundle` from a stalled build) removed. The stale
`.next/dev/types` generated from the escrow branch was moved to
`.next/dev/types.stale-escrow-branch`; it is git-ignored and `yarn dev`
regenerates it.

---

## Phase 1 — Isolate the cold-start work — DONE

`perf/cold-start-clean` was created from `origin/develop` and the three commits
cherry-picked onto it.

Four modify/delete conflicts resolved by dropping files that exist only on the
escrow branch (all preserved in `perf/cold-start`):

- `src/features/commissions/actions/admin-payout.action.ts`
- `src/features/commissions/actions/commission-milestone.action.ts`
- `src/features/commissions/actions/release-milestone.action.ts`
- `tests/int/admin-commission-refund.int.spec.ts`

Plus one dead file removed: `scripts/check-bundle-budget.mjs` (replaced by the
`.ts` gate; nothing referenced it).

Two content conflicts were the same root cause — the cold-start commit was
authored on the escrow branch, so its import-block rewrites dragged escrow
imports into files whose `develop` bodies never use them. Each had exactly one
occurrence (the import line) and was dropped:

- `queue-actions.ts` → `getCommissionById`
- `create-commission-checkout.action.ts` → `isCommissionEscrowEnabled`,
  `getCommissionSettingsByTenant`

**Gate passed:** `tsc --noEmit` clean; no import cycles; 88/88 across the six
core specs and 76/76 across six adjacent specs; eslint 0 errors; no conflict
markers; no escrow files in the diff.

---

## Phase 2 — Measure locally — DONE

Arms built 2026-11-09: `.next-base` (`origin/develop`) and `.next-branch`
(`perf/cold-start-clean`).

### Layer 1 — artifact — MEASURED

| Trace | Baseline | Candidate |
|---|---|---|
| `/platform-home` | 50.2 MB | **30.9 MB** |
| `/` (studio home) | 75.2 MB | **31.6 MB** |
| `/queue` | 75.3 MB | **31.7 MB** |
| `/login` | 50.3 MB | **31.2 MB** |
| `/studios` | 50.2 MB | **30.9 MB** |
| `server/instrumentation.js` | 77.3 MB | 76.7 MB (unchanged leak) |

**77/77 route traces clean.** Total route bytes 3854.5 MB → 2200.8 MB (−43%).

### The remaining artifact leak is not a glob bug

The instrumentation trace still carries 46.2 MB (typescript 18.8,
drizzle-kit 17.9, @esbuild 9.5, esbuild 0.1). Verified in Next 16.4.4:

- `build/collect-build-traces.js` applies `outputFileTracingExcludes` while
  walking `buildTraceContext.chunksTrace.entryNameFilesMap` — route entries only
  (`app/…`, `pages/…`).
- `server/instrumentation.js.nft.json` is not a route entry and the module never
  appears in that file; Turbopack writes the trace directly.

The cause is reachability: `src/instrumentation.ts` imports
`@/lib/tenant/tenant-matrix`, which pulls the Payload config graph
(`@payloadcms/db-postgres` → drizzle-kit/esbuild; Payload → typescript). No glob
can remove it; the import has to go. That is Phase 3.

### Layers 2–4 — timings — FINAL

Three runs, all 2026-11-09, on a normal machine (the coding sandbox cannot bind
a port). 5 samples per arm, one fresh process per sample, alternating order.

**Runs 1 and 2 predate the branding merge.** They are kept as the trail that
justified Phase 3, not as current figures. Run 3 is the one to quote.

**Run 1 — before removing the boot warmup** (this is what justified Phase 3):

| metric | baseline (`origin/develop`) | branch | Δ |
|---|---|---|---|
| listen | 131 / 160 p90 | 130 / 131 | −1 / −29 |
| first byte | 892 / 1086 | 854 / 869 | −38 / −216 |
| register | n/a | 544 | — |
| warmup | n/a (not reported) | **531** | — |

Ranges overlapped, so that −4.3% was noise. What it established was the
decomposition: branch process → first byte = 984 ms, of which 130 spawn +
36 module eval + **544 `register()` (531 of it warmup)** + 274 render. The
warmup was **54% of the cold path**. The baseline pays the same warmup —
`origin/develop:src/instrumentation.ts` is identical; it just has no `[BOOT]`
line to report it with.

**Run 2 — after removing it:**

| metric | baseline | branch | Δ |
|---|---|---|---|
| listen | 131 / 131 p90 | 131 / 132 | +1 / +1 |
| **first byte** | **1015 / 1089** | **291 / 310** | **−724 / −779 (−71.3%)** |
| end | 1061 / 1237 | 335 / 356 | −727 / −880 |
| register | n/a | **0 ms** | — |
| warmup | n/a | none (lazy) | — |

**The distributions do not overlap:** baseline min 984 ms vs branch max 310 ms.
That is the first result here that is not a noise question.

And `next-ready-in` is ~51 ms in **both** arms — unchanged. Next's own "Ready in"
never showed either the 531 ms warmup or the 724 ms it cost, which is exactly why
the hook had to be measured directly.

**Run 3 — post-branding-merge (current figures):**

Run after merging `origin/develop` (PR #119), which rewrote the landing page and
changed the platform palette. Both arms rebuilt from the merged commits.

`/platform-home`:

| metric | baseline | branch | Δ |
|---|---|---|---|
| listen | 132 / 156 p90 | 130 / 131 | −2 / −24 |
| **first byte** | **927 / 1120** | **325 / 332** | **−602 / −787 (−64.9%)** |
| end | 980 / 1272 | 379 / 388 | −602 / −883 (−61.4%) |
| warm (2nd req) | 16 / 19 | **20 / 22** | **+4 / +3** |
| register | n/a | 0 ms | — |

Studio home `/`:

| metric | baseline | branch | Δ |
|---|---|---|---|
| **TTFB** | **919 / 949** | **304 / 317** | **−615 / −632 (−66.9%)** |
| full document | 1128 / 1163 | 1009 / 1021 | −120 / −142 (−10.6%) |
| warm (2nd req) | 16 / 17 | 15 / 16 | −1 |

Still disjoint (platform-home: baseline min 860 ms vs branch max 332 ms).

**Two differences from Run 2, both explained by branding:**

- The *relative* win on `/platform-home` shrank (71% → 65%) because develop got
  faster too — 1015 → 927 ms — after the landing rewrite. The branch also moved,
  291 → 325 ms.
- Warm `/platform-home` went **16 → 20 ms (+22.7%)**. The ranges barely overlap
  (14–19 vs 18–22), so it is probably real: `LiveStudiosRow` and `MarqueeRow`
  serialise more per response. Four milliseconds on a warm request against
  ~600 ms on a cold one.
- Studio full-document improved from −6.3% to −10.6%; more of the render is
  streamed rather than blocked.

### Studio routes — settled by the warm request

`--path / --host-header vronyakat.localhost:3211`, 5 samples. The harness sends a
**second request to the same process**, which is what separates one-time init
from per-request cost. Figures below are **Run 3 (post-merge)**; Run 2's were
909→294 TTFB and 1107→1038 full document.

| metric | baseline | branch | Δ |
|---|---|---|---|
| **TTFB** | **919 ms** | **304 ms** | **−615 ms (−66.9%)** |
| body after TTFB | 209 ms | 705 ms | +496 ms |
| **full document** | **1128 ms** | **1009 ms** | **−120 ms (−10.6%)** |
| **warm document (2nd req)** | **16 ms** | **15 ms** | −1 ms (−9%) |

Across routes:

| route | TTFB | full document |
|---|---|---|
| `/platform-home` | −65% | −61% |
| studio home `/` | **−67%** | **−11%** |

**Why the difference, now measurable rather than inferred.** Next runs
`register()` inside `handleRequest` (`await this.prepare()`), so the boot warmup
was *always* part of the first request — it just blocked before the first byte.
Per cold instance (Run 2 figures; Run 3 shifts both by a few tens of ms):

- baseline: 531 ms warmup, then 210 ms render → TTFB at 909 ms, total 1107 ms
- branch: 0 ms warmup, then 686 ms render → TTFB at 294 ms, total 1038 ms

Same work, different position. The branch is 55 ms cheaper because the
redundant product preload is gone, and it shows HTML 615 ms sooner. It does
**not** make the document finish much sooner on a route that must render per
request.

**Warm is 15–20 ms in both arms.** Steady-state serving was never the problem,
and the change does not affect it. The 686 ms is paid once per instance, on
whichever request arrives first.

`/platform-home` escapes this because its shell is prerendered into the PPR
static output: its body after TTFB is ~45 ms either way, so the whole boot
saving lands in the total. **Caveat:** the branch's build writes a 0-byte shell
file for every `(frontend)` route (see “Prerendered shell — OPEN RISK” above), so
this explanation is only confirmed if a deployment serves a real HTML shell. The
measured body-after-TTFB is consistent with it either way.

**What this means for the next lever.** The render's own data pass is only
~155 ms of that 686 ms; Payload init is the bulk. Cutting queries
(`getCachedLayoutData` → `getCachedSiteSettings` in root `generateMetadata`)
would therefore recover at most ~100 ms, not the 531 ms — so it is **not** the
next lever, despite being the obvious-looking one. The real lever is not paying
Payload init on a cold instance at all: the shared `use cache` handler
(`docs/caching-strategy-and-scaling.md`), or prerendering more of the studio
shell.

### Artifact — FINAL (post-merge)

| Trace | Baseline | Branch |
|---|---|---|
| `server/instrumentation.js` | **77.3 MB** / 1695 files | **0.04 MB** / 2 files |
| traced-out packages in it | 46.2 MB | **0.0 MB** |
| all route traces combined | 3854.6 MB | 2201.0 MB (−42.9%) |
| route traces | 65 of 77 leaking | **77 of 77 clean** |

`yarn measure:cold --artifacts --strict --dist .next-branch` exits **0** — strict
mode included, which was the point of keeping that flag.

### Prerendered shell — OPEN RISK (post-merge build)

Every `(frontend)` route on the branch writes a **0-byte** HTML file:

| route | base | branch |
|---|---|---|
| `index`, `platform-home`, `studios`, `onboarding`, `login`, `prices`, `queue`, `shop`, `[slug]`, `tos`, `privacy-policy`, `theme-preview`, `tip`, … | ~2.2 KB | **0 B** |
| `dashboard` | 2581 B | 2301 B |
| `_not-found`, `_global-error` | unchanged | unchanged |

Evidence it may be benign: both builds set `x-nextjs-prerender: 1` and list the
routes in `prerender-manifest.json`; runtime returns HTTP 200 with comparable
body-after-TTFB on both arms; and the branch's `_full.segment.rsc` is **larger**
(152.6 KB vs 9.7 KB) — more content is captured, not less.

Evidence it may not be: a 0-byte file is not a shell, so if the edge serves it,
nothing arrives until the postponed part streams; and only the `(frontend)` group
is affected, which points at this branch's root-layout change rather than a Next
quirk. `dashboard` (the `(app)` group) is unaffected.

Not resolvable from build output, and the coding sandbox cannot bind a port.
**Verify on a preview deployment before merging** — `curl -s …/platform-home |
head -c 400` should begin with `<!DOCTYPE html>`. See the PR body step 5.

`perf/cold-start-base` (`20af792`) is the escrow tip, 10 commits ahead of
`develop`; `.next-base` was built from `origin/develop` so the delta isolates
these commits only.

---

## Phase 3 — Act on the numbers — DONE (boot warmup removed, measured)

### Done and measured: the boot warmup is removed

`src/instrumentation.ts` now does no work: no import, no queries, no Redis
pipeline. `warmupTenantMatrix()` still exists and is still tested — it is the
building block for option 2 below if the preload is ever wanted back.

**Measured effect** (2026-11-09, `/platform-home`, 5 samples per arm):

| | before | after |
|---|---|---|
| first byte (median) | 1015 ms | **291 ms** (−71.3%) |
| first byte (p90) | 1089 ms | **310 ms** (−779 ms) |
| `register()` | 544 ms (531 warmup) | **0 ms** |
| instrumentation trace | 77.3 MB / 1695 files | **0.0 MB / 2 files** |

Rationale, in order of certainty:

- **The measurement.** The hook was 531 ms of a 984 ms process-to-first-byte
  path (54%), locally, where Postgres is on localhost. Against a remote pooler
  it costs more.
- **The artifact.** 46.2 MB of the 76.7 MB instrumentation trace — typescript,
  drizzle-kit, @esbuild, esbuild. `outputFileTracingExcludes` cannot reach it,
  so the import had to go.
- **The redundancy.** The tenant matrix already resolves lazily: L1 memory →
  L2 Upstash Redis (24 h TTL) → database (`src/lib/matrix/engine.ts`). The
  preload made every cold instance pay for tenants it might never be asked
  about — on a scale-to-zero deployment that is a pessimisation.

**Honest limit:** this does not delete the Payload init cost. The first render
that reads the database (`getFeaturedStudios` on `/platform-home`) still
initialises Payload; it just does it inside the request instead of before it.
The measured 724 ms therefore includes the warmup *plus* the fact that two
sequential `payload.find` scans are gone; it is not purely init.

Guard: `tests/int/instrumentation-boot.int.spec.ts` fails if the hook reaches
the Payload graph again — verified by mutation (re-adding the import fails both
the source check and the behavioural check).

Remaining options if the preload is wanted back:

1. ~~Delete the boot warmup~~ — **done, measured.**
2. **Move the warmup to a cron/route** (`/api/warm-matrix`, invoked after
   deploy). Same trace win if the route is not reachable from the hook; keeps
   preloading; needs its own auth. This is the only supported way back.
3. **Gate it behind `MATRIX_WARMUP` inside `instrumentation.ts`.** Do not: Next
   traces a literal `await import()` regardless of the branch around it, so the
   46.2 MB returns while the gate gives the impression it was handled.

Regression check (both must stay green):

```
yarn measure:cold --artifacts --strict --dist .next-branch
```

```
yarn measure:cold --a .next-base --b .next-branch --samples 5 --path /platform-home
```

### Decided by the timing run

The measurements above resolved every row of the original decision table:

| Observation | Verdict |
|---|---|
| `warmup` dominates | **Confirmed** — 531 ms of a 984 ms path. Removed. |
| `listen` dominates | No — 131 ms and unchanged by this branch. |
| Module graph is the cost | No — 13 ms to import the matrix, 0 ms now. |
| Render data pass is the cost | No — ~155 ms of the 686 ms render. |
| Payload init is the cost | **Yes, and it is unavoidable per cold instance.** Moving it earlier does not delete it. |

**The remaining lever is not in this branch:** a cold instance that must read
the database pays Payload init somewhere. Removing it means not needing the DB
on the first byte — a shared `use cache` handler
(`docs/caching-strategy-and-scaling.md`) so a cold instance reads cached tenant
data from Upstash, or prerendering more of the studio shell the way
`/platform-home` does. Both are separate projects.

**Explicitly rejected:** trimming the render's query set (`getCachedLayoutData`
→ `getCachedSiteSettings` in root `generateMetadata`). The ceiling is ~100 ms
against a 531 ms init, it touches every storefront route, and the effort is
better spent on the shared cache.

---

## Phase 4 — Test deployment — TODO

Local numbers are a ratio, not a forecast. Only Vercel shows the artifact win as
latency, because function boot there includes downloading and extracting the
deployment bundle.

1. Deploy `origin/develop` (or `perf/cold-start-base`) and `perf/cold-start-clean`
   as two preview deployments — same region, env, database.
2. `DEBUG_BOOT_TIMING=1` on the preview environment only.
3. Confirm `x-vercel-cache: MISS` before trusting any number.
4. One request per idle period; redeploy for independent samples.
5. Compare medians of the first request, plus `[TENANT-MATRIX] Warmup complete
   … (Xms)` and the `[BOOT]` line.

Full protocol: `docs/cold-start-measurement-runbook.md` §8.

---

## Phase 5 — PR — TODO

Push `perf/cold-start-clean` and open against `develop`. Description: the three
commit bodies plus the Phase 2 numbers.

Leave `perf/cold-start` local until the escrow branch is separately pushed or
abandoned — it carries the same three commits on top of 10 escrow commits.

---

## Open items, not in this branch

- **Gate the tenant warmup** — done, and the mechanism is gone (see Phase 3).
- **Shared `use cache` handler** — makes "test is always cold" stop being true;
  separate project, documented in `docs/caching-strategy-and-scaling.md`. This is
  the only remaining lever that removes the ~686 ms cold render rather than
  relocating it.
- **`revalidateArtistCache` calls `revalidate.meta(doc.id)`** on the artist
  document. Checked 2026-11-09: **keep it.** The tenant-namespaced tags it
  produces (`tenant_art_…:*`) are junk — no reader registers them — but
  `resolveTenant()` does not return null for an `art_` id, it echoes
  `{ id: 'art_…', subdomain: 'art_…' }`, so the call still revalidates the bare
  `featured-studios` tag. That makes it the fallback that clears the landing
  slice when an artist has **no** resolvable studio (the `studioIds` loop is
  empty). Removing it would quietly reduce invalidation coverage. Left alone;
  a cleanup would have to replace it with an explicit
  `safeRevalidateTag(FEATURED_STUDIOS_TAG)` rather than a deletion.
- **`TestModeBanner` Suspense fallback** renders the amber banner before the
  gate resolves `x-is-live-preview`, so the admin live-preview frame flashes a
  banner it deliberately hides. Cosmetic, test-mode only.
- **`getCachedLayoutData` in root `generateMetadata`** still awaits the full
  layout pass (studio depth-2 + announcements + sprints + queue) before first
  byte on studio routes. Switching to `getCachedSiteSettings` would remove
  three query families — but measured, that render's data pass is only ~155 ms of
  the 686 ms, so the ceiling is ~100 ms. **Rejected as poor value**, not pending.

---

## Rejected: excluding sharp / MJML from page traces (2026-11-09)

The remaining artifact weight per route is dominated by two packages:
`@img/sharp-libvips` (15.2 MB) and the MJML/juice tree (1.8–3.5 MB). Excluding
them from page-route traces looked like the obvious next win. **It is not safe,
and the reason is that both are genuinely reachable from a page render.**

Evidence, from the 2026-11-09 build (`.next-branch`):

| route | sharp | mjml + juice |
|---|---|---|
| `(frontend)/(platform)/platform-home/page` | 15.7 MB | 1.8 MB |
| `(frontend)/(studio)/page` | 15.7 MB | 1.8 MB |

Why, in each case:

- **sharp** — `src/payload.config.ts:7` is a module-scope `import sharp from
  'sharp'`, and every page render loads that config (any Payload read goes
  through `@payload-config`). Excluding sharp from a page trace does not make the
  route skip it; it makes module evaluation throw `MODULE_NOT_FOUND`. Payload's
  `SharpDependency` type is **synchronous** (`(input, options) => sharp.Sharp`),
  so it cannot be swapped for an async import either.
- **MJML** — lazily imported at runtime (`src/lib/email/render.ts`), but server
  actions send email and Next serves a server action from the page route that
  hosts it. `src/features/commissions/actions/submit-commission.action.ts:32`
  imports `@/lib/email` and calls `sendCommissionReceivedEmail` on the studio
  pages, so those routes need MJML at runtime.

What a safe attempt would require (not done, and not to be attempted blind):

1. Replace the module-scope sharp import with a lazy wrapper
   (`const lazySharp: SharpDependency = (i, o) => require('sharp')(i, o)`) so
   config evaluation no longer needs the binary.
2. Add route-scoped `outputFileTracingExcludes` for `sharp`/`@img` on page
   routes **and** matching explicit `outputFileTracingIncludes` on the routes that
   do resize (`/admin`, upload APIs), so the two cannot drift.
3. Prove it with a build plus a real upload through the admin and one through
   the presigned-URL API. Neither step is possible in the coding sandbox (no
   network for `next/font/google`, no `listen`).

Until that is verified, the artifact number in this branch is close to the
practical ceiling for cheap work: the remaining per-route weight is sharp plus
the Payload/Next runtime, all of which pages actually need.

---

## What this plan does not claim

Three measurement runs on 2026-11-09: Runs 1–2 justified the boot change, Run 3
is post-branding-merge and is the one to quote. What remains unproven:

- **Vercel behaviour.** Every timing is local with Postgres on localhost. The
  artifact reduction is measured; the instance-boot benefit it buys (download +
  extract) is not. Phase 4 is the proof.
- **Studio full-document time.** It improved by 10.6%, not 67%. Payload init
  moved from the boot hook into the first render. Anything describing this branch
  as a storefront-wide page-load win is wrong.
- **The prerendered shell — the live risk.** Every `(frontend)` route writes a
  0-byte HTML file on the branch, against ~2.2 KB on the baseline. Build output
  is self-contradictory (`x-nextjs-prerender: 1` and a manifest entry, but no
  shell), so it needs a deployment check. If the shell is genuinely gone, the
  edge serves nothing and every request invokes a function.
- **Warm `/platform-home` regressed 4 ms** (16 → 20). Small, but real and
  attributed to branding's extra client components.
- **Sharp/MJML exclusion.** Rejected on evidence, not attempted (see above).
