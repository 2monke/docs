---
status: shipped
collections: [commissions, pages, product-types, products, studios]
features: [commissions, onboarding, shop, studios]
---

# Studio Onboarding Starter Patterns (SPEC)

> **Status**: SHIPPED for phases 1–3 (pattern field, onboarding step, pattern-aware
> seeding). Phase 4 (tour branching) and phase 5 (settings toggle) remain.
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
  (`src/features/onboarding/components/OnboardingView.tsx`).
- **The setup tour is a fixed 7-step commission sequence** (hero → pricing tiers →
  prices page → queue checklist → legal → Stripe → publish). Steps 2–4 assume
  commissions exist; there is no shop step.
- **Some blocks self-hide when empty** — `latestProducts`/`featuredProducts`
  (`BlocksRenderer.tsx:143`) and `openRounds` (`:214`). Others render a frame
  regardless (`bentoGrid`, `mediaGallery`, `sectionedContent`). This decides which
  blocks are safe to seed.

## 3. The principle that keeps this small

**A pattern seeds content; it never gates capability.**

- Choosing "Digital shop" does not disable commissions. It hides the commission
  pages from the nav and omits commission copy, so the studio reads coherently on
  day one.
- The artist can switch later (add a shop, start taking commissions). Switching
  **adds** an entry to the nav and reveals the system page that was already
  created; it never deletes content or regenerates pages.
- Consequence: store the choice as `starterPattern` on the studio — an
  initialisation hint consumed once — not as a permission.

This is why the plan can reuse `showInHeader` (already on pages) instead of
introducing routing rules: the system pages already exist for every studio, and a
hidden page is exactly "not part of this business yet, but here when you need it".

## 4. Where the choice lives

**New step 2** in onboarding: `Identity → Pattern → Aesthetic → Account → …`.
Three large cards, one tap, no free text. Inserting a step is friction, so it earns
its place by being the only question that changes what the studio *is*.

❓ Alternative to decide: fold the three cards into the bottom of Step 1 (Identity)
as a segmented control, keeping five steps. Cheaper for the artist, but the question
then competes with the name field for attention.

**Storage**: `Studios.starterPattern` — `select` of `'commissions' | 'shop' | 'both'`,
default `'both'` so existing studios and any API-created studio behave exactly as
they do today. Hidden from the studio form (`admin.hidden`) once set: it describes
how the studio was born, not a setting an artist should re-edit; a later switch is
a separate "add a shop / start taking commissions" action.

**Plumbing**: `ArtistOnboardingInput.starterPattern` →
`artist-onboarding.action.ts` passes it to `createDefaultPagesForTenant` and to a new
`buildCommissionSettingsInput({ pattern })` / `buildProductTypesInput({ pattern })`.

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

### 5.1 Commission settings (Commissions, Both)

Seed **structure, never prices**:

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

### 5.2 Product types (Shop, Both)

Three types, `isEnabled: true`, slugs matching what the home blocks link to:

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

All six system pages are still created for every pattern. What changes is
`showInHeader` + `headerOrder`:

- Commissions: `shop` → `showInHeader: false`
- Shop: `prices` + `queue` → `showInHeader: false`
- Both: all visible (today's behaviour)

Home `bentoGrid` cards per pattern:

- Commissions: **Commission Prices** → `/prices`, **Live Queue** → `/queue`.
  Drop the Digital Assets and My Courses cards.
- Shop: no bento. `latestProducts` (self-hides until the first product exists), then
  `mediaGallery` if the artist uploaded art.
- Both: Live Queue + Digital Shop cards, then `latestProducts`.

Every seeded block must be one that **renders nothing when empty** (see §2). A
`mediaGallery` or `sectionedContent` block seeded with no content would show a hole.

### 5.4 Hero copy

The aesthetic step already supplies `suggestedHeroTitle` / `suggestedHeroBio`
(`AESTHETIC_PRESETS`). The pattern supplies the *business* sentence, and the
existing copy stays as the fallback:

- Commissions: "Custom character art, open for commissions."
- Shop: "Digital art, ready to download."
- Both: the shipped "Your art / commissions" copy.

### 5.5 Setup tour

The 7-step tour is commission-shaped. Per pattern:

- **Commissions**: current tour, with any shop mention removed (it has none today).
- **Shop**: replace steps 2–4 (pricing tiers, prices page, queue) with
  "Add your first product", "Organise product types", "Delivery & files".
- **Both**: current tour plus one shop step, before the Stripe step.

This is the largest hidden cost of the feature: the tour is a declarative step
config, so branching is feasible, but it doubles the tour's content surface.

## 6. Shared base (all three patterns)

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
   all three patterns still generate today's content).
2. Pattern-aware `createDefaultPagesForTenant`: nav visibility + home blocks +
   hero copy.
3. Pattern-aware commission settings and product-type seeding.
4. Tour branching (largest; can ship after 1–3, since the tour is guidance and an
   unused step is only a nag).
5. Admin affordance to switch pattern later.

Steps 1–3 are the shippable feature. Step 4 is the polish that makes the first
session feel tailored.

## 10. Decisions (locked 2026-11-20)

1. **Step count**: a new step 2 (`Identity → Studio type → Aesthetic → Account`).
2. **Shop-only and the queue**: hidden from the nav, page created and intact.
3. **Product-type starter set**: three types (`character_base`, `psd`, `course`).
4. **Pattern storage**: `Studios.starterPattern`, default `both`.
5. **Switch-pattern UX**: a settings toggle (phase 5, not built yet).

## 11. What shipped

- `src/features/onboarding/lib/starter-patterns.ts` — the single table: labels, hero
  copy, nav visibility, home blocks, starter commission types, starter product types.
- `Studios.starterPattern` (`commissions | shop | both`, default `both`, hidden field)
  plus migration `20261120_003000_add_studio_starter_pattern`.
- `StepPattern` as onboarding step 2; the progress bar, footer, draft restore and
  step validation were renumbered around it (`TOTAL_STEPS = 4`).
- `createDefaultPagesForTenant({ pattern })` — nav flags from the pattern, home
  blocks from `starterHomeBlocks`, and **no FAQ** for a shop-only studio (its
  answers assert refund terms, which the artist must write).
- `buildCommissionSettingsInput({ pattern })` seeds one `illustration` type for
  commission-taking studios and none for a shop.
- Starter product types created during registration (errors logged, never fatal).

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

## 12. Still open

- **Tour branching** (phase 4): steps 2–4 assume commissions exist.
- **Settings toggle** (phase 5): flipping `showInHeader` on the relevant system
  pages, and adding starter product types if the shop is being turned on.
- **Existing studios**: untouched (`both`), so their old bento still carries the
  shop links. A backfill that rewrites historical pages is deliberately not part of
  this change.
