# UB-Mode Improvement Plan

Status: Phase 0 shipped 2026-10-27 · Baseline measured 2026-10-27 13:40 (local Postgres, full `tests/int` under coverage)

This plan turns the ub-mode gates from prose into an enforced ratchet, then pays down
the debt in verifiable units. It is deliberately **not** a big-bang refactor: the
baseline is 799 functions over CRAP 10, so the first job is to stop new debt and only
then reduce the old.

---

## 1. Baseline (what the gates say today)

Measured with `crapper` (v8 coverage from the full int suite), `dryer`, and `mutator`
(all three in `~/.ub-tools/`, metrics in `.metrics/`).

**Complexity / CRAP** — 1,718 functions at the start; **re-measured 2026-10-27 15:10** after
Phase 1 and the first Phase 2 units: **1,725 functions, 794 over CRAP 10, 263 covered-and-over,
178 duplicate file pairs** (from 188). The table below is the original baseline, kept for the
before/after; `quality/baseline.json` always holds the current one.

| Group | Functions | Coverage >= 50% | CRAP > 10 |
|---|---:|---:|---:|
| `lib/*` | 483 | 373 | 159 |
| `features/*` | 540 | 226 | 295 |
| `collections/*` | 83 | 62 | 49 |
| `components/*` | 351 | 171 | 199 |
| `app/*` | 175 | 9 | 83 |
| `access/*` | 21 | 15 | 8 |
| **Total** | **1,718** | **875** | **799** |

Movements since that baseline (measured, not estimated):

- `registerArtistWithAI` CRAP **3192 -> 600** (CC 56 -> 24), `resolveOnboardingTheme` **506 -> 30**.
- `NavHeader` CRAP **669 -> 382** (CC 82 -> 73).
- `BlocksRenderer` CRAP **1379 -> 67.5** (coverage 0% -> 16 render tests; CC 56 -> 52).
- Six functions got *numerically* worse with unchanged CC: five are coverage attribution
  between the parallel run the first baseline came from and the serial runs used since, and one
  is deliberate — `createStripeExpressAccount` is 0% by default now that its only spec is opt-in
  (`RUN_STRIPE_SANDBOX_TESTS`). CI sets that flag when a Stripe test key exists.

- 465 of the 799 are **0% covered** — mostly admin UI that Playwright owns, not vitest.
- **255 of the 875 covered functions still exceed CRAP 10** — the real debt.
- Worst covered: `ReviewsModerationView` 763, `Image` 673, `NavHeader` 669,
  `PagesListView` 481, `enforcePageTypeRules` 313.
- Worst untested non-UI: `registerArtistWithAI` (CC 56), `postRoomMessageAction` (CC 50),
  `loginAction` (CC 44), `api/files/[id]` route (CC 46).

**Duplication** — 979 pairs at Jaccard >= 0.82. After removing generated migrations
(744), same-file pairs (65), and Payload boilerplate (10): **190 actionable pairs**.

- `lib/email/templates` — 20 pairs; five templates repeat the same
  `renderActionEmailLayout({...})` shape.
- `features/*/actions` — get-product / get-order / get-tip are 1.00 duplicates (~30 lines);
  studio-settings theme / notification / commission actions are 1.00 (~26 lines).
- `components/admin/ImageAlignmentField` ↔ `ImageGradientField` — 1.00, ~122 lines.
- 19 route-shell pairs across the `(platform)` / `(studio)` auth and legal pages.

**Mutation** — the harness is fixed (`@` aliases anchored to `process.cwd()`) and five
modules are re-measured: **137/141 killed (97.2%)**. `fee-schedule` is 46/46 with the
correct oracle (`product-net-pricing`), `commission-form-submit` 43/44, `action-crud` and
`mascot-guidance` 21/22 each, `redirect` 6/7. The 100% figures previously published were
produced by the broken overlay (see the correction in `docs/ub-mode-architecture.md`).

**Tests** — 230/242 int spec files green (2235/2281 tests). 11 files red, all
pre-existing (identical on the parent commit): missing `VERCEL_API_TOKEN`, QStash/Gemini
fallback paths, an FK residue from a deleted studio, DOM assertions.

**Churn x complexity** (6 months of `git log`): `artist-onboarding.action.ts` (34 commits,
CC 56, 0% covered), `NavHeader.tsx` (27, CC 82), `BlocksRenderer.tsx` (30, CC 56),
`CommissionForm.tsx` (23, CC 47), `AIThemeGeneratorAdminField.tsx` (23, CC 145),
`PlatformLandingPage.tsx` (23, CC 30).

---

## 2. Gate policy (decision required)

A single global "CRAP < 10" cannot be reached soon (799 failures) and would be the wrong
target anyway: a 150-line admin view rendered once and covered by Playwright is not the
same risk as an untested payment path. Proposed tiers:

- **Tier A — money, auth, tenant, data**: `lib/`, `features/*/actions`, `collections/hooks`,
  `access/`. Gate: **CRAP < 10**, and **100% of non-equivalent mutants killed** on the pure
  parts of any file touched.
- **Tier B — UI components**: `components/`, `features/*/components`, `app/**/page.tsx`.
  Gate: **CRAP <= 30** and CC <= 20, with behavior pinned by the existing Playwright suite.
  Anything above is a refactor candidate, not a merge blocker.
- **Excluded**: `src/migrations/**`, `src/payload-types.ts`,
  `src/payload-generated-schema.ts`, seeds and dev-only routes (`api/seed`).
- **Ratchet**: the gate fails on **regressions** against a committed baseline
  (`quality/baseline.json`: per-function CC and CRAP), not on the absolute backlog. Each
  iteration lowers the baseline.
- **CC fails, CRAP reports** (added 2026-10-27, after the first CI run): CC is structural and
  environment-independent, while CRAP is CC scaled by coverage — so it rises whenever a spec
  is *skipped*, not only when code gets worse. The first CI run failed seven functions whose
  CC had not moved at all, because `push`, S3 media and matrix specs skip when secrets are
  absent. The gate now fails on CC regression only and prints CRAP-only growth as
  `coverage drift (not failing)`; the absolute CRAP limit still applies to new functions.

Open decision: adopt the tiers as written, or merge Playwright V8 coverage into the
report and keep one global threshold. Merging is ~1 day of work but makes admin components
honest instead of exempt.

### What dryer cannot see (added 2026-10-27, after the email extraction)

Dryer scores pairs, not duplicated lines, and it normalizes literals away. Two consequences:

- **Sibling templates score 1.00 by construction.** The two quote emails were 57-line
  duplicates; extracting their shared MJML card removed ~40 lines and left the score at
  1.00, because the two functions still have the same shape — they *are* the same email
  with different words. The same applies to the five action emails and to per-collection
  CRUD wrappers (`theme` / `notification-settings` / `commission-settings`): identical
  shape, different collection name.
- **Deletion barely moves the number.** 744 of the 974 pairs are generated migrations and
  another ~65 are same-file; the actionable set is ~190, and most of it is shape, not logic.
- **Two different slugs look identical.** `subdomainFromStudioName` (hyphens, no underscores)
  and `deriveProductTypeSlug` (underscores) score 1.00 together. They are deliberately
  separate — unifying them would break one of the two formats — and the decision is
  recorded above the function that might otherwise be "simplified" into the other.

So the gate should not be optimized for pair count. The pairs worth removing are ones where
the duplicated block is *logic* (a decision, a rule, a sequence of effects). For shape
duplication the honest move is to extract the shared layout/helper when it buys a real thing
(here: one place to change the email card) and otherwise leave the copy readable. A future
revision should score duplication in lines or scope dryer to logic-bearing modules.

---

## 3. Phases

### Phase 0 — Make the gates enforceable (shipped 2026-10-27)

- `scripts/quality-gates.mjs` — ratchet over `.metrics/crap.edn` and `.metrics/dry.edn`:
  new functions must meet their tier limit, existing functions may not regress structurally
  (CC +1; CRAP-only growth is reported as coverage drift, see §2), new duplicate file-pairs
  fail. `--write-baseline` refreshes the snapshot; `--strict` applies the absolute limits to
  legacy code too.
- `quality/baseline.json` — 1,804 functions (1,083 Tier A, 675 Tier B) and 178 duplicate
  file-pairs grandfathered (refreshed after the session's extractions; the numbers at the
  start of the campaign were 1,718 / 188).
- `yarn quality:baseline` / `yarn quality:gates`; `/target` ignored.
- `.github/workflows/quality-gates.yml` — Postgres 16 service, migrations **and `yarn seed`**
  (the first run had a migrated-but-empty database, so every spec that picks an existing
  studio failed), `test:int --coverage`, pipx-installed crapper/dryer, gate script.
- **Shipped tooling became visible again**: `.gitignore`'s `/scripts/*` had no re-include for
  `check-vercel-errors.ts`, `sync-s3-media.ts` or `migrate-ronya-to-artpack.ts`, so CI ran a
  spec against a file that only existed on one machine and `yarn sync:s3` pointed at nothing.
  All three are now committed.
- `tenant-matrix.int.spec.ts` mocks `@/lib/infra/upstash` — it silently required a real Redis
  credential in the environment and asserted against a pipeline that was never called.
- `docs/ub-mode-architecture.md` section 4 corrected with the real mutation results.

Still open from Phase 0: mutator is not yet part of the gate script (it needs the changed
pure modules and their specs), and the CI job must be proven on a real PR.

**Exit**: a PR that adds a CC-40 untested function to `lib/` fails CI; a PR that leaves
duplication at 0.9 fails CI; `main` is green.

### Phase 1 — Stop the bleeding (in progress)

**Suite triage shipped 2026-10-27** — the 11 red files are down to 1, and three of them
were real product defects, not test drift:

- `ResponsiveNavPortal` returned `null` for theme routes before the breakpoint check, so
the phone burger opened nothing. Fixed; the desktop exclusion now stops at 769px.
- `get-user.action` statically imported `@payload-config` and is imported by collection
endpoints, closing a cycle that left a collection `undefined` while
`withAdminLeaveGuard` mapped over the array. Broken at two points; 36 modules still
import the config statically (structural pass, Phase 2).
- `stripe-connect-v2` used a fixed artist id, permanently burning its Stripe idempotency
key (409 on payload change). Its Accounts v1 fallback is now rejected outright by Stripe —
the fallback is dead weight and a failed v2 leaves a confusing error (product decision).

**Suite is green**: `242/242` files (240 passed, 2 skipped), `2274/2285` tests, serial.
The last red file was the same cycle: `Orders.ts -> sendOrderReviewInvite.ts -> @/features/reviews`
(barrel) pulled every review action, `create-product-review.action` imported `@payload-config`,
and the config re-entered `Orders` while it was still evaluating. The endpoints now import
`get-reviews.action` directly.

**Cycle class removed**: every module reachable from `payload.config` now loads the config
through `resolvePayloadInstance()` (lazy) instead of a static `@payload-config` import — 23
files, the two leftovers being an unused import and a `profile-actions` call. `scripts/check-import-cycles.mjs`
reports zero cycles and runs as `yarn quality:cycles` in CI, so the class cannot grow back
silently.

Parallel runs still flake (5s timeouts, failed inserts, global row counts) because 242 files
share one Postgres; CI runs `--no-file-parallelism` (193s vs 73s).

Still open in this phase:

1. **Fix the parallel flakiness** — `dashboard-stats` counts global users, `shop-multi-tenant-media`
   times out at 5s under load, `push-subscription-validation` sees a stale endpoint row.
   Either keep the CI run serial or make the fixtures tenant-scoped.
2. **Kill the boilerplate duplicates** (targets 190 -> <= 120 actionable pairs):
   - quote emails: shared footer card extracted (40 lines less MJML, byte-identical
     output, pinned by `email-quote-templates`). Dryer still scores the pair 1.00 —
     the two functions are structurally identical because they are the same email
     with different words; collapsing further means a data table, not clearer code.
   - the five action emails -> one data-driven `actionEmail({title, body, cta, footer})`
     table (removes 20 pairs);
   - settings actions -> one `upsertTenantSettings(collection, tenantId, data)` helper
     (removes ~6 pairs);
   - `ImageAlignmentField` + `ImageGradientField` -> **shipped 2026-10-27**: one
     `createSegmentedIconField({ options, defaultValue })` control; each field file is now its
     option table (122 duplicated lines removed, 8 render tests).
   - get-product / get-order / get-tip -> the existing `executeFindById` from
     `lib/tenant/action-crud.ts` (removes 3 pairs).
3. **Delete dead code** — `ForgotPasswordView`, `ResetPasswordView` (exported, never used;
   pages inline the same `Suspense`), finish adopting `redirectIfUser` in the studio
   login/register routes, and drop the duplicate `Suspense` wrappers.
4. **`action-crud.ts` semantics — closed, no change needed.** The concern was that the
   helper converts errors into `null`/`false` where the actions it replaced threw. Audited
   against `05565bc^`: `delete-product-type`, `delete-queue-item`, `update-queue-item`,
   `update-product`, `delete-commission`, `get-product`, `get-order` all caught and swallowed
   before the refactor, and the create paths (`create-product`, `create-sprint`,
   `create-commission`) had no handler — `executeCreate` rethrows, so they still propagate.
   `tests/int/action-crud.int.spec.ts` already pins all three outcomes. Residual smell, not a
   regression: `null` conflates "not found" with "query failed"; distinguishing them would be
   a new contract, not a repair.

**Exit**: 242/242 int specs green; actionable duplicates <= 120; no dead exports in
`features/auth`; the CRUD helper no longer masks errors.

### Phase 2 — Hotspot refactors, one unit per PR (2–3 weeks)

Order = CRAP x churn. Each unit follows the same loop:
**pin behavior with tests -> extract pure functions -> mutation-test the pure part ->
prove CRAP < 10 -> commit alone.**

1. `artist-onboarding.action.ts` — **partly shipped 2026-10-27.** The theme default table
   (written twice, disagreeing on `useLiquidGlass`), the commission-settings input, the two
   relationship-id merges and the catch-block error mapping moved to
   `features/onboarding/lib/onboarding-records.ts` with 13 assertions and a mutation run
   (31/33, two equivalent survivors). CC: `registerArtistWithAI` **56 -> 24**,
   `resolveOnboardingTheme` **22 -> 5**. Still open: the DB orchestration itself (CC 24, no
   integration test) — the next unit, and the one that needs a transaction-level test.
2. `NavHeader.tsx` — **partly shipped 2026-10-27.** Link derivation and the "More" overflow split
   moved to `lib/nav/header-nav.ts` (9 unit tests) and the render spec gained the overflow cases
   (5 links on floating -> 3 + More, opening reveals the rest; 4 links flat; showQueue=false drops
   the queue). CC **82 -> 73** — the remainder is JSX style ternaries (`isBrutalist` alone appears
   21 times), which a helper would not remove. Still open: `resolveHeaderChrome()` if those styles
   ever need to change.
3. `AIThemeGeneratorAdminField.tsx` — CC 145, 23 commits. Split prompt handling / apply /
   undo; the apply path is already covered by `ai-theme-generation`.
4. `BlocksRenderer.tsx` — **shipped 2026-10-27.** Tenant resolution moved to
   `lib/blocks/tenant-scope.ts` (4 tests); the dispatcher now has 16 render tests with every child
   mocked, pinning which component each of the 11 block types mounts, that unknown types and empty
   lists render nothing, and that the carousel only appears when the selection resolves. CC
   **56 -> 52** on the switch itself; coverage **0% -> 20 tests**. A component map would trade eleven
   cases for eleven prop-builders, so the switch stays.
5. `CommissionForm.tsx` — CC 47 (from 131), 23 commits. Finish the split started in
   `63a65cf`; the submit logic is now tested, the component body is not.
6. `ReviewsModerationView` (763) and `enforcePageTypeRules` (313) — the two worst covered
   functions.
7. `syncProductToStripe` (161), `createCommissionCheckoutSession` (121),
   `getReviewQueue` (112), `applyCropRectToUploadEdits` (107), `defaultThemeValues` (106),
   `tenant-tag.resolveTenant` (99).

**Also shipped 2026-10-27:** `getCombinedStudioArtistData` — the worst remaining Tier A
function (CRAP 952, CC 70, no unit tests, runs on every studio settings page). The six-source
studio-id cascade and the three copies of media population moved to
`features/studios/lib/studio-profile.ts` and `lib/media/resolve-media-document.ts`; CC
**70 -> 31**, 12 tests, mutation 15/20 with five equivalent survivors.

The write-side sibling `saveStudioArtistProfile` followed (CRAP 322, CC 46 -> 29):
`canEditStudioProfile`, `buildStudioUpdateData` (admin-only subdomain, null-vs-absent per field),
`buildArtistUpdateData` and `buildArtistCreateData`; 14 tests, mutation 36/40 (90%).

**Exit**: covered-but-CRAP>10 count 255 -> <= 150; every file in the list has a passing
mutation score on its pure part; no behavior change (pinned by tests written first).

### Phase 3 — Untested critical logic (ongoing, 2–4 weeks)

89 non-UI functions with CC >= 15 have 0% coverage. Triage:

- **Pay now**: `postRoomMessageAction` (50), `loginAction` (44), `api/files/[id]` (46),
  `api/deliverables/[id]`, `getCombinedStudioArtistData` (73), `performDeepCleanup` (57).
  Each gets a behavior spec plus a mutation run.
- **Pay later / accept**: seeds, dev-only routes, one-off scripts.
- **Never**: admin views already exercised by Playwright — unless Playwright coverage is
  merged.

**Exit**: zero CC >= 15 functions with 0% coverage in Tier A paths.

### Phase 4 — Structural hardening (after Phase 2)

1. **Type the boundaries**: `commission-form-submit` takes `user: any`, `sectionGroups: any[]`,
   returns `Record<string, any>`; `buildCommissionSubmitData`'s result is cast `as any` at
   the call site. Replace with a zod parse at the entry and a `CommissionSubmission` DTO.
2. **One rule, one place**: `drawingTier !== 'icon' && !background` exists in both
   `commission-form-submit` and `mascot-guidance`. Extract `requiresBackground()`.
3. **Illegal states**: `selectedTiers: Record<string,string>` duplicates
   `drawingTier`/`customTier`/`background`; the form keeps two representations. Model one
   (`TierSelection` union) and derive the other.
4. **UI effect out of the lib**: move `fireSubmissionConfetti` to the component layer; the
   submission lib stays pure.
5. **Equivalent-mutant policy**: document that `?? 0` vs `|| 0` on a `number | null` is
   equivalent and excluded from the 100% target.

---

## 4. Verification per phase

| Phase | Command | Pass condition |
|---|---|---|
| 0 | `node scripts/check-quality-gates.mjs --changed` | exits 0 on `main`, 1 on a seeded regression PR |
| 1 | `yarn test:int` | 242/242 files, 2281 tests |
| 1 | `dryer --source-root src src` | actionable pairs <= 120 |
| 2 | `crapper --use-existing-coverage` | touched functions CRAP < 10; global covered->10 count down |
| 2 | `mutator` on each touched pure module | >= 95% killed, survivors documented |
| 3 | `crapper` filtered to `cov==0 && CC>=15` | zero in Tier A paths |
| 4 | `tsc --noEmit` + boundary spec | no `any` in commission submission input/output |

Every iteration ends with: metrics regenerated, baseline lowered, one commit per unit,
and a one-line proof in the commit message.

---

## 4b. Session log — 2026-10-27

Seven units shipped in one session. Each was: pin behavior with tests, extract the pure
decision, mutation-test it, prove the CC/CRAP drop, commit alone.

| Unit | CC | CRAP | Evidence |
|---|---|---|---|
| `commission-form-submit` (pre-session example) | 3–8 | max 8.0 | 23 tests, mutation 43/44 |
| `artist-onboarding.action` | 56 -> 24 | 3192 -> 600 | 13 tests, mutation 31/33 |
| `resolveOnboardingTheme` | 22 -> 5 | 506 -> 30 | (same module) |
| `NavHeader` | 82 -> 73 | 669 -> 382 | 9 unit + 3 render tests |
| `BlocksRenderer` | 56 -> 52 | 1379 -> 67.5 | 16 render + 4 unit tests |
| `getCombinedStudioArtistData` | 70 -> 31 | 952 -> 214 | 12 tests, mutation 15/20 |
| `saveStudioArtistProfile` | 46 -> 29 | 322 -> 162 | 14 tests, mutation 36/40 |
| `syncProductToStripe` | 33 -> 14 | 161 -> 47 | 20 tests, mutation 39/43 |
| `createCommissionCheckoutSession` | 36 -> 14 | 121 -> 42 | 21 tests, mutation 32/36 |
| `requireDashboardAccess` | 34 -> 29 | 1190 -> 870 | 10 tests, mutation 8/9 |
| `loginAction` | 44 -> 32 | 1980 -> 1056 | 10 tests, mutation 8/8 (100%) |
| `/api/files/[id]` | 46 -> 27 | 2162 -> 756 | 17 tests, mutation 27/28 (96.4%) |
| `postRoomMessageAction` | 50 -> 26 | 2550 -> 702 | 15 tests, mutation 35/35 (100%) |
| `submitCommissionReviewAction` | 38 -> 18 | 1482 -> 342 | 13 tests, mutation 18/18 (100%) |
| `performDeepCleanup` | 57 -> 39 | 3306 -> 1560 | 14 tests, mutation 13/13 (100%) |
| `useOnboardingForm` | 88 -> 84 | 7832 -> 7140 | 10 tests, mutation 8/8 (100%) — helpers only |
| `createStripeExpressAccount` | 25 -> 2 | 650 -> 6 | 11 tests, mutation 18/19 (94.7%) — v1 fallback removed |
| `updateRound` | 32 -> 14 | 1056 -> 210 | 14 tests, mutation 19/20 (95%) |
| `getEffectiveTheme` | 27 -> 3 | 756 -> 12 | 14 tests, mutation 20/22 (90.9%) |

**Cumulative, from the original baseline to the refreshed one:** 1,718 -> 1,765 functions
(the extractions added 47), **CRAP > 10: 799 -> 794**, sum of CRAP **217,862 -> 212,477**
(-5,385 since the first checkpoint), duplicate file pairs **188 -> 178**. Tenant/auth reads and
writes that carry money moved from "no unit tests" to mutation-scored pure modules.

**Infrastructure that outlasts the session:** `scripts/quality-gates.mjs` (ratchet over
CRAP + duplication), `quality/baseline.json`, `scripts/check-import-cycles.mjs` (a class of
crash that hit twice, now a CI gate), `@vitest/coverage-v8` + lcov, `reportOnFailure`, the
`@` alias fix that makes mutation testing meaningful, serial CI, and the corrected
`docs/ub-mode-architecture.md`.

**Two corrections made along the way**, both worth remembering:

- The `action-crud` "swallows errors" concern was wrong: the pre-refactor actions already
  swallowed, and `tests/int/action-crud.int.spec.ts` pins the contract. The item was
  retracted rather than "fixed".
- Dryer counts *shapes*, not logic: two sibling emails score 1.00 forever. The gate now
  documents what it cannot see, and the plan no longer chases pair count.

---

## 5. Risks / open questions

1. **Tier policy** — adopt Tier A/B, or merge Playwright coverage and keep one threshold?
   Without a decision, Phase 0 cannot fail the build.
2. **CI database** — a Postgres service container is ~15 lines of workflow but needs
   migrations to run fast; the Supabase test DB is the alternative (network + secrets in CI).
3. **Parallel writer** — `artpack-parallel` / codex worktrees are actively editing
   `PlatformLandingPage.tsx`, `action-crud.ts`, and shop/queue actions. Phase 1.2 targets
   those same files; ownership must be agreed before starting.
4. **Local teardown** — the int suite writes into the dev database (30 studios). Phase 1
   should add tenant-scoped cleanup helpers so repeated runs do not leave FK residue.
5. **Effort** — Phase 0+1 are days; Phase 2 is weeks at ~1 unit/day; Phase 3 depends on
   how many of the 89 functions are real risk versus dev-only code.
