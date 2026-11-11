---
status: plan
collections: [commission-settings, media, pages, queue, studios]
features: [studios]
reviewed: true
---

# Pricing Sections as Page Blocks — Proposal & Analysis

**Status:** PROPOSAL — analysis + recommended design. Nothing implemented yet (2026-10-13).

**The ask (as stated):** move the prices-page layouts into a Payload block — one block that defines
sections, like the prices page does today — so (a) the same sections can be reused on other pages,
and (b) editing them inside the Pages admin inherits Payload's built-in live preview. Possibly
combined with Commission Settings.

This document grounds each claim in the current code, compares the three viable data models, and
recommends one with a phased rollout.

---

## 1. What exists today (ground truth)

**The prices page is a route, not a block-driven page.**
`src/app/(frontend)/(studio)/prices/page.tsx` hardcodes `<PriceList pricing={data.pricing} />` above
the page's custom blocks, and **filters out any legacy `priceList` block** to avoid double rendering:

```ts
const customBlocks = (page.blocks || []).filter((b: any) => b?.blockType !== 'priceList')
```

**Presentation lives in Commission Settings, not in the page.**
`commissionTypes[].sections[]` carries `layout`, `heading`, `subtitle`, `pricesOrder`, `images`
(`src/collections/CommissionSettings.ts`); `fetchPricing` (`src/features/studios/cache/studios.cache.ts`)
shapes them into `pricingSections` for `PriceList`. The original decision was deliberate: *"settings
the artist edits directly, with no hidden block config behind them."*

**The charging pipeline reads the same source.**
- Commission form block + route: `pricing` (tiers), `pricingSections`, `commissionTypes` — sections carry
  the behavioral `selectable` flag the form needs.
- AI estimator: `getPricingContext` → `pricingTiers` (the `{ sectionKey: [{name, price}] }` dictionary).
- `buildInitialAiAnalysis`, price lookup by tier name — all read `pricingTiers`.
- Anything price-related (percentages, the new note fix) lives in `pricingTiers`.

**Pages already have the infrastructure the proposal wants:**
- Blocks field with 10 first-class blocks + a custom block-picker admin (`PagesBlocksField`).
- `livePreview` config with breakpoints; URL = `<studio>/<slug>?preview=true&<signed token>`.
- **A live-preview streaming bridge already runs on the frontend:** `BlocksRenderer` calls
  `useBlocksLivePreview` (`src/components/live-preview/`), built on Payload's official
  `useLivePreview` client + postMessage fallbacks, with media scanning and a block-isolation pill.
  Block field edits stream into the admin's preview pane **while typing** (caveat: media swaps and
  server-resolved artwork can lag until save — the "hero artwork caveat" in `BlocksRenderer`).
- System-page protections: the prices page cannot be deleted (`preventSystemPageDeletion`), its slug
  and `pageType` are locked by access control.
- A `blockType → data need` table (`src/lib/blocks/block-data-needs.ts`) — a new block must classify
  itself (`'pricing'` is an existing need), enforced by a classification test.
- No Payload drafts anywhere (`versions` unconfigured — stated in `revalidatePagesCache`). The live
  streaming above is *form-state* streaming from the admin, which does not require drafts.

**The legacy block is half-removed, not gone — and most of its wiring survived.**
`src/blocks/PriceList.ts` (title-only stub) exists but is not in the Pages block list; `BlocksRenderer`
still has `case 'priceList': return <PriceList pricing={pricing} />`; `BLOCK_DATA_NEEDS` already carries
`priceList: ['pricing']` with classification-test coverage in `block-data-needs.int.spec.ts`; and the
prices route filters the block out. This is residue of the pre-Commission-Settings design — the
proposal is essentially finishing that idea properly. Missing pieces: real fields on the block, the
Pages block-list entry, route preference (remove the filter), and the migration.

---

## 2. Decomposing the proposal — what is actually being claimed

| Claim | Verdict |
| --- | --- |
| Layouts can become a reusable block | ✅ Cheap: block def (stub), renderer case, and the `['pricing']` data-need row **already exist**. Missing: real fields, Pages block-list entry, route preference. |
| Usable on other pages | ✅ Real win, near-free: a block on any page gets `pricing` via the existing needs table. |
| Payload live preview "for free" | ✅ Mostly true: the pages preview bridge streams block fields on keystroke today. Media caveat applies. |
| "Combined with Commission Settings" | ⚠️ This is the real design decision — see §3. |

---

## 3. The three data models

### Model A — the block owns everything (sections **and** tier items)

Sections become blocks with their own `{ name, price, description, sampleImages }` rows.

- ✅ Full live preview including prices themselves; one editing surface.
- ❌ **Duplicates the charging source of truth.** The form, AI estimator, and any quote math need
  `pricingTiers` keyed by section. If that lives in a page document, every consumer must read the
  prices page doc — and a second block instance on a landing page would hold a *copy* of the items
  that will drift from the first.
- ❌ `selectable` (behavioral, used by the form) belongs with sections, so items-in-blocks would
  split one concept across two documents — worse coupling than today.
- ❌ Migration + a much bigger rewrite of `fetchPricing`, `getPricingContext`, the form contract.

**Verdict: not recommended** as the primary model. It buys preview of number fields at the price of
coupling the charging pipeline to a content document.

### Model B — the block is **presentation**; Commission Settings stays the pricing source (recommended)

One `priceList` block with a `sections` array. Each section references a commission section by key and
owns only presentation: `categoryKey`, `layout` (the existing 5), `heading`, `note`, `gallery`,
order = array order (native drag), plus a `visible` toggle if wanted.

- ✅ Zero duplication: prices (`pricingTiers`) stay in one place; presentation moves to the page
  where it conceptually belongs, and where it previews.
- ✅ The five layout components (+ the just-fixed `subtitle` support) are reused as-is — the block
  becomes a second consumer of the same contract.
- ✅ Works on any page with an existing `['pricing']` data need; per-page presentation is a feature
  (a landing embed can have its own heading/note for the same category).
- ✅ The charging pipeline (form, AI, quotes) is untouched.
- ⚠️ Two editors: tiers in Commission Settings; look in the page. That split already exists
  conceptually — this just moves the "look" half next to the page it belongs to.
- ⚠️ Partially reverses the earlier "no hidden block config" decision — mitigated by a legacy
  fallback (below) so no existing studio's prices page can break.

### Model C — status quo

No preview for price presentation; layouts locked to one page. Fine, but the proposal's wins are real
and cheap under Model B.

---

## 4. Recommended design (Model B), concretely

**Block** — resurrect the `priceList` slug (renderer case + block thumbnail already exist):

```
priceList (block)
└── sections (array, drag-order = render order)
    ├── categoryKey   select — options from the studio's commission sections
    │                 (custom field component fetches them, precedent: theme/tour pickers)
    ├── layout        select — Big cards / List with gallery / Color panel / Compact grid / Simple list
    ├── heading       text   — defaults to the section label
    ├── note          text   — the "Prices page note" (subtitle)
    └── gallery       media (many) — the category gallery images
```

**Route** — `/prices` renders the page's `priceList` block when present (remove the filter), else
falls back to today's auto-rendered sections. The fallback is what makes this a zero-risk rollout:
no studio's prices page changes until its block exists.

**Renderer** — `case 'priceList'` shapes block sections + `pricing.pricingTiers` into the existing
`PriceList` contract. The layouts and section resolution do not change.

**Data needs** — already present: `priceList: ['pricing']` and its classification-test coverage. No new row to add.

**Commission Settings** — keeps tiers, descriptions, sample images, `selectable`, AI rules, queue and
checklist. The per-category *appearance* disclosure (`.cs-appearance-summary`: Category style,
Heading, Note, Position, gallery) is migrated to the block and then removed (or shown as a pointer at
the prices page). `pricesOrder` becomes block array order.

**Migration** — one pass per studio: read `pricingSections` from `fetchPricing` and write the
equivalent `priceList` block onto the prices page doc (ronya first; the migration script already owns
this kind of step). Fixtures/seeds: after the commission-settings fields are removed, `createDefaultPages`
seeds the block; before that, the fallback covers them.

**Tour & guide copy** — `src/lib/tour/curriculum.ts` already says *"Your /prices page stays empty
until the first tier exists"* and *"Your tiers become the price sheet on /prices automatically"*;
those stay true. Add one line/step for the new home of the look ("shape how prices appear in the
Pages editor — with live preview"). `src/lib/system-pages-guide.ts` copy gets the same pointer.

**Specs** — `commission-settings-mobile.int.spec.ts` pins the appearance disclosure (would move or
retire); new pins: block def fields, route preference + fallback, renderer wiring, data need.
`price-layout-subtitles.int.spec.ts` keeps passing untouched (layout contract unchanged).

---

## 5. Live preview reality check (what "for free" means)

What you get with the block on the prices page:

- The Pages admin's preview pane (`?preview=true&token`, mobile/tablet/desktop breakpoints) pointed at
  `/prices`, rendering the page's block.
- **Keystroke-level streaming** of block fields through the existing `useBlocksLivePreview` bridge —
  layout, heading, note, gallery ordering update live.
- Caveat (existing, documented in `BlocksRenderer`): media *swaps* can lag until save; text/layout
  do not.
- No Payload drafts involved — so nothing new to enable, nothing to migrate on the versions front.

What you do **not** get:

- Live preview of tier **prices/items** — they stay in Commission Settings (Model A territory).
  That is the deliberate trade: number fields don't need a visual preview; layouts do.
- Also note the theme editor's bespoke cookie/postMessage draft mechanism is a different system;
  this proposal does not need it.

---

## 6. Phased plan

- **P0 — cleanup (tiny).** Delete the dead `PriceList` stub or mark it for resurrection; remove the
  stale filter comment path once the block is real.
- **P1 — block + route (the core).** Real fields on `priceList`; route prefers the block, keeps the
  legacy fallback; renderer shapes block → `PriceList`; data-need row; block-editor options field.
  Migration script for ronya → block instance. Tests: route preference/fallback, renderer, contract.
- **P2 — flip the editors.** Commission Settings drops the appearance disclosure (fields removed
  after migration); seeds write the block; tour/guide copy updated; appearance pins retired/moved.
- **P3 — optional.** More page types may host it (already possible), section `visible` toggles,
  per-section anchor links, and — only if ever needed — enabling Pages drafts for true
  server-rendered draft previews (separate project; current streaming already covers the use case).

---

## 7. Risks & open questions

- **Two editors during transition** (P1): one-way migration + fallback + a pointer line in Commission
  Settings ("the look of your prices page now lives on the page itself").
- **`categoryKey` select source**: options need the studio's sections in the block editor — custom
  field component; validate on save against existing sections and degrade gracefully if a section is
  later renamed/deleted (fall back to legacy rendering for that section, like `resolveTierItems`
  already does for unknown keys).
- **Guided tour / system-pages guide copy** must land in the same PR as P2, or the "Starter"/
  "Action Needed" guidance points at a field that no longer exists.
- **Block slug**: reuse `priceList` (renderer case and thumbnail exist) vs a fresh name —
  recommendation: reuse.
- **Media caveat** for gallery swaps — accepted; documented behavior, not new.

## 8. Recommendation

Adopt **Model B**, phased as above. It delivers both stated wins at the lowest risk: the five layouts
become reusable sections on any page, and editing them happens inside the Pages admin with the
existing live-preview streaming — while the charging pipeline (commission form, AI pricing, quotes)
keeps one stable, document-independent source of truth. The prices page keeps rendering exactly as
today until a studio opts into the block, so nothing breaks while the migration runs.

The one honest cost: the artist edits prices and look in two places — but that is already true today
(tiers and appearance both live in Commission Settings, one screen away from the page they belong to);
the proposal makes each place match the concern it owns.
