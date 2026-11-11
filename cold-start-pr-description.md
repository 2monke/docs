# Cold start: HTML arrives ~600 ms sooner, and 43% of the deployment traces are gone

> **Read this before the tables.** Two things improve and one does not.
>
> - **First byte** (when the user sees HTML) drops by 65–67% on both surfaces.
> - **Deployment artifacts** drop 43%; the instrumentation trace loses all
>   46.2 MB it used to carry.
> - **Full-document time on studio routes barely improves** (−11%). Payload init
>   used to run in the boot hook and now runs in the first render — same work,
>   earlier paint. Warm requests are ~15–20 ms in both arms either way.
>
> `/platform-home` is the exception: it prerenders its shell, so there the boot
> saving lands in the total too (−61%).
>
> All figures are measured **after** merging the branding work from
> `origin/develop` (PR #119).

## Summary

Serverless cold start on a scale-to-zero deployment is dominated by work that
happens before the first request is served. This branch removes the largest
piece of it — 531 ms of the first request, and 46.2 MB of what the deployment
must download.

Because Next runs `register()` inside `handleRequest` (`await this.prepare()`),
the warmup was always part of the first request; removing it moves Payload init
out of the pre-first-byte path into the streamed render. That is why the first
byte improves so much and the total does not.

## Measurements

`/platform-home` (prerendered shell), 5 samples per arm, one fresh `next start`
per sample, baseline `origin/develop`:

| metric | baseline | branch | Δ |
|---|---|---|---|
| **first byte** | **927 ms** (1120 p90) | **325 ms** (332 p90) | **−602 ms / −64.9%** |
| full document | 980 ms (1272 p90) | 379 ms (388 p90) | −602 ms / −883 p90 |
| `register()` (boot hook) | 544 ms | **0 ms** | −544 ms |
| listen (spawn → socket) | 132 ms | 130 ms | unchanged |
| **warm document (2nd request)** | **16 ms** | **20 ms** | **+4 ms** |

The distributions do not overlap: baseline minimum 860 ms, branch maximum
332 ms.

Studio home (`/`, `--host-header vronyakat.localhost:3211`), same arms:

| metric | baseline | branch | Δ |
|---|---|---|---|
| **TTFB** | **919 ms** | **304 ms** | **−615 ms / −66.9%** |
| body after TTFB | 209 ms | 705 ms | +496 ms |
| full document | 1128 ms | 1009 ms | **−120 ms / −10.6%** |
| **warm document (2nd request)** | **16 ms** | **15 ms** | −1 ms |

Per cold instance: baseline spent 531 ms warming then rendered the rest;
branch renders it all. Same work, repositioned, and HTML arrives ~600 ms sooner.

Across routes:

| route | TTFB | full document |
|---|---|---|
| `/platform-home` | −65% | −61% |
| studio home `/` | −67% | −11% |

**Warm is ~15–20 ms on both arms.** Steady-state serving was never the problem.
The cold cost is paid once per instance, on whichever request arrives first.

**One number moved the wrong way:** warm `/platform-home` went 16 → 20 ms
(+4 ms, +22.7%). The ranges barely overlap (baseline 14–19, branch 18–22), so it
is probably real — branding's `LiveStudiosRow` and `MarqueeRow` serialise more
per response. It is 4 ms on a warm request, against ~600 ms on a cold one, but
it is reported rather than omitted.

## What is not the next lever

Cutting the render's queries (root `generateMetadata` uses `getCachedLayoutData`
where `getCachedSiteSettings` would do) recovers at most ~100 ms of the 686 ms —
Payload init is the bulk, and it is ~155 ms of data pass plus ~530 ms of init.
Measured and rejected as poor value for a change touching every storefront route.

The real remaining lever needs a cold instance to answer without touching the
database: a shared `use cache` handler (`docs/caching-strategy-and-scaling.md`),
or prerendering more of the studio shell the way `/platform-home` does. Both are
separate projects.

Deployment artifacts, same build:

| | baseline | branch |
|---|---|---|
| `server/instrumentation.js.nft.json` | 77.3 MB / 1695 files | **0.04 MB / 2 files** |
| all route traces combined | 3854.6 MB | 2201.0 MB (−42.9%) |
| route traces carrying typescript/drizzle-kit/esbuild | 65 of 77 | **0 of 77** |

The instrumentation trace loses 46.2 MB of typescript, drizzle-kit and esbuild —
not because a glob pruned it, but because nothing imports the Payload config
graph from `src/instrumentation.ts` any more.

## The main change

`src/instrumentation.ts` no longer warms the tenant matrix. Next awaits
`register()` in `prepare()`, which `handleRequest` calls before serving
anything, so the hook's cost was added to the first request of every cold
instance.

It ran a Payload init, two `payload.find` calls (up to 1,000 studios and 2,000
products) and a Redis pipeline of ~4,000 SETs: measured at **531 ms of a 984 ms
process-to-first-byte path (54%)**, against a localhost database. Against a
remote pooler it costs more.

It also cost **46.2 MB** of the instrumentation trace, because
`resolvePayloadInstance` reaches `@payload-config` through a literal dynamic
import, pulling Payload, `@payloadcms/db-postgres` → drizzle-kit/esbuild, and
typescript into the deployment. `outputFileTracingExcludes` cannot remove it —
Next applies those globs while walking route entries only, and Turbopack writes
the instrumentation trace itself (verified in Next 16.4.4 source).

The tenant matrix already resolves lazily: L1 memory → L2 Upstash Redis (24 h
TTL) → database. The preload made every cold instance pay for tenants it might
never be asked about.

`warmupTenantMatrix()` is unchanged and still tested — it is the building block
for a post-deploy cron/route if a warm matrix is ever wanted back. Gating it
behind an env var inside the hook would **not** help: Next traces a literal
`await import()` regardless of the branch around it.

## Also in here

**Merged `origin/develop` (branding, PR #119)**

Four conflicts, two of them semantic:

- **Both sides added the same platform short-circuit** in `getEffectiveTheme`.
  Branding's returns `PLATFORM_THEME_VARS` (light brand palette); ours returned
  the studio generator's default (dark). Kept branding's palette, kept our
  `cache()` and `isStudio` flag.
- **`StaticPlatformTheme` was injecting the dark generator default.** With the
  branding palette now light, that would have painted a dark shell on a light
  brand — the flash this branch was fixing, inverted. It now injects
  `PLATFORM_THEME_VARS`.
- Root layout: branding's fonts/icons/viewport plus our removal of theme
  injection from the shared root.
- `PlatformLandingPage`: kept the CTA-slot extraction, moved branding's restyled
  final CTA (board yellow) into `PlatformFinalPrimaryCta`.
- `platform-landing-studios` spec: took branding's rewrite, re-applied our flag
  pin and CTA tests.

**Client JavaScript — measured, −14% on average**

The barrel-to-direct-import work is a first-visit win that the earlier draft of
this description missed. From the two builds' `route-bundle-stats.json`
(first-load, raw):

| route | before | after | Δ |
|---|---|---|---|
| `/platform-home`, `/studios`, `/invite/[token]` | 1000 KB | 556 KB | **−444 KB** |
| `/onboarding` | 1040 KB | 641 KB | −398 KB |
| `/tip`, `/queue/[id]/discussion` | 1083 KB | 788 KB | −291 KB |
| `/queue/[id]`, `/chat/[roomId]` | ~1416 KB | ~1240 KB | −177 KB |
| `/login` | 1075 KB | 836 KB | −238 KB |
| storefront (`/`, `/queue`, `/shop`, `/prices`) | ~1347 KB | ~1165 KB | −182 KB |

Average across all routes: **1207 KB → 1034 KB**. Every route improved; none
regressed.

**Cold request path**

- `/platform-home` renders the landing shell into the static PPR shell; only the
  session-aware CTAs suspend, with signed-out fallbacks.
- `getEffectiveTheme` is React-cached and short-circuits platform hosts before
  the layout cache.
- **Theme injection moved into the route groups.** The shared root layout used
  the platform default (dark) as the Suspense fallback for the request-resolved
  theme, so a light-themed studio painted light → dark → studio. `(platform)`
  now renders the static default (synchronously, so it stays in the shell),
  `(studio)` resolves its own theme behind the same Suspense as its content, and
  the root renders neither.
- `getCachedFeaturedStudios`: depth 1 + explicit select, narrower DTOs, `hours`
  lifetime.
- `TestModeBannerGate` returns before reading headers when test mode is off.
- Direct action imports replace `@/features/*` barrels across actions,
  onboarding, collections and sitemap.

**Cache correctness**

- The landing slice is now invalidated by every write that can change a card:
  artist (names), theme (brand logo), media (logo file) joined studio and
  sprint. Previously three of five were missing, so with its `hours` lifetime a
  rename or logo swap could stay stale for the window.
- The tag is a single exported constant (`FEATURED_STUDIOS_TAG`).

**Tooling**

- `yarn measure:cold` — cold-start A/B harness: one fresh process per sample,
  listen-socket probe, one request, alternating order, nearest-rank
  medians/p90. `--artifacts` inspects traces without a server; `--strict` fails
  on a retained instrumentation trace; `--host-header` for studio routes;
  `--cookie` for session-gated paths. Fails closed on a dist dir without
  `BUILD_ID` and on an occupied port, and warns on a non-200 response so a fast
  404 cannot masquerade as a fast page.
- `DEBUG_BOOT_TIMING=1` logs the boot split.
- `scripts/check-bundle-budget.ts` replaces the five-route `.mjs` gate: every
  route the build emits, per-surface budgets, boundary-aware matching,
  fail-closed on missing stats. CI builds into `.next-bundle` and gates on it.
- `outputFileTracingExcludes` for typescript, drizzle-kit and esbuild.
- `PAYLOAD_DB_PUSH` opt-in (`yarn dev:push`): `yarn dev` no longer runs the
  schema push.

## Tests

- `tests/int/instrumentation-boot.int.spec.ts` — the hook must not reach the
  Payload graph, must not warm the matrix, must stay silent unless timing is
  enabled. Mutation-checked: re-adding the import fails both checks.
- `tests/int/featured-studios-invalidation.int.spec.ts` — per-hook behaviour
  plus a coverage invariant that every hook invalidating studio `meta` carries
  the landing tag. Mutation-checked.
- `tests/int/theme-injection-owner.int.spec.ts` — which layout owns the theme,
  and why the root cannot: the platform default is dark and the CSS baseline is
  light. Mutation-checked (restoring the old root fallback fails it).
- `tests/int/measure-cold-start.int.spec.ts` — percentiles, `[BOOT]` parsing,
  trace classification, request headers, arg validation.
- `tests/int/bundle-budget.int.spec.ts` — reads a fixture rather than the
  working tree's `.next`, so it passes before `yarn build` in CI.
- `platform-landing-studios` pins `ARTIST_REGISTRATION_OPEN` instead of
  inheriting the developer's env; both open and closed states covered.

## Verification

`tsc --noEmit` clean · no import cycles · eslint 0 errors · `yarn check:bundle`
0 routes over budget · all specs above green.

## Not verified

- **Deployment.** All timings are local, with Postgres on localhost. The
  artifact reduction is measured but the instance-boot benefit it buys on Vercel
  (download + extract) is not — a two-preview-deployment A/B is still the proof.
  Check `x-vercel-cache: MISS` before trusting any single number.
- **Other studio routes.** `/` is measured (above). `/queue` and `/shop` were
  not; they go through the same hook and should show the same TTFB shape and the
  same caveat about the full document.
- **Payload init is not gone, only moved** — the first render that reads the
  database still initialises it, inside the request instead of before it. This
  is why the studio full-document number barely moves.
- **The prerendered shell — open, and this one is a real risk.** On the
  post-merge build, **every** `(frontend)` route writes a **0-byte** HTML file,
  where the baseline writes ~2.2 KB:

  | route | base | branch |
  |---|---|---|
  | `index.html`, `platform-home.html`, `studios.html`, `onboarding.html`, `login.html`, `prices.html`, `queue.html`, … | ~2.2 KB | **0 B** |
  | `dashboard.html` | 2581 B | 2301 B |
  | `_not-found.html`, `_global-error.html` | unchanged | unchanged |

  Evidence that it may be benign: both builds mark those routes
  `x-nextjs-prerender: 1`, both list them in `prerender-manifest.json`, runtime
  responses are HTTP 200 with comparable body-after-TTFB, and the branch's
  `_full.segment.rsc` is **larger** (152 KB vs 9.7 KB) — i.e. more content is
  captured, not less.

  Evidence that it may not be: a 0-byte HTML file is not a shell, so if the edge
  serves it, nothing arrives until the postponed part streams; and only the
  `(frontend)` group is affected, which points at this branch's root-layout
  change rather than a Next quirk.

  Not resolvable from build output alone, and the coding sandbox cannot bind a
  port to test it. **Check on the preview deployment** (see below) before
  trusting the "prerenders its shell" premise above.
- **Visual result of the theme fix.** The palette mismatch is proven from code;
  whether the swap was visible depends on first-paint timing on a given host.
  Worth one look at a light-themed studio on a cold load.

## How to confirm on a preview deployment

Everything above is local. The one claim this cannot settle is the instance-boot
benefit of the smaller bundle — Vercel downloads and extracts the deployment
before any code runs, and nothing local reproduces that.

Two preview deployments, same region / env / database: `origin/develop` and this
branch.

1. **Function, not CDN.** The document must report a miss — otherwise you are
   timing the edge cache:

   ```
   curl -sI https://<preview-host>/platform-home | grep -i x-vercel-cache
   ```

   Expect `MISS`.

2. **Bundle size** in the deployment's build output. What to look for: the
   instrumentation entry should no longer list `typescript`, `drizzle-kit` or
   `esbuild`, and its traced file count should drop from ~1700 to a handful.

3. **Boot log.** Set `DEBUG_BOOT_TIMING=1` on the preview environment only, then
   read the function logs:

   - branch: `[BOOT] … warmup none (lazy)` and `register` near 0 ms
   - baseline: `[TENANT-MATRIX] Warmup complete: N studios and M products … (Xms)`
     with `Xms` in the hundreds-to-thousands

4. **First request after idle.** One request per idle period (redeploy for
   independent samples); record `time_starttransfer`:

   ```
   curl -o /dev/null -s -w 'ttfb=%{time_starttransfer} total=%{time_total}\n' https://<preview-host>/platform-home
   ```

5. **The 0-byte shell (highest priority).** This is the open risk. On the branch
   deployment, check whether the document arrives with content on the *first*
   byte, not just eventually:

   ```
   curl -s https://<preview-host>/platform-home | head -c 400
   curl -sI https://<preview-host>/platform-home | grep -iE 'content-length|x-vercel-cache|x-nextjs'
   ```

   - Healthy: the response begins with `<!DOCTYPE html><html …` and carries a
     `content-length` in the low kilobytes, like the baseline.
   - Broken: the body is empty at first byte and everything arrives in the
     stream. That means the edge is serving a shell-less document, every request
     invokes a function, and the "prerenders its shell" premise is false.

   Compare both deployments side by side. If it is broken, do not merge before
   finding out which root-layout change removed the shell.

**What would falsify the artifact claim:** the instrumentation trace still
carrying `typescript`/`drizzle-kit`/`esbuild` on the branch deployment, or
`[TENANT-MATRIX] Warmup complete` appearing in the branch's logs.

**What would falsify the timing claim:** branch TTFB medians overlapping the
baseline's across ≥5 idle samples.

**What would falsify the prerender claim:** step 5 showing an empty document
body on the branch while the baseline returns HTML.

## Docs

- `docs/cold-start-plan.md` — phases, the three measurement runs (two pre-merge,
  one post-merge), the decision table, and the rejected options.
- `docs/cold-start-measurement-runbook.md` — copy-paste procedure, including
  the four layers, why Next's "Ready in" log is not the number, and the preview
  deployment protocol.
