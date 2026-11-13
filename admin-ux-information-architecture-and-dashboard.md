---
status: shipped
collections: [announcements, artists, commissions, customers, media, orders, pages, products, push-subscriptions, queue, sprints, studios, themes, tips]
features: [admin-dashboard, admin-navigation, studio-settings, studios]
---

# Admin UX: Information Architecture Alignment & Artist-Friendly Dashboard

How the Payload admin panel (`/admin`) was re-architected from a developer-oriented
collection dump into an artist-first studio control panel: one shared information
architecture that the sidebar and dashboard both mirror, plain-language entity names,
and a custom role-aware dashboard with quick actions and edit-first cards.

## Motivation

The admin panel serves non-technical artists. The original UX failed them in
three ways:

1. **The sidebar and dashboard told different stories** — the same entity appeared in
   different groups with different labels depending on which half of the screen you
   looked at (e.g. `Commission Settings` lived under the sidebar's "🏢 Studio Setup"
   but under the dashboard's "Studio Management"; `Pages` was "Studio Setup > Pages"
   in the sidebar but "🌐 Content & Media" on the dashboard).
2. **Developer language** — "Sprints" (Scrum jargon), "Raw Media", "Digital Assets",
   "Notification Settings", GUID row titles and a "Search by Guid" placeholder.
3. **Meaningless actions** — the dashboard offered "Create new Studio / Theme /
   Commission Settings / Notifications / Artist Profile" even though those collections
   are 1:1 per tenant: an artist already has one, and clicking create errors or
   produces an orphan row.

## Architecture

```
                       ┌───────────────────────────────────────────┐
                       │   src/lib/utils/admin-ia.ts (single source of   │
                       │   truth: ADMIN_GROUPS + ADMIN_LABELS)     │
                       └──────┬──────────────┬─────────────┬───────┘
                              │              │             │
              collection/     │    custom     │   breadcrumbs /
              global admin    │    nav        │   step-nav /
              configs         │    labels     │   view headings
                              ▼              ▼             ▼
              ┌─────────────────────┐  ┌─────────────┐  ┌───────────────┐
              │ sidebar (nav)       │  │ dashboard   │  │ AdminHeader-  │
              │ StudioAwareNav +    │  │ cards       │  │ Actions /     │
              │ DirectStudioLinks   │  │ (from       │  │ SetStepNav    │
              │ + groupNavItems     │  │ admin.group)│  │               │
              └─────────────────────┘  └─────────────┘  └───────────────┘
                              │              │
                              └──────────────┴── mirror by construction ──┘

        /admin (dashboard route, templateType 'default' — RootPage wraps
        the view in DefaultTemplate, so the custom view renders content only):

        ┌───────────────────────────────────────────────────────────────┐
        │ DashboardViewServer (views.dashboard, importMap entry)        │
        │   ├─ DashboardHero        welcome + active studio + live link │
        │   │                        + open-commissions count            │
        │   ├─ DashboardQuickActions role-aware plain-language tiles     │
        │   └─ CollectionGrid       getNavGroups() → same groups/order/ │
        │                            labels as the sidebar; "Edit" cards │
        │                            for 1:1 collections                 │
        └───────────────────────────────────────────────────────────────┘
```

## Key Components

### 1. Shared IA module (`src/lib/utils/admin-ia.ts`)

- `ADMIN_GROUPS`: My Studio, My Work, Shop & Sales, Media Library, System & Security,
  Legal & Policies — plain text, no emoji zoo.
- `ADMIN_LABELS`: artist-facing names — `Artist Profile`, `Notifications`, `Payouts`,
  `Integrations`, `Work Batch(es)`, `Download(s)`, `Original File(s)`.
- Every collection/global `admin.group`, every hardcoded nav label, breadcrumb, and
  step-nav entry references these constants — renaming a label changes it everywhere.

### 2. Taxonomy applied across all 18 collections + 2 globals

| Group | Entities |
|---|---|
| My Studio | Studios, Artists, Themes, CommissionSettings, NotificationSettings, Pages (+ Payouts and Integrations custom views) |
| My Work | Commissions, Queue (admin-only), Sprints → **Work Batches** |
| Shop & Sales | Products, Orders, Tips, DigitalAssets → **Downloads** |
| Media Library | Media, RawMedia → **Original Files**, Announcements |
| System & Security | Users, Customers |
| Legal & Policies | Terms of Service, Privacy Policy |

`payload.config.ts` collection order was rearranged to match this group order, so the
dashboard card list (config order) and the sidebar group list (insertion order in
`groupNavItems`) are identical by construction.

### 3. Custom dashboard (`src/components/admin/DashboardViewServer.tsx`)

Wired via `admin.components.views.dashboard` in `payload.config.ts`. The dashboard
route runs with `templateType: 'default'`, so RootPage already provides
`DefaultTemplate` — the view component renders content only (unlike custom-path
views such as Payouts, which wrap their own template).

- **Hero** (`DashboardHero`): "Welcome back, {name}", active studio (resolved from the
  `payload-tenant-context` cookie), "View your live site" link (URL rules mirror
  `getDynamicLivePreviewUrl`), and an open-commissions stat.
- **Quick actions** (`DashboardQuickActions`): plain-language tiles with Phosphor
  icons — Review Commissions, Set Prices & Slots, Payouts, Upload Artwork, Edit Your
  Site; admins additionally get Users & Access and Customers.
- **Card grid** (`CollectionGrid`): built from `getNavGroups(permissions,
  visibleEntities, config, i18n)` — the same grouping function the sidebar's
  `DefaultNavClient` uses — so groups, order, and labels always mirror. 1:1 tenant
  collections render a single **Edit** card (direct doc href from
  `getDirectNavLinks`), never "Create new". The Payouts and Integrations cards are
  appended to the My Studio group to match the sidebar's static links. Custom views
  that are not collections must stay in `MY_STUDIO_CUSTOM_CARDS`, or the e2e
  invariant "dashboard card titles mirror sidebar link labels" fails.
- **Icons**: `@phosphor-icons/react` uses React context with no `'use client'`
  directive — it cannot render inside RSC. Icons are isolated in a client leaf
  (`QuickActionIcon`, `ExternalLinkIcon`) and passed by name; the server components
  stay server-only.

### 4. Supporting changes

- `resolveActiveTenantId` extracted and exported from
  `get-direct-nav-links.action.ts` (must be `async` — every export of a `'use server'`
  module has to be an async function).
- `getDashboardData` (feature action, `overrideAccess: false` + `req.user`) resolves
  the active studio and the open-commissions count (status not in
  `['completed', 'rejected']`).
- Latent bug fixed: the My Studio nav group's open/close preference was written under
  label "🏢 Studio Setup" but read under "Studio Management" — the collapse state
  never persisted. Both sides now use `ADMIN_GROUPS.MY_STUDIO`.
- Breadcrumbs / step-nav in `AdminHeaderActions`, `CommissionSettingsAdminView`, and
  `PayoutsView` use the shared labels ("My Studio / Payouts").
- **Branded header persistence fix (2026-09-19).** `CustomAdminHeader` used to
  render `null` when it detected it had been hidden, because Payload/Next keeps
  the previous view's header mounted during soft navigation and React's Offscreen
  bookkeeping hides it with an inline `display: none !important`. But "hidden"
  is also true of the LIVE header mid-transition, that state was sticky, and
  unmounting destroyed React's ability to un-hide the node — so going back to
  Studio & Artist or Commission Settings could leave the page with **no header at
  all** until a hard reload (reproducible via browser back/forward, and
  intermittent enough to look random).

  The header element now stays mounted unconditionally (a hidden slot renders a
  bare element with zero children, so it still cannot mount a second tour runner
  or a duplicate portaled card), the watcher is attached through a callback ref
  (a mount-time effect missed the element whenever `user` arrived late and never
  re-ran), the route is re-checked on every `usePathname()` change, and a
  self-heal clears a stuck inline hide **only** when no sibling header is visible
  and the instance is the one that rendered the route currently in the address
  bar.

  Verified with `scripts/probe-admin-header-clean.mjs` (clean, uninstrumented):
  header visible after every step (longest gap 750 ms = navigation latency) across
  sidebar hops, back/forward, the tour's Next Step and breakpoint resizes —
  against the previous component, where browser-back left it invisible for the
  entire 6 s sample window and never recovered. A permanent guard lives in
  `tests/e2e/admin-header-persistence.e2e.spec.ts`.

## Files Changed

Phase 1 (IA alignment):

- `src/lib/utils/admin-ia.ts` — new shared IA constants
- `src/collections/*.ts` (18 files) + `src/globals/{TermsOfService,PrivacyPolicy}.ts`
  — groups and labels from shared constants; Sprints/DigitalAssets/RawMedia/
  NotificationSettings renamed
- `src/payload.config.ts` — collection order mirrors sidebar groups
- `src/components/admin/{StudioAwareNav,DirectStudioLinks,AdminHeaderActions,PagesNavSubList,CommissionSettingsAdminView,PayoutsViewServer}.tsx`
  — shared labels, preference-key fix, comment updates
- `src/features/admin-navigation/actions/get-direct-nav-links.action.ts` — shared
  labels, exported `resolveActiveTenantId`
- `src/features/studio-settings/components/PayoutsView.tsx` — labels + heading
- `src/payload-types.ts` — regenerated (order-only change)

Phase 2 (custom dashboard):

- `src/features/admin-dashboard/` — new feature module
  - `actions/get-dashboard-data.action.ts` — studio + open-commissions data
  - `components/DashboardHero.tsx` — hero + quick actions (server)
  - `components/QuickActionIcon.tsx` — client icon leaf (Phosphor)
  - `components/CollectionGrid.tsx` — mirroring card grid
  - `styles.ts` — theme-var style tokens
  - `index.ts` — barrel
- `src/components/admin/DashboardViewServer.tsx` — dashboard view wrapper
- `src/payload.config.ts` — `views.dashboard` registration
- `src/app/(payload)/admin/importMap.js` — regenerated via `generate:importmap`

Phase 3 (nav cleanup):

- `src/features/admin-navigation/components/NavItemIcon.tsx` — client Phosphor
  icon map (slug → icon) with fallback glyph
- `src/features/admin-navigation/components/IconNavLinks.tsx` — icon-aware
  replacement for Payload's `DefaultNavClient` (same NavGroup/preferences/link
  styling, icon per link)
- `src/components/admin/StudioAwareNav.tsx` — uses `IconNavLinks`
- `src/components/admin/DirectStudioLinks.tsx` — icons on My Studio links
- `src/components/admin/PagesNavSubList.tsx` — tenant-scoped fetch
  (`where[tenant][equals]` from context cookie / first owned tenant) and
  logical ordering (home first → headerOrder → title)
- `src/components/TenantSelector.tsx` — renders nothing for single-studio
  artists (dead space removed), writes the tenant-context cookie on first
  load (deterministic active studio), plain-language switcher copy
- `src/app/(payload)/styles/_admin-theme.scss` — `.nav__link-icon` styling

Phase 4 (collection polish):

- `src/components/admin/StatusPillCell.tsx` — generic status pill for list
  cells (labels read from field options, value-keyed colors in the admin
  theme, light + dark variants); wired on Commissions, Orders, Sprints, Tips
- `src/collections/Commissions.ts` — `useAsTitle` → `characterName` (label
  "Character name", so search says "Search by Character name"); GUID column
  dropped from list defaults; guid + Stripe Payment Info hidden from artists
  (admin-only `condition`); friendly collection description
- `src/app/(payload)/styles/_admin-theme.scss` — `.status-pill--*` palette

Phase 5 (visual pass + regression suite + infra repairs):

- `src/features/admin-dashboard/components/CollectionGrid.tsx` — card icons
  (reuses `NavItemIcon`), stable `admin-dashboard-*` classes for tests,
  Payload `Link` for the Payouts card
- `src/features/admin-dashboard/components/DashboardHero.tsx` — stable classes
- `src/components/admin/ListEmptyState.tsx` — data-aware friendly empty state
  via `useListQuery` (rendered through `beforeListTable`); wired on
  Commissions with per-collection copy
- `src/app/(payload)/styles/_admin-theme.scss` — hover/focus-visible states, icon chips,
  empty-state styling
- `tests/e2e/admin-ux.e2e.spec.ts` — 5 regression tests: hero + role-aware
  quick actions, icons on every nav link, sidebar groups == dashboard groups
  (labels + order), card titles mirror sidebar links per group, 1:1
  collections offer Edit and never Create new
- `src/collections/Studios.ts` — added missing 'Twitch' platform option
  (pre-existing seed bug: the seed seeds a Twitch link but the option list
  lacked it, breaking `/api/seed` with a 500)
- `src/lib/loaders/css-loader.mjs` — bare package CSS imports now delegate to
  Node resolution (previously mangled, breaking the e2e suite under Node 24)
- `package.json` — `test:e2e*` scripts include the repo's `register-css`
  loader (without it the suite fails loading specs on CSS imports)

Responsive sidebar fix (reported after Phase 5):

- `src/features/admin-navigation/components/ResponsiveNavPortal.tsx` —
  portaled sidebar for widths <= 1440px. Payload's `<aside class="nav">` is
  hit-test-dead in Chromium at its `l` breakpoint (`max-width: 1440px`): the
  sidebar renders but never receives pointer events (closed sidebar AND
  <769px modal alike). The component re-renders the identical nav content
  portaled to `<body>` — outside the template's stacking context — as a
  persistent fixed sidebar at 769–1440px and as a modal at <769px (driven by
  Payload's own nav state via `useNav`, closed by default, opened by the
  app-header hamburger).
- `src/app/(payload)/styles/_admin-theme.scss` — hides the native aside at <=1440px,
  forces the 275px grid column + `grid-column: 2` on the wrap at 769–1440px
  (no content/sidebar overlap), hides the template toggler in that range
- `src/components/admin/StudioAwareNav.tsx` — renders nav content inside
  `ResponsiveNavPortal`

Visual block editor clarity (block picker + preview bridge):

- All 11 blocks (`src/blocks/*.ts`) — `admin.images.thumbnail` (600×400
  screenshots captured from real renders, `public/images/blocks/`),
  `admin.components.Label` → `CustomBlockLabel` on all blocks (row headers
  show the block's title), CommissionForm preview toggles moved into a
  collapsed "Preview controls" group with plain labels
- `src/components/admin/BlockHoverBridge.tsx` — hover/focus on a block row in
  the page editor broadcasts a `scroll` preview event (row index as the
  targeting key)
- `src/lib/theme/livePreviewEventBus.ts` + `src/lib/eventBus/index.ts` — preview
  payload extended with `blockIndex`
- `src/components/BlocksRenderer.tsx` — blocks carry `data-block-index`;
  scroll handler prefers `blockIndex` for exact targeting (works with
  duplicate blocks); fixed pre-existing bug: mediaGallery read `block.items`
  but the field is `images` (gallery never rendered)
- `src/components/admin/CustomBlockLabel.tsx` — added missing
  `sectionedContent` name

Page types & auto-rendered live sections:

- `src/collections/Pages.ts` — new `pageType` select (`standard | prices |
  shop | queue`) with plain-language labels; Prices/Shop/Queue are
  singletons per studio (hook `enforcePageTypeRules` throws a friendly error
  on duplicates); slugs locked for automatic pages; the three dynamic blocks
  (priceList/queueView/shopCatalog) removed from the blocks picker
- `src/collections/hooks/enforcePageTypeRules.ts` — singleton + auto-slug
  enforcement
- Routes `/prices`, `/shop`, `/queue` — always render their live section
  (synthetic block prepended) with the artist's custom blocks below;
  legacy dynamic-block rows in page data are filtered out (no double render)
- `src/lib/blocks/createDefaultPages.ts` + `src/seedGlobals.ts` — default/seed pages
  set `pageType` by slug; prices keeps only FAQ, shop/queue start empty
  (sections are auto-rendered)

System pages (protect, hide, badge):

- `src/collections/Pages.ts` — `isSystem` (platform-managed, admin-only,
  seeds mark the 7 defaults) + `isHidden` (artist-facing, hidden pages
  return 404; Home can never be hidden — enforced in `enforcePageTypeRules`);
  `delete` access extended: system pages (pageType-based OR `isSystem`) are
  undeletable by artists; slug title column uses a custom `PageTitleCell`
- `src/components/admin/PageTitleCell.tsx` — Pages list title renders
  "System" / "Hidden" badges
- `src/components/admin/PagesNavSubList.tsx` — sidebar page sub-list shows the
  same badges and dims hidden pages
- `src/features/studios/actions/page.action.ts` — `getPageBySlug` filters
  `isHidden` (single site-side point → hidden pages 404 everywhere incl.
  /prices /shop /queue; admin still sees them)
- Routes /prices /shop /queue — `notFound()` when the page is hidden/missing
  (bare fallbacks removed)

## Artist-facing visibility: document bookkeeping (2026-09-29)

Artists no longer see when a document was created or last modified, in either place
Payload prints it:

- **List views** — Payload's table renders the header as
  `<th id="heading-createdAt">` and the cells as `<td class="cell-createdAt">`
  (`@payloadcms/ui` elements/Table), so both are targeted by id/class.
- **Document toolbar** — the "Last Modified: …" / "Created: …" items are the only
  `.doc-controls__value-wrap` list items carrying a `.doc-controls__label` paragraph
  (elements/DocumentControls), which is what the selector keys on. No `!important`
  is needed: the rule's specificity (body + attribute + two classes + `:has()`) beats
  Payload's `.doc-controls__list-item { display: flex }`.

One `body[data-is-admin='false']` block in `_admin-theme.scss` covers every
collection, next to the existing API-tab rule. Admins keep the information.

**Why display-only.** The timestamps stay readable through the API on purpose:
`PagesListView`'s "Starter" pill compares `createdAt === updatedAt`, commission
cards print `createdAt`, and shop cards badge new products from it. Denying field
read access (a per-role lever that *does* remove columns —
`filterFieldsWithPermissions` drops fields the user cannot read) would have hidden
two columns by breaking real artist-facing features.

**Known gaps.** The timestamps remain listed in Payload's column picker: its pills
carry no per-field class or attribute, so hiding those entries would mean guessing
by position. Custom list views (e.g. the studio Pages list) render their own markup
and were never showing these columns.

### Whole collections

Plumbing collections hide themselves with `admin.hidden` — the same mechanism that
keeps Users and Chat Rooms out of an artist's panel. `push-subscriptions` joined them
(2026-09-29) with the precise shape `({ user }) => !(user?.collection === 'users' && user?.roles?.includes('admin'))`.
`getVisibleEntities` evaluates that predicate per request, and both admin surfaces
read from it: `StudioAwareNav` filters the sidebar by `visibleEntities.collections`,
and the dashboard builds its cards with `getNavGroups(permissions, visibleEntities, …)`.
The collection's API is untouched (the push actions use `overrideAccess: true` and an
explicit user), and hidden is a UI statement, not a gate: a typed URL still runs
through the collection's access control.

## Verification

- Artist-facing bookkeeping: `tests/int/artist-admin-visibility.int.spec.ts` pins
  both halves — the selectors live in the artist-scoped block (and nowhere else, so
  admins keep them) and the installed `@payloadcms/ui` 3.90.1 still emits the ids,
  cell classes and toolbar classes the rule depends on. Not browser-verified in this
  sandbox (no login); the rule adds no `!important` and no style literals.
- `tsc --noEmit` clean; eslint reports no new issues (pre-existing errors in
  untouched files remain).
- Browser-verified with throwaway users (deleted after): superadmin and artist roles —
  sidebar groups == dashboard groups, same order, same labels; "Edit" cards replace
  "Create new" on 1:1 collections; artist sees no Users/Customers/Queue; live-site
  link and open-commissions count render.
- Phase 3 re-verified in browser: every nav link renders a Phosphor icon; the Pages
  sub-list is scoped to the active tenant (admin sees 6 pages, not ~40 across all
  tenants) and ordered home-first; single-studio artists see no tenant switcher UI.
- Phase 4 re-verified in browser (after restart): commissions list rows are titled
  by character name (Astraea Starweaver, Ignis the Red Dragonborn — no GUIDs), the
  search box says "Search by Character name", status pills render (Awaiting
  Payment / Paid / In Progress), the GUID column is gone, and the commission edit
  view shows guid + Stripe Payment Info for admins only (hidden for artists).
- Phase 5 verified: `npx playwright test tests/e2e/admin-ux.e2e.spec.ts` → 5/5
  pass (run with the repo's `test:e2e` loader flags); the friendly empty state
  renders on an empty commissions list ("No commissions yet — …").
- Responsive fix verified at 1280/1440/1600/1920 (nav click → navigates; no
  content overlap at 769–1440) and at <769px (app-header hamburger opens the
  modal nav; links click through). Breadcrumb home icon navigates to /admin
  from a list page at 1440px. Admin-ux e2e suite still 5/5 after the change.
- Block editor clarity: thumbnails captured from live renders (11 blocks,
  `public/images/blocks/*.png`, 600×400); `data-block-index` + scroll-by-index
  verified against the running dev server; the hover bridge and the admin
  config features (thumbnails in the block drawer, row labels on all blocks,
  collapsed preview controls) take effect after the next dev-server restart.
  NOTE: the e2e suite was NOT re-run for this batch because global setup
  reseeds the dev DB.
- Page types: the DB was dropped and reseeded by the user after this change
  (fresh schema push includes `pageType`); verification of the reseeded state
  (singleton enforcement, picker without dynamic blocks, auto-rendered
  sections on /prices /shop /queue) happened during the user's reseed flow.
- System pages: badges verified visually in the Pages list + sidebar;
  hidden pages 404 on the site; delete protection + home-not-hideable
  enforced via access + beforeValidate hook.
- Screenshot/DOM evidence kept in `scripts/.ux-audit/` (`11-p2-admin-dashboard.png`,
  `12-p2-artist-dashboard.png`, `13-p3-admin.png`, `13-p3-artist.png`, `14-p4-commissions-list.png`,
  `15-p4-commission-edit-artist.png`, `16-p5-empty-state.png`, `p3-admin.txt`,
  `p3-artist.txt`, `p4-list.txt`, `after-dump.txt`, `p2-admin-dump.txt`).

## Debug Reference

- **Admin config changes not appearing**: the Payload instance caches
  `admin.components` — restart the dev server after adding/removing admin views
  (`views.dashboard` etc.). Collection groups/labels hot-reload; admin view wiring
  does not reliably.
- **`createContext is not a function` on /admin**: a client-context package
  (`@phosphor-icons/react`) was rendered inside a Server Component — move icon
  rendering into a `'use client'` leaf and pass icon names as props.
- **`Server Actions must be async functions`**: every export of a `'use server'`
  module must be `async` — including helpers like `resolveActiveTenantId`.
- **Dashboard route template**: `/admin` runs with `templateType: 'default'` — do NOT
  render `DefaultTemplate` inside `views.dashboard` (double sidebar). Custom-path
  views (`/admin/payouts`) DO wrap their own `DefaultTemplate`.
- **Nav preference keys**: `NavGroup` stores open/close state under its rendered
  label — the read side must use the exact same constant.
- **Counts are non-fatal**: `getDashboardData` swallows count failures so the
  dashboard never breaks over a stat.

## Next Steps

Phase 5 completed the planned roadmap. Follow-ups worth considering:

- A custom list view (full replacement) if richer empty states or inline
  guidance per status are ever needed.
- Icons on the remaining default Payload surfaces (e.g. collection edit
  headers) if desired.
- Run the full e2e suite (`yarn test:e2e`) — it now loads under Node 24; the
  admin-ux spec is the new baseline.

## Debug Reference

- **Sidenav stays hidden after leaving the Theme editor (fixed 2026-09-29)**: the
  full-canvas rule hid `.nav` through *two* writers — the server-rendered class
  `:has(.collection-edit--themes)` and `body[data-is-theme-view]`. Next keeps the
  previous route's tree mounted in a hidden Activity, so the stale theme view kept
  the class matching on every later admin page until a hard reload. Fix: the class
  selectors are gated on `html:not([data-admin-hydrated='true'])` (a pre-hydration
  fallback only, still flash-free on a fresh load), while
  `src/lib/theme/admin-canvas-view.ts` owns the attributes — written through an
  isomorphic layout effect so navigating *into* the editor cannot flash the sidenav
  either. If a canvas-view rule ever stops applying, check whether
  `data-admin-hydrated` and `data-is-theme-view` / `data-is-page-view` reached
  `<html>`/`<body>`; if a *global* rule regresses, check whether it keys on a stale
  class instead of an attribute.
- **`condition` function signature**: Payload calls
  `condition(data, siblingData, { user, ... })` — the user object is the
  THIRD argument. `condition: ({ user }) => ...` silently destructures the
  form data instead and the check always fails. Use the full three-parameter
  form, like the tenant field does.
- **List scoping via tenant cookie**: `ownTenantOnlyQuery` scopes admin list
  views to the active tenant from `payload-tenant-context`. "Empty" lists in
  verification are often just the wrong active studio — set the cookie or use
  the switcher.
