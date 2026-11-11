# Cold-Start Measurement Runbook

Step-by-step procedure for measuring whether the cold-start work on
`perf/cold-start-clean` actually improved anything. Written to be copy-pasted from
this file.

Companion docs:
- `docs/page-load-performance-remaining-plan.md` §9 — why these layers exist and
  why Next's own "Ready in" log is not the number.
- `docs/caching-strategy-and-scaling.md` — the shared `use cache` handler, which
  is the lever that fixes "test is always cold" (not part of this branch).

---

## 0. What is being measured

Cold start has four layers. A single TTFB number cannot tell them apart, so the
run below measures them separately.

| Layer | What it is | How it is checked |
|---|---|---|
| 1. Artifact | Files Vercel downloads and extracts before any code runs | `--artifacts` on the build's `.nft.json` traces |
| 2. Process | Node spawn + module graph evaluation until the port listens | `listen` column of `yarn measure:cold` |
| 3. Warmup | `instrumentation.register()` — tenant matrix DB + Redis | `warmup` / `register` columns from the `[BOOT]` line |
| 4. Request | First render, with an empty per-instance `use cache` | `first byte` column |

Layer 1 needs only a build. Layers 2–4 need a running server and a reachable
Postgres.

---

## 1. Branches

| Branch | Commit | Purpose |
|---|---|---|
| `origin/develop` | `16213dc` | **Baseline for the A/B.** `perf/cold-start-clean` is cut from here, so diffing against it isolates exactly these commits. |
| `perf/cold-start-clean` | branch head | The work under test. |
| `perf/cold-start-base` | `20af792` | Older baseline: the escrow branch tip, 10 commits ahead of `develop`. Only use it if you deliberately want the escrow work in the comparison. |

`develop` can be used instead of `origin/develop`, but it also contains unrelated
recent merges, so a difference can no longer be attributed to the cold-start
work alone.

---

## 2. Prerequisites

Postgres must be running — every storefront page reads from it.

```
brew services start postgresql@16
```

Check it answers:

```
pg_isready -h 127.0.0.1 -p 5432
```

Expected: `127.0.0.1:5432 - accepting connections`.

No dev server on port `3211` (the harness refuses an occupied port) and enough
disk for two extra builds (~2 GB).

**Network access is required for the build.** `(frontend)/layout.tsx` uses
`next/font/google` (Sora), which downloads font files at build time. Without
network the build hangs at `Creating an optimized production build ...`
indefinitely — it does not fail, it waits. A sandbox with network blocked cannot
produce the arms.

Verify before starting:

```
curl -s -o /dev/null -w '%{http_code}\n' -m 8 https://fonts.googleapis.com/css2?family=Sora
```

Expected: `200`. Anything else (or `000`) means the build will hang — run it
outside the sandbox, or on a machine with egress.

---

## 3. Kill any wedged build, clear stale arms

A previous build in this session stalled with no output. Stop it and remove the
partial output before starting:

```
pkill -f "next build"
```

```
rm -rf .next-base .next-branch
```

---

## 4. Build the baseline arm

```
git checkout --detach origin/develop
```

```
NEXT_DIST_DIR=.next-base yarn build
```

Confirm the build finished:

```
cat .next-base/BUILD_ID
```

Expected: a build id string, not "No such file".

---

## 5. Build the candidate arm

```
git checkout perf/cold-start-clean
```

```
NEXT_DIST_DIR=.next-branch yarn build
```

```
cat .next-branch/BUILD_ID
```

Both arms must be built with the same environment (same `.env`, same
`NODE_ENV`). The build sets `NODE_ENV=production` itself.

---

## 6. Layer 1 — artifact check (no server, no database)

```
yarn measure:cold --artifacts --a .next-base --b .next-branch
```

This reads every `.nft.json` trace in both arms and reports, per trace, total
size and any `typescript` / `drizzle-kit` / `esbuild` files that
`outputFileTracingExcludes` was supposed to remove.

In two-arm mode the arms are reported as **baseline** and **candidate**:

- `EXPECTED` on the baseline — it predates the exclude config, so carrying these
  packages is correct.
- `PASS` on the candidate — the excludes match what the build emits.
- `FAIL` on the candidate — the globs do not match. Exit code `1`.

**Measured 2026-11-09** (`origin/develop` → `perf/cold-start-clean`):

| Trace | Baseline | Candidate |
|---|---|---|
| `/platform-home` | 50.2 MB (18.8 leaked) | **30.9 MB** |
| `/` (studio home) | 75.2 MB (18.8 leaked) | **31.6 MB** |
| `/queue` | 75.3 MB (18.8 leaked) | **31.7 MB** |
| `/login` | 50.3 MB (18.8 leaked) | **31.2 MB** |
| `/studios` | 50.2 MB (18.8 leaked) | **30.9 MB** |
| `server/instrumentation.js` | 77.3 MB (46.2 leaked) | **0.0 MB (0 leaked)** |

All 77 route traces are clean on the candidate; total route bytes
3854.5 MB → 2200.8 MB (−43%).

**The instrumentation trace cannot be fixed with a glob.** Verified in Next
16.4.4: `build/collect-build-traces.js` applies `outputFileTracingExcludes`
while walking `buildTraceContext.chunksTrace.entryNameFilesMap`, whose keys are
route entries (`app/…`, `pages/…`). `server/instrumentation.js.nft.json` is not
one of them and the module never appears in that file — Turbopack writes that
trace directly.

The tool reports a retained instrumentation trace as a `NOTE` and exits 0
(use `--strict` to fail on it). The remedy is to stop importing the Payload
graph from `src/instrumentation.ts` — done on this branch, which is why the row
above now reads 0.0 MB.

`yarn measure:cold --artifacts --strict --dist .next-branch` exits **0** on this
branch, strict included.

Single-arm mode (`--dist .next-branch`) always treats route leftovers as a
failure, so it can be used as a check on its own.

**Pass for the candidate arm:**

- `PASS  all N route traces are clean` and exit code `0`.
- A `NOTE` about the instrumentation trace is expected and is not a failure —
  see above for why no glob can reach it.

**Fail** on a route trace means the globs in `next.config.mjs` do not match what
the build emits. The current patterns were verified against the same matcher Next
uses (`picomatch` with `{dot: true, contains: true}`), including the negative
case: `typescript-eslint` and `@esbuild-kit` are **not** matched.

This step needs no database and binds no port, so it is also the cheapest way to
check that a fresh build is worth measuring.

---

## 7. Layers 2–4 — cold-start A/B

```
yarn measure:cold --a .next-base --b .next-branch --samples 5 --path /platform-home
```

What it does: for each sample it starts a fresh `next start` on a free port,
waits for the listen socket, sends exactly one request, then kills the process.
Nothing is reused, so every sample pays the real first-request path —
`handleRequest` awaits `prepare()`, which loads `instrumentation` and awaits
`register()`. Order alternates `A B B A` so drift lands on both arms.

Useful variants:

```
yarn measure:cold --a .next-base --b .next-branch --samples 5 --path /platform-home
```

Studio routes need a `Host` header, because the harness connects to `127.0.0.1`
and the app resolves that to the platform root. Use a studio that exists in the
database you are pointed at:

```
yarn measure:cold --a .next-base --b .next-branch --samples 5 --path / --host-header my-studio.localhost:3211
```

Session-gated paths need a cookie:

```
yarn measure:cold --a .next-base --b .next-branch --samples 5 --path /dashboard --cookie "payload-token=<token>"
```

**A non-200 response prints a `WARN`.** A fast 404 is not a fast page — a wrong
`--host-header` yields a beautiful cold-start number for the not-found route.
Check for that warning before believing a studio-route result.

`--json` prints the raw samples. `--port <n>` if `3211` is busy. `--host` defaults
to `127.0.0.1`.

### Reading the output

A line per sample:

```
branch #1: listen 131 ms | first byte 294 ms | end 1038 ms | warm 15 ms | next-ready-in 51 ms | warmup n/a | process->ready 136 ms | http 200
```

- **`listen`** — spawn to listen socket (layer 2).
- **`first byte`** — the cold request (layers 3+4). This is the headline.
- **`end`** — full document for the cold request.
- **`warm`** — a **second request to the same process**. This is the column that
  separates one-time-per-instance init from per-request cost. If `end` is large
  and `warm` is tiny, the cost is paid once per instance and every later visitor
  is fast; if `warm` is large, the render itself is expensive. On the studio home
  this read 15–16 ms in both arms while the cold document was ~1 s — which is why
  removing the boot warmup improved HTML arrival but not the total there.
- **`next-ready-in`** — Next's own `Ready in` log. It is emitted *before* the
  config loads and before `register()` runs, so it excludes the warmup. The gap
  between this and `first byte` is the cost that log hides.
- **`warmup`** — `[TENANT-MATRIX] Warmup complete … (Xms)`. `n/a` on this branch
  is correct: the hook no longer warms anything.
- **`process->ready`** — end of `register()`, i.e. when the instance can actually
  serve.

Then a summary block with median/p90/min/max per metric, and a
`Candidate vs. baseline` section with Δmedian and Δp90.

Percentiles are nearest-rank and medians are used instead of a mean: cold starts
are bimodal (host page cache, CPU steal, pooler handshake) and a mean hides that.

### Interpreting the result

- **`first byte` median not worse** — the batch did not regress the cold request.
- **`warmup` shows a number** — the hook is doing boot work again. It should read
  `n/a` here because `register()` no longer touches the tenant matrix; a value
  means `tests/int/instrumentation-boot.int.spec.ts` was bypassed or the line is
  coming from an older build.
- **`register` at ~0 ms** — correct for the current branch. It was 544 ms
  (531 of it warmup) before 2026-11-09.
- **Large `listen`, small `register`** — the remaining cost is the module graph /
  process start; the artifact and import work is what moves it.
- **Ranges that overlap** — the delta is noise at n=5. Increase `--samples`
  before drawing a conclusion; the 2026-11-09 result was only credible because
  baseline min (984 ms) exceeded branch max (310 ms).
- **Local numbers are a ratio, not a forecast.** A local Postgres on a Unix
  socket makes warmup look cheap; a remote pooler is where the seconds live.

### Measured reference — 2026-11-09

`/platform-home`, 5 samples per arm, baseline = `origin/develop`:

| metric | baseline | branch |
|---|---|---|
| listen | 131 / 131 p90 | 131 / 132 |
| **first byte** | **1015 / 1089** | **291 / 310** (−71.3%) |
| register | n/a | 0 ms |
| warmup | n/a | none (lazy) |
| `next-ready-in` | ~51 ms | ~51 ms (unchanged) |

The unchanged `next-ready-in` is the point of the whole procedure: Next's own log
never showed the 531 ms warmup or the time it cost.

---

## 8. The verdict — test deployment

Only Vercel answers the real question, because function boot there includes
downloading and extracting the deployment bundle.

1. Deploy `origin/develop` (or the `perf/cold-start-base` commit) and
   `perf/cold-start-clean` as two preview deployments — same region, same
   environment variables, same database.
2. Set `DEBUG_BOOT_TIMING=1` on the preview environment only (never production).
3. Confirm the request hit a function, not the CDN:

```
curl -sI https://<preview-host>/platform-home | grep -i x-vercel-cache
```

Expected: `x-vercel-cache: MISS`. A `HIT` measures the edge cache, not a cold
instance, and the number is meaningless for this purpose.

4. Send **one** request per idle period. Test has no traffic, so an instance is
   recycled quickly; redeploying is the cheapest way to get independent samples.
5. In the function logs, read the `[BOOT]` line and the
   `[TENANT-MATRIX] Warmup complete … (Xms)` line for each deployment.
6. Compare medians of the first request per deployment, plus the warmup `Xms` —
   never a single curl.

```
curl -o /dev/null -s -w 'ttfb=%{time_starttransfer} total=%{time_total}\n' https://<preview-host>/platform-home
```

---

## 9. What each change can and cannot show

| Change | Layer | Expected result |
|---|---|---|
| `outputFileTracingExcludes` | 1 | Measurable: typescript/drizzle-kit/esbuild gone from traces |
| Direct action imports (barrel removal) | 2 | Smaller module graph; shows in `listen` and cold module eval |
| `PAYLOAD_DB_PUSH` opt-in | dev only | **Zero** on a deployment — production never ran the schema push. Do not grade it down for this. |
| `cacheLife('hours')` | 4, warm instance only | Little effect on a scale-to-zero deployment; most requests still start with an empty `use cache` |
| `/platform-home` PPR shell | 4 (perceived) | Better first paint and smaller payload; not instance init |
| `getEffectiveTheme` short-circuit + React cache | 3/4 | Removes a headers read and a layout-cache pass on platform routes |
| `getCachedFeaturedStudios` select/depth | 4 | Less work on a cold landing render |

---

## 10. Troubleshooting

**`listen EPERM` / `port already in use`** — stop whatever holds the port:
`yarn measure:cold --port 3212`. On macOS with a dev server on `3000`, the harness
is unaffected; it only cares about the port it is told to use.

**`dist dir has no BUILD_ID`** — the build did not finish. Re-run the build for
that arm; `--artifacts` also refuses an unfinished build.

**`no .nft.json traces`** — same cause: an incomplete build.

**Warmup shows `n/a`** — the database was unreachable and `register()` swallowed
the error (by design: it must not block boot). Start Postgres and re-run.

**Numbers swing wildly between runs** — expected on a laptop. Increase
`--samples` and read the median; do not compare single samples.

**Everything is fast locally but slow on test** — almost always network: a local
socket vs. a remote Postgres pooler. This is why §8 exists.

---

## 11. One-liner status check

After the run, the branch should still be green:

```
yarn test:int tests/int/measure-cold-start.int.spec.ts tests/int/bundle-budget.int.spec.ts tests/int/platform-landing-studios.int.spec.ts
```

```
npx tsc --noEmit
```
