---
status: shipped (phases 1–5)
collections: [commissions, pages, product-types, products, studios]
features: [commissions, onboarding, shop, studios]
---

# Studio Onboarding Starter Patterns (SPEC)

> **Status**: SHIPPED — patterns, onboarding step, pattern-aware seeding, the shop
> visibility toggle (`/admin/shop-settings`) and the shop topic tour. The setup tour
> stays commission-only by design (§5.5).
> **Related**: `docs/guided-studio-setup-tour.md` (the tour a pattern must branch),
> `docs/monetization-listing-gates-and-subscriptions.md` (Stripe gate — unchanged by this),
> `docs/digital-shop-product-types-and-library.md` (product-type semantics),
> `docs/multi-round-commissions.md` (what a commission type carries).

## 1. The idea

Onboarding asks an artist for a name, an aesthetic and an email, then generates a
studio. It never asks **what kind of business this is**, so every studio is born
with the same content — which today is commissions-shaped, with shop cards bolted
on. An artist who only takes commissions gets a "Browse Courses" card pointing at
an empty shop.

Two starter patterns, chosen once during onboarding:

1. **Commissions + shop** (`both`) — commissions and a digital shop side by side.
   The shipped default, and what every existing studio is.
2. **Commissions** (`commissions`) — the artist takes commission requests; the
   shop page exists but stays out of the nav until they open it.

**There is deliberately no shop-only pattern.** A storefront with no way to request
work leaves a visitor who arrives from the directory with nothing to do, and every
artist on the platform is here for commissions first. An artist who only wants to
sell files starts on `commissions` and opens the shop from Settings when their
first product exists — the page and the product types are one toggle away.

## 2. Verified current state (facts this plan rests on)

- **A new studio is content-empty.** `buildCommissionSettingsInput`
  (`src/features/onboarding/lib/onboarding-records.ts`) creates
  `commissionTypes: []` and `pricingTiers: {}` — only `defaultMaxSlots: 5` and
  `currency: 'EUR'` are set. The comment there says the artist adds types later
  "in Commission Settings".
- **Pages are seeded from one fixed template.** `createDefaultPagesForTenant`
  (`src/lib/blocks/createDefaultPages.ts`) creates six pages: home, prices, queue,
  shop, privacy-policy, tos. The home page carries a `bentoGrid` whose cards are
  **Live Artist Queue**, **Digital Assets** (`/shop?filter=character_base`) and
  **My Courses** (`/shop?filter=course`). Prices carries a commission FAQ (TAT,
  refunds, backgrounds).
- **No studio-level feature flags exist.** There is no `shopEnabled` /
  `commissionsEnabled` on `Studios`. Shop visibility is currently implicit: the
  shop page exists in nav for every studio, and `ProductTypes.isEnabled` governs
  individual types.
- **Onboarding is 5 steps**: Identity → Aesthetic → Account → Generating → Launch
  (`src/features/onboarding/components/OnboardingView.tsx`). Now 6, with Studio type
  inserted as step 2.
- **The setup tour is a fixed 7-step commission sequence** (hero → pricing tiers →
  prices page → queue checklist → legal → Stripe → publish). Steps 2–4 assume
  commissions exist; there is no shop step.
- **Some blocks self-hide when empty** — `latestProducts`/`featuredProducts`
  (`BlocksRenderer.tsx:143`) and `openRounds` (`:214`). Others render a frame
  regardless (`bentoGrid`, `mediaGallery`, `sectionedContent`). This decides which
  blocks are safe to seed.

## 3. The principle that keeps this small

**A pattern seeds content; it never gates capability.**

- Choosing `commissions` does not disable the shop. It hides the shop page from the
  nav and skips shop-only sections, so the studio reads coherently on day one.
- The artist can open the shop later. That **adds** an entry to the nav and reveals
  the page that was already created; it never deletes content or regenerates pages.
- Consequence: store the choice as `starterPattern` on the studio — an
  initialisation hint consumed once — not as a permission.

This is why the plan can reuse `showInHeader` (already on pages) instead of
introducing routing rules: the system pages already exist for every studio, and a
hidden page is exactly "not part of this business yet, but here when you need it".

## 4. Where the choice lives

**New step 2** in onboarding: `Identity → Pattern → Aesthetic → Account → …`.
Two large cards, one tap, no free text. Inserting a step is friction, so it earns
its place by being the only question that changes what the studio *is*.

**Storage**: `Studios.starterPattern` — `select` of `'commissions' | 'both'`,
default `'both'` so existing studios and any API-created studio behave exactly as
they do today. Hidden from the studio form (`admin.hidden`): it describes how the
studio was born, not a setting an artist should re-edit; opening the shop later is
a Settings toggle.

**Plumbing**: `ArtistOnboardingInput.starterPattern` →
`artist-onboarding.action.ts` passes it to `createDefaultPagesForTenant` and to
`buildCommissionSettingsInput({ pattern })`.

**Product types are not seeded by onboarding.** The Studios `afterChange` hook
(`initializeStudioProductTypes`) already writes `DEFAULT_PRODUCT_TYPES` for every
new studio, so the shop has its categories whether or not its page is visible. An
earlier draft of this feature carried a second list in the onboarding module; it
disagreed with the platform's (it invented a `psd` slug and marked the system
types non-system), which would have given a new studio duplicated categories.

## 5. What each pattern seeds

`●` seeded · `◐` created but hidden from nav · `—` not seeded

| | Commissions | Both (commissions + shop) |
|---|---|---|
| **Nav (in order)** | Home, Prices, Queue | Home, Prices, Queue, Shop |
| Prices page | ● | ● |
| Queue page | ● | ● |
| Shop page | ◐ | ● |
| Commission types | ● 1 type, 3 sections | ● 1 type, 3 sections |
| Pricing tiers | — (structure only) | — (structure only) |
| Product types | — (seeded when the shop opens) | ● 3 enabled |
| Home blocks | hero → open rounds | hero → open rounds → latest products |
| Tour | 7 steps, shop wording removed | current 7 steps |
| Hero copy | "{studio} — commissions open" | "{studio} — commissions and digital art" |

### 5.1 Commission settings (both patterns)

Every studio takes commissions, so both patterns get the same structure. Seed
**structure, never prices**:

```
commissionTypes: [{
  key: 'illustration',
  label: 'Illustration',
  slotMode: 'open',
  sections: [
    { key: 'drawingTier', label: 'Drawing Tier', selectable: true },
    { key: 'backgrounds', label: 'Backgrounds', selectable: true },
    { key: 'extras',      label: 'Extras',      selectable: false },
  ],
}]
pricingTiers: {}   // unchanged: the artist sets their own numbers
defaultMaxSlots: 5
```

The sections are already the platform's default vocabulary
(`DEFAULT_COMMISSION_TYPES`, `src/features/commissions/lib/commission-sections.ts`),
so this seeds nothing new — it just writes down what the storefront already assumes.

**No prices.** Inventing numbers ("€40 sketch") would put a fake price list on a
live storefront the moment the studio publishes. Instead the tour's first task
becomes "set your prices", and `/prices` shows the existing empty state.

### 5.2 Product types (both pattern only)

Three types, `isEnabled: true`, slugs matching what the home blocks and the shop's
filters expect:

| Name | Slug | Why |
|---|---|---|
| Character Base | `character_base` | the default bento already filters on this slug |
| PSD / Layered File | `psd` | the most common digital-art product |
| Course | `course` | also referenced by the default bento (`?filter=course`) |

`isSystem: false` — these are the artist's to rename or disable. Seeding types
matters because the shop filter UI is driven by them: an empty type list gives the
shop no categories at all, so the first product upload has nowhere to go.

**No products.** A shop with three fake products is worse than an empty shop with
working categories; the empty state already explains what to add.

### 5.3 Pages and nav

All six system pages are created for both patterns. What changes is `showInHeader`:

- **Commissions**: `shop` → `showInHeader: false`. Prices and queue stay visible.
- **Both**: all visible.

Opening the shop later is therefore a nav edit (`showInHeader: true` on the shop
page) plus the starter product types — never a page that has to be created
retroactively.

Home blocks, both patterns:

- **Commissions**: hero → `openRounds` → `priceList`. What the studio does, then
  what is bookable, then the prices page's own block. `openRounds` self-hides
  until a round opens; `priceList` self-hides until a tier carries items.
- **Both**: hero → `latestProducts` → `openRounds` → `priceList`. Products lead
  because they are the thing a visitor can act on today; the rounds follow. Each
  section self-hides until it has content.

No `bentoGrid`, `mediaGallery`, `shopCatalog` or `sectionedContent` is seeded in
either: those draw a frame even with no content, so a starter page would ship a
visible hole. The block picker offers all of them once the artist has something to
put in them.

### 5.4 Hero copy

The pattern supplies the *business* sentence. The theme does **not**:
`suggestedHeroTitle` / `suggestedHeroBio` were removed from `ThemeGenerationSchema`,
the local fallback and `AESTHETIC_PRESETS`, because a colour prompt was deciding the
artist's first words (and re-rolling the aesthetic silently rewrote their page).

- **Commissions**: "{studio} — commissions open" / "Custom artwork, made to order…"
- **Both**: "{studio} — commissions and digital art" / "Custom work made to order, and
  digital products you can download today."

Both are neutral placeholders the tour's first step asks the artist to replace.

### 5.5 The setup tour stays commission-only

The 7-step **setup** tour is commission-shaped, and that is now correct for both
patterns — every studio can take commissions. The shop half of a mixed studio is
**not** given a step there, deliberately:

- The setup tour's contract is "everything you need before you can take paid
  commissions". A shop task inside it would follow every commissions-only artist
  through work they do not have.
- A mixed studio already learned its shop exists in onboarding's pattern step, and
  the shop page is in its header.

Instead the shop guidance lives in the **shop topic tour**, which is reached from
the "?" on the screens it describes:

| Step | Route | Intent | What it says |
|---|---|---|---|
| `shop-visibility` | `/admin/shop-settings` | know | The switch only decides whether clients see the shop; nothing is deleted. |
| `shop-products` | `/admin/collections/products` | do | Add the first product: type, price in cents, instant delivery. |

This is smaller than making the setup tour pattern-aware (which would mean
threading facts through the runner, the widget and every progress count, with a
real risk of "step 5 of 8" disagreeing with the steps actually played).

### 5.6 Starting layout from the theme (geometry and visibility only)

The theme generator now also answers **how the hero is arranged**, and that choice
becomes the blocks' starting `layout` at creation time. It is the one place the
theme touches a page, so the boundary is written down here:

| The theme decides | It never decides |
|---|---|
| Where the headline sits (left / centred) | Which words the headline says |
| Whether the intro paragraph exists | What the intro paragraph says |
| The headline's type scale (poster) | Which blocks the page has, or their order |
| How much of the hero the artwork takes | Whether the studio has a shop |
| One call to action or two | Anything about an existing studio |

Three fields were added to `ThemeGenerationSchema` (all optional, all enums, no
numbers) plus the local fallback and the curated presets:

- `heroComposition`: `editorial` (left copy — the default), `centered`,
  `poster` (oversized headline, intro removed, one action), `minimal` (headline
  only — intro and second action removed).
- `heroArtworkEmphasis`: `subtle` / `balanced` / `dominant` (full-bleed, the
  untouched default).
- `heroButtonEmphasis`: `single` / `pair`.

`src/features/onboarding/lib/starter-block-layout.ts` expands those names into
`HeroLayout` entries, passes them through `sanitizeHeroLayout`, and returns the
layout per block type. `starterHomeBlocks(pattern, name, layout)` attaches it. The
default composition expands to **nothing**, so a studio created without a
composition is byte-identical to one created before this feature.

**Visibility is part of layout.** `HeroLayoutEntry.hidden` removes an element
from the block entirely (not `opacity: 0` — it is absent from the DOM, so it takes
no space and screen readers never reach it). Only `true` is stored. The headline
and the background artwork cannot be hidden: the first carries the studio's name,
the second already has `mascotArtwork.showMascot`. A hidden element is restored
from the live preview's **Hidden** list, because a removed element has no box to
select — the per-element toolbar's hide button and that list are the two halves of
the same control.

**Applied once, never re-applied.** `/api/ai/generate-theme` and the admin theme
field keep writing styling only; re-generating an aesthetic on an existing studio
does not touch its pages. Pinned by `tests/int/starter-block-layout.int.spec.ts`
and the existing “theme generation produces styling, not content” cases.

## 6. Shared base (both patterns)

Unchanged from today: artist + user + studio (`profileStatus: 'draft'`), theme from
the aesthetic step, commission settings record, six system pages, legal pages,
Stripe gate and the publish rule.

## 7. Explicitly not seeded

| Not seeded | Why |
|---|---|
| Prices / price tiers | Fake numbers become a live price list |
| Products | A fake catalogue is worse than an honest empty state |
| Commission rounds | Rounds need real dates and a schedule |
| Media / artwork | Only the artist's own work belongs here; placeholders would ship to a public page |
| Stripe state | Publishing remains an outcome of connecting payouts, not of onboarding |
| Portfolio content | Same reason as media |

## 8. Backfill and reversibility

- Existing studios: `starterPattern: 'both'` — behaviour identical to today, no
  content is touched.
- A pattern change after creation is a **nav-visibility edit only**: flip
  `showInHeader` on the relevant system pages, add the starter product types if the
  shop is being turned on and none exist. Never delete pages, products or types.
- Rollback of the feature: drop the field; pages keep whatever `showInHeader` they
  were given, which is a valid studio either way.

## 9. Rollout order

1. `starterPattern` field on `Studios` + onboarding input + step-2 UI (data only;
   both patterns still generate today's content).
2. Pattern-aware `createDefaultPagesForTenant`: nav visibility + home blocks +
   hero copy.
3. Pattern-aware commission settings and product-type seeding.
4. Tour branching (largest; can ship after 1–3, since the tour is guidance and an
   unused step is only a nag).
5. Admin affordance to switch pattern later.

Steps 1–3 are the shippable feature. Step 4 is the polish that makes the first
session feel tailored.

## 10. Decisions (locked 2026-11-20)

1. **Two patterns only**: `commissions` and `both`. No shop-only studio — a
   storefront with no way to request work is a dead end for a directory visitor.
2. **Step count**: a new step 2 (`Identity → Studio type → Aesthetic → Account`).
3. **Product types**: not seeded by onboarding at all. The `initializeStudioProductTypes`
   hook already writes `DEFAULT_PRODUCT_TYPES` for every studio (superseding the
   earlier "three starter types" decision).
4. **Pattern storage**: `Studios.starterPattern`, default `both`.
5. **Switch-pattern UX**: a settings toggle at `/admin/shop-settings`.

## 11. What shipped

- `src/features/onboarding/lib/starter-patterns.ts` — the single table: labels, hero
  copy, nav visibility, home blocks, starter commission types, starter product types.
- `Studios.starterPattern` (`commissions | both`, default `both`, hidden field)
  plus migration `20261120_003000_add_studio_starter_pattern`. The migration drops
  and recreates the enum, so a development database that applied the earlier
  three-value draft recovers; `resolveStarterPattern` maps a stale `'shop'` to
  `both` rather than failing.
- `StepPattern` as onboarding step 2 (two cards, `both` preselected); the progress
  bar, footer, draft restore and step validation were renumbered around it
  (`TOTAL_STEPS = 4`).
- `createDefaultPagesForTenant({ pattern })` — nav flags from the pattern, home
  blocks from `starterHomeBlocks`, and the commission FAQ guarded by
  `patternTakesCommissions`. With no shop-only studio that guard is always true
  today; it stays so a future shop-only shape cannot inherit refund copy it has no
  business asserting.
- `buildCommissionSettingsInput({ pattern })` seeds one `illustration` type.
- **No product-type seeding in onboarding.** The Studios `afterChange` hook
  (`initializeStudioProductTypes`) already writes `DEFAULT_PRODUCT_TYPES` (Digital
  Asset, Course, Character Base, Bundle) for every studio, so the shop has its
  categories whether or not its page is visible. One definition, owned by the shop.
- **Shop visibility settings** — `/admin/shop-settings`
  (`ShopSettingsViewServer` + `ShopSettingsView` + `setShopEnabled`). Turning the
  shop on sets the shop page's `showInHeader` and the studio's `starterPattern` to
  `both`; turning it off reverses both. Products, orders and product types are
  never touched, so closing a door does not delete a catalogue. Authorization is
  `getRoundActor`'s: session user, assigned studio, switcher cookie honoured only
  when it names a studio the user belongs to — the tenant never comes from the
  client. The view is styled from admin tokens (`_admin-theme.scss`), because the
  admin shell does not load Tailwind.
- **Shop topic tour**: `shop-visibility` (know) on the settings page, before the
  existing `shop-products` (do) step. The **setup** tour stays commission-only; see
  §5.5 for why.

**Home blocks use only self-hiding sections.** `openRounds` and `latestProducts`
return `null` with no data, so they are safe before the studio has a single round or
product. `bentoGrid`, `mediaGallery`, `shopCatalog` and `sectionedContent` draw a
frame regardless, so seeding them empty would ship a visible hole — the previously
shipped bento (Live Queue / Digital Assets / My Courses) is therefore gone from the
starter page. Artists add those blocks themselves once they have content; the block
picker offers all of them.

**The theme generator no longer writes copy.** `suggestedHeroTitle` /
`suggestedHeroBio` were removed from `ThemeGenerationSchema`, from the local
fallback, from the four poster fixtures and from `AESTHETIC_PRESETS`: a colour
prompt used to decide the artist's first words, and re-rolling the aesthetic
silently rewrote their landing page. Hero copy now comes from the pattern (a
business answer), and the theme answers only "how does it look".

**The theme does now write a starting layout** (§5.6): `heroComposition`,
`heroArtworkEmphasis` and `heroButtonEmphasis` are expanded by
`starter-block-layout.ts` into the blocks' hidden `layout` field — alignment,
text scale, artwork box, and whether the intro / second button exist at all. That
required one new layout capability, `HeroLayoutEntry.hidden`, plus a **Hidden**
list in the live preview so a removed element can be brought back. The default
composition expands to nothing, so any caller that omits it (the seed route, an
older client) produces exactly the pages it produced before.

## 12. Still open

- **Existing studios** keep `both` and their old bento carrying the shop links. A
  backfill that rewrites historical pages is deliberately not part of this change.
- **The header's own preview** of a hidden shop: a commissions-only studio's
  `/shop` URL still resolves when typed directly. Hiding is a nav decision, not an
  access rule; adding a 404 for a hidden system page would break the artist's own
  preview and any link they shared before closing it.
