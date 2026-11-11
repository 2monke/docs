---
status: shipped
collections: [commission-settings, commissions, pages, products, tips]
features: []
---

# Admin Theme Token Contract & Cross-View Unification

Single visual language for every Payload admin page: **one surface chrome, one
accent, one radius scale, and no drop shadows on static surfaces**. The custom
views (Studio/Artist, Commission Settings, Payouts, Dashboard, Pages list) and
Payload's built-in views (collections, document edit views, account) all consume
the same token contract and recipe layer.

## 0. Flat-surface language (2026-09-19 recalibration)

The first unification pass extended the custom views' *liquid-glass* look to the
built-ins, which left the admin reading as a stack of tinted, shadowed boxes:
cards at 16–20px radius with 30–50%-opacity black shadows, every collection row
floating on a drop shadow with a hover lift, page titles at 22px on custom views
vs 32px on built-ins, and Payload's own 3px pills next to 8px custom buttons.

The recalibration reverses the *direction* of convergence — instead of dressing
the built-ins up, the custom views were flattened down onto Payload's native
calm, and Payload's geometry was re-pointed at the shared scale:

1. **No shadows on static surfaces.** Cards, collection rows, page headers, list
   headers, hero/avatar tiles and primary buttons carry no `box-shadow` in any
   state. Shadows survive only on genuine overlays (`--ap-shadow-float`,
   `--ap-shadow-popover`: drawers, popovers, menus, toasts, the tour) plus focus
   rings. Hover feedback is a border/background colour change, never a lift.
2. **One card chrome.** `--ap-card-bg` is now `--theme-bg` (cards are the page
   surface, not a tinted panel), with a 1px `--ap-card-border` hairline and the
   single `--ap-radius-xl` (14px) radius — the same chrome as Payload's `.card`,
   the collection grid rows and the pages list groups.
3. **No nested boxes.** Settings rows (`.cs-control-box`), tier items
   (`.cs-tier-item-block`) and social links (`.studio-artist-social-item`) are
   hairline divider rows inside their section card at *every* breakpoint — the
   mobile flattening rule is now the desktop rule too.
4. **Page headers are chrome-less.** `.studio-artist-hero`, `.cs-header` and
   `.commissions-list-header` draw no fill, border, radius or shadow: a page
   header is not a surface.
5. **Navigation is chrome-less.** the segmented `ap-tabbar` no longer renders a
   container box; the tinted active pill is the only signal.
6. **One title scale.** `--ap-title-page` (32px / 600) is used by built-in
   `.list-header__title` + `.doc-header__title` and by every custom page title.
7. **Payload inherits our geometry.** `--style-radius-s/m/l` are re-pointed at
   `--ap-radius-sm/md/xl` (8/10/14px) so built-in buttons, inputs, code editors,
   paginator, file fields and drawer controls join the scale without new
   overrides; `.pill` (which Payload hardcodes to 3px) is explicitly mapped.
8. **One accent.** The `--ap-accent-*` hsl family and the `--ap-accent-hex-*`
   names are aliases of the same `#6366f1` ramp, and every alpha tint derives
   from it — previously tabs/sidebar used `hsl(241,67%,67%)` while buttons and
   gradients used `#6366f1`.
9. **Desktop tail.** `--ap-page-pad-bottom` (110px) is the mobile save-dock
   budget; desktop uses `--ap-page-pad-bottom-desktop` (48px).

## 1. Token contract

`src/app/(payload)/styles/_theme-tokens.scss` — the `--ap-*` block inside
`@layer payload` is the **single source of truth** for admin visual values:

- **Accent** — `--ap-accent-300…700` (hsl indigo scale) + hex equivalents
  `--ap-accent-hex-*` for gradients/glows + alpha tints `--ap-accent-a10…a60`.
- **Status** — `--ap-status-success/-warning/-danger` (+ `-bg`/`-border` and
  `-strong` variants).
- **Surfaces** — `--ap-card-bg/border/input-bg` aliasing Payload `--theme-*`
  (`--ap-card-bg` = `--theme-bg`; cards are not tinted panels).
- **Radius scale** — `--ap-radius-micro…5xl` (3/4/6/8/10/12/14/16/18/20/24px) +
  `--ap-radius-pill` (999px), plus the Payload bridge `--style-radius-s/m/l`
  → `--ap-radius-sm/md/xl`.
- **Geometry/type** — `--ap-control-height: 44px`, `--ap-title-page: 32px`,
  `--ap-font-*`, `--ap-gutter-mobile/desktop`, `--ap-page-pad-bottom: 110px`
  (mobile) / `--ap-page-pad-bottom-desktop: 48px`.
- **Elevation** — overlays only: `--ap-shadow-float/popover`, `--ap-focus-ring*`.
  Static-surface shadow tokens (`--ap-shadow-btn/card/card-lg/card-hover`,
  `--ap-shadow-glow*`) were deleted by the flat-surface pass — reintroducing one
  is a contract violation, not a styling choice.
- **Motion** — `--ap-transition: all 0.15s ease`.
- Disabled grays (light/dark), muted text, scrollbar chrome, liquid tints.

No hardcoded accent colors, radii, shadows or control geometry belong in
`_admin-theme.scss`, `_blocks-ui.scss` or any view CSS. Residual intentional
literals: the multi-hue commission status palette and social brand colors.

## 2. Recipe layer

`src/app/(payload)/styles/_admin-primitives.scss` — mixins shared by custom
views and the Payload override layer:

- `ap-page-gutter` — the standard content frame (8px/24px gutters, 110px bottom pad)
- `ap-card` / `ap-card-interactive` — card chrome + hover lift
- `ap-control` / `ap-label` — inputs, selects, labels
- `ap-btn-base` / `ap-btn-primary` / `ap-btn-secondary` / `ap-btn-disabled`
- `ap-tabbar` / `ap-tab` — the **single tab language**: segmented pill bar,
  indigo-tinted active pill
- `ap-badge` — status pill

Imported by `custom.scss` between `theme-tokens` and `admin-theme`.

**Gotcha (Dart Sass):** mixins declared inside an `@layer { … }` block are
scoped to that block and cannot be `@include`d from other files. All mixins in
`_admin-primitives.scss` live **outside** any layer; only the `@include` sites
(in `@layer payload`) are layered. (Verified: same-file inclusion also fails —
see `@include` error at the layer boundary.)

## 3. Migration status

### 3a. Flat-surface pass (2026-09-19)

**Done** — every site listed in section 0. Removed: hero/card/row/header/button
shadows (`--ap-shadow-btn`, `-card`, `-card-lg`, `-card-hover`, `-glow*` tokens
deleted), the hero's radial gradient tints (`--ap-tint-*` deleted), the tinted
card fill (`--ap-card-bg` → `--theme-bg`), the collection-row hover lift, the
block-picker thumbnail lift, the sticky-header shadow, and the mobile
back-button glow. Flattened: `.cs-control-box`, `.cs-tier-item-block`,
`.studio-artist-social-item` (divider rows at all breakpoints),
`.studio-artist-card` (20px → 14px), `.cs-card`/`.cs-single-card` (16px → 14px),
`.pages-group` (12px → 14px). Chrome-less headers: `.studio-artist-hero`,
`.cs-header`, `.commissions-list-header`, `ap-tabbar`. Unified titles via
`--ap-title-page`. Payload bridge: `--style-radius-s/m/l` → 8/10/14px, `.pill`
radius pinned to 8px, inputs/textarea `box-shadow: none` (incl. hover), one
accent ramp for hsl + hex + tints, desktop page tail 48px.

**Verified live (2026-09-19)** — `scripts/verify-admin-invariants.mjs`
(authenticated, same invariants as the e2e spec, no destructive global setup):
**41/41 checks pass** — flat list rows at rest *and* hover (14px, hairline,
no shadow, no lift), custom cards matching the row chrome, chrome-less headers
on all three custom views + the commissions list, one 32px/600 title scale,
`--style-radius-s/m/l` = 8/10/14px, `.pill` 8px, 44px/10px shadow-free
controls, and byte-identical primary buttons (height/radius/gradient, no glow).
Computed-style audit before → after, elements carrying a drop shadow:
studio-profile 8→3, commission-settings 5→3, payouts 6→3, collections
9/7/6/7→3, account 11→3 — the three survivors on every page are the tour's
spotlight ring, tour card and tour button (floating guidance overlay).
Nested card-inside-card elements: studio-profile 6→3, commission-settings 4→1
(remaining: the two dashed upload drop-zones and the tour's tips box).
Screenshots: `.ui-review/shots/` (before) vs `.ui-review/shots-after/` (after).

### 3b. Unification pass (2026-09-18)

**Done** — global tokenization pass over `_admin-theme.scss` and
`_blocks-ui.scss`: accent hsl/hsla/hex/rgba values, status colors, muted text,
disabled grays, radius literals (incl. multi-value), shadow recipes, focus
rings and transitions all replaced with `--ap-*` references; dead
`var(--theme-*, #fallback)` fallbacks removed (theme tokens are always
defined, so the fallbacks never applied). Tab unification: `.cs-tab-bar` /
`.cs-nav-tab` and `.studio-artist-tabs` / `.studio-artist-tab-btn` now
`@include ap-tabbar`/`ap-tab`.

**Done (structural convergence)** — bespoke view sections now build on the
recipe mixins: `.studio-artist-*` container/card/input/textarea/select/buttons/pills,
`.cs-*` containers/cards/inputs/touch-controls/dashed buttons,
`.commissions-list-header` (new recipe classes replacing inline styles).
Payload built-ins converge through a new "Payload Edit-View Controls" section:
all `input[type=text/email/password/number/url/search]`, `select`, `textarea`
and react-select `.rs__control`s get 44px height, input-bg surface, 10px
radius and the accent focus ring (with exceptions for `.toolbar-input` and
live preview toolbar controls, which retain their original `var(--base)`
compact height). Payload `.card` radius bumped to the shared
14px; collection-list rows already matched (14px + accent hover).

**Done (Tailwind purge)** — the admin bundle never loaded `globals.css`, so
Tailwind utilities in admin components were dead. Replaced with recipe/utility
classes (`ap-spin`, `ap-icon-sm/xs`, `ap-text-muted/accent/success`, `ap-link`);
`hidden sm:inline` subtitle hiding moved into SCSS; the fully Tailwind-built
`CommissionListHeader.tsx` (unreferenced dead code) was deleted and the live
`CommissionsListHeader.tsx` rebuilt on semantic classes.

**Deliberate visual changes** (Option A unification):
1. Commission Settings tabs switched from underline style to segmented pills;
   Studio/Artist active tab switched from neutral raised pill to the shared
   indigo-tinted active pill.
2. Checkbox `accent-color` in commission-settings toggles is now indigo
   (`--ap-accent-hex-500`) — previously Payload's default primary leaked
   through because `--theme-primary-500` was never themed.
3. Tier inputs and CS touch controls converged from 8px to the shared 10px
   control radius; Payload edit-view inputs grew to the 44px/10px control.
4. Input/textarea focus now uses the shared ring (`--ap-focus-ring`, a12) and
   `--ap-accent-a50` border instead of per-view one-offs.

**Remaining** — drawers/collapsibles/table-card internals still hand-written
(though token-driven); `tests/e2e/admin-theme-unification.e2e.spec.ts` (rewritten
for the flat-surface contract) still needs a run, but note its global setup
POSTs `/api/seed`, which performs a destructive DB reset — use
`scripts/verify-admin-invariants.mjs` for a non-destructive equivalent pass.

## 4. Verification

### One list grammar for the shop screens (2026-10-08)

Products and Product Types are two screens of the same shop section, and they
read as two different products: Product Types is the admin's table view, while
Products rendered through Payload's default list — which this theme turns into
**cards** (see the list-table conversion above) — with a different heading
(`1.65rem/800` in a 1200px centred wrapper vs `--ap-title-page` in the page
gutter).

Now both render through one shell:

- `src/components/admin/AdminListPage.tsx` — heading (`--ap-title-page`, weight
  600, the same token `.list-header__title` uses), one page container
  (`@include ap-page-gutter`, the Studio & Artist container), one search row,
  then the caller's table. Chrome only; data and actions stay in the view.
- `src/components/admin/ProductsListView.tsx` — the shop's products as a table
  (Product / Type / Price (cents) / Updated / Actions) over `AdminTable` and the
  shared `.pt-*` row kit, with search, a Create Product action, delete via
  `deleteProduct`, and an honest “first 100 rows” note (the studio's shop is
  small; the default card lists remain the place for filters and bulk actions).
- `src/components/admin/ProductTypesListView.tsx` was refactored onto the same
  shell, so the two cannot drift again; the CSS block that used to be
  `.product-types-list-view*` is now `.ap-list-view*`.
- The shop tour's step pointed at Payload's `.list-controls`; its `targetSelector`
  now matches `.ap-list-view__top` first, so the guide still highlights the
  products screen.

Measured live (1440px): both screens report one heading each —
`Products` and `Product Types`, 31.85px / 600 at the same left (331px) and top
(96px) — and no `.list-header__title` remains on either.

Pins: `tests/int/products-list-view.int.spec.ts` (shell, columns, type labels,
search by title *and* type, delete, empty state), the refactored
`product-types-list-view` spec, `tests/e2e/admin-list-view-grammar.e2e.spec.ts`
(chrome parity between the two screens in a real browser; products render a
table or the empty state, never the card list), and the updated title-scale case
in `admin-theme-unification.e2e.spec.ts` — which now resolves `--ap-title-page`
in px at runtime instead of comparing against a hardcoded `32px` (the token is
`2.45rem`, i.e. 31.85px at the 13px root, so that constant had gone stale).

### Clipping containers reserve room for focus rings (2026-10-08)

The panel keeps itself from scrolling sideways with `overflow-x: clip` on the
field containers. That clip also cut the **3px focus ring** off wherever a field
sits flush against the edge — most visibly on the **sidebar fields**, which have
no slack at all, so the ring's left band simply vanished (reported as "the left
edge near the hovered input is cut, also for the sidebar elements"). The main
column only looked intact because Payload's rows hand their first child 5px of
accidental room.

Fix: the same clip boxes get `padding-inline: 4px` with `margin-inline: -4px`,
so the clip region grows while every child stays exactly where it was (measured:
field lefts unchanged at 306.8px / 968.1px, desktop and phone, no horizontal
overflow). Two details are load-bearing:

- **`:not(.row__fields)`** — Payload's row container carries `render-fields`
  too, and it aligns its fields with a −5px margin that the compensation would
  flatten (the row's fields slid 5px right in the first attempt).
- the gutter keeps its `padding-inline: 0` from the flat-surface pass, so that
  one declaration has to force its padding through — **the ratchet's count grows
  by exactly one** (1531 → 1532; the budget itself is red from
  unrelated work and needs re-baselining, see §5).

Pins: the ratchet spec now requires the compensation (and the row exclusion) to
stay in the sheet — pixels do not exist in jsdom — and
`tests/e2e/admin-focus-ring-room.e2e.spec.ts` asserts in a real browser that the
ring's 3px band fits inside every clipping ancestor for both a main-column and a
sidebar field, with no horizontal overflow. It passes against the dev server, as
does the pixel probe that found the bug (ring pixels left of the control: 0
before, 84 after).

- `node_modules/.bin/sass` compile of `custom.scss` — clean.
- Dev server (Next + sass-loader pipeline) hot-compiles the final state clean.
- All `--ap-*` vars referenced in compiled output are defined (script check).
- `npx tsc --noEmit` — clean.
- Int tests: `blocks-ui` + `commission-settings-mobile` + the
  `studio-artist-profile` SCSS contract test pass (assertions updated to the
  token forms). Full-suite failures observed are environmental (Redis
  timeouts) or from pre-existing in-flight commission refactors, not this
  change.
- New e2e guard: `tests/e2e/admin-theme-unification.e2e.spec.ts` — rewritten
  for the flat-surface contract: flat rows (no shadow/lift at rest *and* on
  hover), custom cards matching the row chrome, chrome-less page headers, one
  page-title scale, Payload's geometry bridge, ap-control convergence, and
  shadow-free primary buttons. Its invariants were executed live and pass
  41/41 via `scripts/verify-admin-invariants.mjs`; the Playwright wrapper itself
  is unrun because its global setup reseeds the database.

## 5. Slimming strategy

`_admin-theme.scss` is ~4.5k lines with 1,138 `!important`s. That is a
maintenance cost, not a performance one (admin-only bundle, ~120KB, cached).
Three moves, in order of value:

1. **Ratchet guard (done)** — `tests/int/admin-theme-contract.int.spec.ts`
   makes regrowth structurally impossible: no color literals, no radius
   literals outside `_theme-tokens.scss`, and an `!important` budget
   (currently 1,138) that can only be lowered. The entire override sheet is
   now literal-free — every value flows through the token contract.
2. **Dead-rule audit (next, when the browser is reachable)** — a source-only
   scan can't prove a rule dead (Payload builds some class names at runtime
   and splits code across three packages), so the reliable pass is a live-DOM
   crawl: visit every admin view, collect the rendered class set, and delete
   rules whose selectors never match. Expected payoff: a few hundred lines of
   legacy-era selectors (`cell-*`, old `doc-tabs`/`nav-tag` shapes,
   pre-refactor settings classes).
3. **`!important` diet (one dedicated session, with visual verification)** —
   restructure `custom.scss` to `@layer payload-default, payload, ap-admin` and
   move the override sheet into the `ap-admin` layer; then non-important rules
   beat Payload's layered rules by layer order alone, and only the
   emotion-injected (unlayered) conflicts genuinely need `!important`.
   Realistic cut: 50–70% of the 1,138. Do it per-section, with the e2e parity
   spec as the safety net.

Explicitly rejected: rewriting the hacks onto Payload's official theming APIs
(only a minority have official equivalents — buttons/tables/drawers don't) and
accepting the size as-is without the ratchet (the ratchet makes the remaining
size a bounded, monotonically shrinking problem).
