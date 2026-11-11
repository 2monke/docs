---
status: spec
collections: [artists, media, pages, queue, studios, themes]
features: [onboarding, pricing, shop]
---


# Studio Theme Customization Levers

Shopify-grade customizer controls on the `themes` collection (2026-09-21):
base body type scale, heading/body font split, heading scale/tracking/transform,
card artwork framing, page width, section rhythm, and button shape — every
lever live-previewable in the Payload admin.

All ten fields live in `src/collections/Themes.ts` (Typography, Style, and a new
📐 Layout group). Stepped sliders reuse `RangeSliderField`.

## The levers

- **Base Body Font Scale** (`fontSizeScale`, 85%–115%, slider) → `--font-scale`
  → `html { font-size: var(--font-scale, 100%) }`. Scales every rem-based size
  (text, prices, badges, inputs, spacing), matching the spec's "root rem base"
  behavior.
- **Heading Font split** (`headingFontSource` = `same_as_body` | `google` +
  `headingFontFamily` from the shared Google set) → `--font-display`.
  `same_as_body` resolves to the body stack, byte-identical to before.
- **Heading Font Scale** (`headingFontSizeScale`, 90%–130%, slider) →
  `--heading-scale`; wired heading sites use
  `text-[calc(<base>*var(--heading-scale,1))]`.
- **Heading Letter Spacing** (`headingLetterSpacing`, slider) →
  `--heading-letter-spacing`; sites use
  `tracking-[var(--heading-letter-spacing,<site value>)]`.
- **Heading Text Transform** (`headingTextTransform`) →
  `--heading-text-transform`; sites use
  `[text-transform:var(--heading-text-transform,<site value>)]`.
- **Card Aspect Ratio** (`cardAspectRatio`) → `--card-aspect-ratio` +
  `data-card-ratio` dataset attribute (see "Natural framing" below).
- **Container Max Width** (`containerMaxWidth`) → `--container-max-width`.
  Consumed by every page frame and every block's content container; unset
  falls back to each surface's built-in width (`none` for the full-bleed
  shop catalogue and queue board).
- **Section Spacing** (`sectionSpacing`) → `--section-spacing-y` /
  `--section-spacing-y-sm` (mobile / `sm:` values).
- **Button Shape** (`buttonShape`) → `--button-radius`. `default` (or unset) preserves
  each button's built-in proportional rounding (decoupled from cards and panels);
  `pill` = 9999px; `sharp` = 0px. Consumed by the `Button` and `ThemedLink`
  components (below) rather than by inline classes at call sites; the remaining
  inline usages are the deliberate exceptions listed under "Button Shape wiring".
  The `pill`/`pillActive` Button variants keep their semantic `rounded-full`.

### Accent ink (`--accent-ink`) — not a lever

`--accent` is a **surface** token: hover fills, 10% tints, status bands. Sites
that label with the accent (eyebrows, `Badge variant="accent"`,
`Button variant="dashed"`) read `--accent-ink` instead.
`generateThemeVarDeclarations` always emits it, defaulting to the theme's own
`accent`, so studio storefronts render byte-identically. The platform overrides
it with the brand plum (`PLATFORM_THEME_COLORS.accentInk`): the platform's
accent is the board's Accent Soft (#EBD8F8), which measures ~1.1:1 as text on
the page background and cannot carry a label.

## Neutral-value semantics (deliberate design)

Unset (or neutral) levers must not change any existing storefront. For every
lever whose neutral value means "keep the built-in design", the generator
**omits** the CSS variable:

- `headingLetterSpacing: 'normal'` → no variable; each heading keeps its
  built-in tracking.
- `headingTextTransform` unset → no variable; every site keeps its built-in ALL
  CAPS. (`none` is no longer the neutral value — it is an explicit
  "Natural / As Typed" and emits `--heading-text-transform: none`, see the
  casing note below.)
- `cardAspectRatio`, `containerMaxWidth`, `sectionSpacing`, `buttonShape`
  unset → no variable.

Two levers are *always* emitted rather than omitted, because their neutral
value is an absolute rather than a site default: `--font-scale: 100%` and
`--heading-scale: 1` (and `--font-display`, which is the body stack unless a
separate heading font is chosen). Every consumer of those two uses a fallback
that matches, so the rule still holds.

Each consumer site consumes the variable **with a per-site literal fallback**
(`var(--container-max-width,80rem)`, `tracking-[var(--heading-letter-spacing,-0.05em)]`,
…), so with no theme value the compiled CSS renders exactly the pre-lever
design. Sets and mappings live in `src/lib/utils.ts`
(`FONT_SIZE_SCALES`, `HEADING_FONT_SCALES`, `CARD_ASPECT_RATIOS`, …), which
also validate every value against the editor enums.

Two consequences of the fallback design are worth stating explicitly, because
both were wrong in the first cut:

- **The tracking scale is anchored to the fallback, not to zero.** Every wired
  heading site falls back to `-0.05em` (Tailwind's `tracking-tighter`), so
  `tighter`/`tight` must sit *below* that value — otherwise dragging the
  `RangeSliderField` "up" would loosen type the neutral value already sets
  tight. `NEUTRAL_HEADING_LETTER_SPACING` names the baseline, and
  `tests/int/theme-components.int.spec.ts` asserts both the monotonic ordering
  and that the call sites still use it.
- **`headingTextTransform` defaults to ALL CAPS (`uppercase`)**, matching the built-in
  editorial design of the storefront headings. Selecting "Natural / As Typed" (`none`)
  explicitly emits `--heading-text-transform: none;`, which overrides the CSS fallback
  and renders headings in their natural casing as typed. Existing themes are preserved
  rather than flipped: the old default (and the old "Keep built-in" option) stored
  `none`, which used to render ALL CAPS, so the migration
  `20261005_120000_migrate_legacy_theme_lever_values` folds stored `none`/`capitalize`
  to `uppercase` — no existing storefront changes on deploy, and Natural becomes a
  deliberate choice.

## Pipeline integration

- **SSR**: `generateFullThemeVariables()` → `generateThemeStyleCss()`
  (`src/components/ThemeStyle.tsx`) → `<style>` via `DynamicTheme`.
- **Live preview**: the `--…` declaration list is built once by the exported
  `generateThemeVarDeclarations(vars, { important })` so SSR and the
  `lp-theme-override` stylesheet (`LivePreviewThemeOverride`) can never drift.
- **Draft security**: every new key is whitelisted in `sanitizeThemeDraft()`
  (`src/lib/theme/theme-draft-codec.ts`) with its exact enum set — the `?draft=`
  URL vector remains a hard trust boundary. The studio preview **cookie**
  (a plain `document.cookie`, tossable from any sibling subdomain) is routed
  through the same `extractThemeFromDoc()` whitelist as the URL and
  postMessage vectors; it used to reach `applyThemeStyle` raw,
  and `tests/int/live-preview-cookie-sanitize.int.spec.ts` now pins that.
- **Draft plumbing**: `getThemeLivePreviewUrl` (`src/lib/theme/livePreviewUrl.ts`)
  and `AIThemeGeneratorAdminField` (preview-cookie mirror) both enumerate
  fields explicitly — new levers were added to both.
- **Structural dataset**: `ThemeScript` sets `data-card-ratio` before
  hydration; `LivePreviewThemeOverride` keeps it in sync (and deletes it when
  cleared).

## Natural framing — the one non-variable rule

`square` / `portrait` / `classic` are plain `aspect-ratio` values that
`next/image` fill layouts honour inside their positioned box. `natural` cannot
be expressed that way: an absolutely-positioned fill image can never size its
parent by its intrinsic ratio. The `img` is therefore returned to normal flow
by a static rule in `src/app/globals.css`:

```css
html[data-card-ratio='natural'] .card-media img {
  position: static !important;
  inset: auto !important;
  width: 100% !important;
  height: auto !important;
}
```

Consumers of the ratio lever are the four card surfaces, each tagged with
`card-media`:

- `src/features/shop/components/ShopItem.tsx` (fallback 16/10)
- `src/features/shop/components/ProductCarousel.tsx` (16/10, sm 4/3)
- `src/features/pricing/components/layouts/CategoryGallery.tsx` (4/3, sm 16/9)
- `src/features/pricing/components/layouts/HeroLayout.tsx` (16/10)

**Deliberately excluded** (structure-driven, not ratio-driven): the
`HomeBentoGrid` tiles and `MediaGalleryBlockComponent` grid (min-height /
`auto-rows` driven, aspect wiring would fight their layout), the nav header
(`NavHeader` glass variant), per-block `PageContainer` widths, the platform
chrome, and the site footer.

## AI generation

The AI theme generator produces the design levers too, and its three
consumers stay in sync through one contract:

- **Schema + prompt** (`src/lib/ai/generateTheme.ts`): `ThemeGenerationSchema`
  carries every lever as an optional enum with model-facing descriptions —
  pairing, tracking and framing guidance lives in both the zod `.describe()`
  texts and the system prompt. The no-API-key fallback
  (`generateDynamicThemeFromPrompt`, exported for tests) returns the same enum
  space deterministically: curated presets (cyber/gothic/pastel) plus a hashed
  path for arbitrary prompts.
- **Apply semantics** (`src/components/admin/aiThemeApply.ts`): the result is
  treated as a *complete* theme — every generator-controlled field gets the
  AI's value when present, otherwise its neutral default (null for the
  optional layout levers), so values from a previous generation can never leak
  through. The raw API response is reduced through `sanitizeThemeDraft` before
  it touches form state, and the Discard button restores a full snapshot taken
  before applying.
- **Onboarding** (`src/features/onboarding/actions/artist-onboarding.action.ts`):
  AI-generated levers flow into the Theme record created at signup; archetype
  presets keep the neutral defaults.

Tests: `tests/int/ai-theme-generation.int.spec.ts` — schema accept/reject per
lever, fallback presets round-tripped through both the AI schema *and* the
draft whitelist (enum-sync guarantee), determinism, and the apply/neutral
contract.

### The field artists actually see (2026-09-29)

The admin field (`AIThemeGeneratorAdminField`, the name is internal) presents itself
as **"Theme Generator"** and never mentions AI: an artist wants their studio to look
like them, and "AI" makes the palette feel borrowed. The prompt is a **textarea**
(3 rows, `resize: vertical`) — a studio's vibe is a sentence or two, and a
single-line input hides half of what the artist typed while they type it. Enter is a
newline there; ⌘/Ctrl+Enter (hinted under the field) runs the generator, and the
Generate button does the same. The setup guide's theme tip was updated with it (it
used to name the "AI Theme Generator").

**Why the review-portal `Textarea` could not be reused.** `@/components/ui/textarea`
is Tailwind + `--background`/`--input`/`--primary` tokens, and the admin route group
never loads them: `(payload)/layout.tsx` imports `@payloadcms/next/css` + `custom.scss`,
which pulls only the four admin SCSS partials — the repo's own note in
`_admin-theme.scss` ("Replace dead Tailwind utilities (globals.css is NOT loaded by the
(payload) layout, so w-4/h-4/text-* never applied in the admin)") records the same
conclusion. The generator keeps the component's existing house style: inline styles on
`var(--theme-elevation-*)`, which need no new stylesheet.

Pinned by `tests/int/theme-generator-field.int.spec.ts` (jsdom): no `\bai\b` in the
rendered copy, exactly one `<textarea rows="3">`, zero `<input>` elements — both
mutation-checked.

## The editor's groups

The theme form used to keep one `✨ Style` group holding the card style, the corner
rounding, the border weight, the header style and the button shape — five controls
that between them answered "which element am I changing?" nowhere. The groups now
name the element:

- 🎨 **Colors** · 🔤 **Typography** — unchanged, and still the two groups the guide
  dives into field by field.
- 🖼️ **Header** — `headerStyle`.
- 🃏 **Cards & media** — `cardStyle`, `borderRadius`, `borderWidth`, `artworkFraming`:
  the surfaces, border weight, corner radius, and artwork framing across cards,
  bento tiles, hero panels and socials.
  - **One framing decision (2026-10-04).** The group used to ask the framing question
    twice — "Product card artwork ratio" and "Gallery & pricing artwork ratio", four
    options each, no visible default. It is one intent, and the docs/AI had always
    treated it as one: `ArtworkFramingField` (a `ui` field rendering the shared
    `ChoiceControl`, the presentational core `OptionSwitcherField` also uses) reads
    both scoped keys and writes them together. The two selects remain in the schema
    as `admin.hidden` data — Payload keeps hidden fields in the form state (they
    render as `HiddenField`), so the AI generator's writes and legacy values survive
    saves — and the pipeline fallback (`product || gallery || cardAspectRatio`) is
    untouched. A stored state where the two disagree (older themes only) shows
    nothing selected plus a note; picking any option applies it to both. Labels now
    read Square / Portrait / Classic / **Original** (artist-facing rename only; the
    stored value stays `natural`). `landscape` (16:9) was retired before launch —
    it duplicated classic (16:10) — and the migration converts any pre-release row
    holding it to `classic` while it shrinks the enum.
  - **The card radius has no Pill.** The old option emitted 9999px on `--radius`
    and turned every tall card into a capsule (reported as "all cards becoming like
    egg", 2026-10-04). The radius scale is Sharp/Small/Medium/Large only; legacy
    themes that saved `full` still validate (the draft codec accepts it) but
    `generateFullThemeVariables` clamps it to the roundest supported stop (1rem, the
    same as Large), and `RangeSliderField` points a present-but-unknown value at that
    last stop so the editor and the render agree. Button pills are untouched — they
    live on `buttonShape`/`--button-radius`. A stored `full` is also folded to `large`
    by the same migration (`20261005_120000_migrate_legacy_theme_lever_values`) —
    identical 1rem rendering, now a value the schema validates, so select validation
    can never block a save on a legacy theme.
- 🔘 **Buttons & links** — `buttonShape`: corner shape for buttons and links, independent
  from card rounding.
- ✨ **Effects** — `useLiquidGlass`: the liquid-glass finish (frosted blur, edge
  lighting, glow) for surfaces that already use a glass style — Glassmorphism
  cards, `.glass-button` buttons, inputs and the glass header. It is a finish
  layer, not a global paint: Flat/Elevated/Bordered cards and solid buttons are
  left untouched by design (`globals.css` gates the recipe on
  `[data-liquid-glass="true"]` × the glass classes), and primary CTAs keep
  their solid fill even when glass-classed. The Colors section of
  `/theme-preview` renders a glass panel + glass/solid button pair so the
  toggle is visible regardless of the artist's card style.
- 📐 **Layout** — `containerMaxWidth`, `sectionSpacing`.

The groups are **unnamed** Payload groups, so this is presentation only: every field
still lives at the root of the document and no generator, live-preview or onboarding
code moved.

The names match the showcase's section names on purpose, and
`tests/int/theme-editor-groups.int.spec.ts` makes that a contract rather than a
coincidence: every lever must sit inside a group, every group must explain itself,
and for the element groups every field in the group must scroll the preview to that
group's own section — so "Cards" in the editor and "Cards" in the preview cannot
drift apart. The same spec keeps the `guide-theme-*` anchors the setup guide and the
scroll bridge target.

## The controls — one kit, two densities (2026-10-04)

The controls had grown one family per field: two native react-selects, a native
checkbox, colour cards, option cards carrying a `‹ Switch ›` cycler, sliders, and
a two-segment toggle with *its own* physical switch. Eight fields repeated the same
`‹ Switch ›` pill, selection was styled three different ways (indigo card, neutral
segment, indigo glow), option labels were clipped mid-value in the narrow
live-preview rail (`Default — built-in butto…`, `Compact (1…`), the slider had no
thumb (inline styles cannot reach `::-webkit-slider-thumb`), nothing had a real
focus ring, and every widget re-declared its own radii, heights and colours instead
of the admin's `--ap-*` token contract.

The pass that fixed this is one shared stylesheet — `_admin-fields.scss`, imported
by the admin's `custom.scss` — plus one rule for which control a field gets:

- **Choice** (`OptionSwitcherField`) — the only control for "pick one of these".
  Description-carrying options (`Title — detail`) render as a two-column card grid
  (labels wrap, they never ellipsize); two or three description-less options render
  as a segmented control. The retired `ToggleField` became that segmented mode, so
  the binary selects (`headingFontSource`, `headingTextTransform`), `fontSource`
  and `headerStyle` all render through it — no native select is left in the form.
- **Scale** (`RangeSliderField`) — ordered levers; filled track, clickable ticks, and
  a real visible thumb drawn in CSS. The active tick marks the current step.
- **Specimen** (`FontPickerField`) — the 13-font chip wall, every chip set in its own
  typeface.
- **Switch** (`SwitchField`) — booleans (`useLiquidGlass`): a real (visually hidden)
  checkbox keeps keyboard, screen-reader and the generator's DOM-mirror
  (`setDomCheckbox`) paths working under a styled track.
- **Colour** (`ColorPickerField`) — swatch + monospace hex in one card that takes
  the focus ring through `:focus-within`.

Header grammar is identical everywhere: label + `?` on the left; a prev/next cycler
on the right for card fields. The `‹ Switch ›` pill is gone, and so is the
current-value **readout chip** that briefly sat next to the cycler (removed
2026-10-04): it wrapped the accessory to a second line at rail widths ~290px, and
it was redundant anyway — the highlighted card with its check, the checked specimen
chip, and the bold tick all show the value. The header is explicitly
`flex-wrap: nowrap`: when space runs out, the label wraps its own text and the
cycler stays put. An **unset** field (this editor's neutral = "inherit the built-in
styling") shows nothing selected, and cycling out of unset enters the list at the
top rather than skipping the first option.

The rail is a **container**, not the viewport: with the live preview open the form
is ~290px wide, so `@container ap-field (max-width: 360px)` collapses the card grid
to one full-width row per option and stacks segmented options, while the slider's
tick row wraps instead of clipping. The header never re-flows (pinned by a
stylesheet guard). Measured live (see Verification): 1-column cards at 287px,
2-column at 492px, zero clipped text in both.

Mechanical rules the tests enforce rather than trust:

- `tests/int/option-switcher-field.int.spec.ts` — card vs segmented mode chosen by
  the copy itself, `radiogroup`/`radio` semantics, arrow-key selection, cycler,
  no `Switch` literal, no readout chip, and stylesheet guards that option
  titles/subtitles never declare `nowrap`/`ellipsis` (they wrap instead) and the
  header never re-flows.
- `tests/int/font-picker-field.int.spec.ts` — 13 chips, per-chip typefaces,
  cycler wrap-around, arrow keys.
- `tests/int/range-slider-field.int.spec.ts` — active tick = current step, one stop
  per option, tick jumps, drag maps to the option value.
- `tests/int/switch-field.int.spec.ts` — named checkbox (what the generator's
  `setDomCheckbox` mirror targets), `label[for]` association, boolean writes.
- `tests/int/color-picker-field.int.spec.ts` — swatch, hex input, label wiring.
- `tests/int/admin-theme-contract.int.spec.ts` — `_admin-fields.scss` is a watched
  sheet: the kit cannot regress to colour/radius literals.
- `tests/int/theme-editor-groups.int.spec.ts` — unchanged: every lever still sits in
  its element group and the `guide-theme-*` anchors the setup guide and the scroll
  bridge target are intact.

The collection's groups are ordered **Header → Colors → Typography → Cards & media →
Buttons & links → Layout → Effects** — the header leads because it is the first
thing a visitor sees on the page, so it is the first control the artist meets; the
setup guide's walk (colors → typography → cards → effects) skips it and still runs
front-to-back. The order is a deliberate choice, pinned by
`tests/int/theme-editor-groups.int.spec.ts` so a reorder has to be a conscious edit.
The long react-select labels were rewritten as title + reason
(`Floating Pill Island (Centered Floating)` → `Floating pill island — centered,
floats above the page`, `Google Fonts Library` → `Google Fonts`). Values were not
touched — only labels, rendering and styling.

## The showcase route (`/theme-preview`)

Why: the editor's live preview used to point at the studio **home page**, which is
one arrangement of blocks — an artist changing card aspect, button shape or
container width was editing levers that might not occur on the screen in front of
them at all. `/theme-preview` is a synthetic page (no CMS document, no tenant data)
that renders every themable surface from the *real* storefront components
(`Button`, `Card`, `ShopItem`, `FAQBlockComponent`, `SocialLinksBlockComponent`,
`Container`, form controls) with hard-coded samples, so it renders the same for a
studio created a minute ago and one that has been live for a year.

- **Placement & policy**: `src/app/(frontend)/(studio)/theme-preview/page.tsx`, so
  it inherits the studio header/footer and the theme pipeline by placement;
  registered in `STUDIO_ONLY_PREFIXES` (`src/lib/security/route-access.ts`) so platform hosts
  404 it. `theme-preview` is also reserved
  (`RESERVED_STUDIO_ROUTE_SLUGS` in `src/lib/blocks/systemPages.ts`, enforced in
  `enforcePageTypeRules`) — a CMS page with that slug would be shadowed by the
  route and silently unreachable.
- **Reachability** (`src/lib/theme/theme-showcase-access.ts`): a live-preview request, or
  the studio's artist / a platform admin. Everyone else gets a 404 — including on a
  *published* studio, where the storefront's own draft guard would otherwise let any
  visitor through. Not indexed (`robots: noindex`), never linked from the site.
- **Coverage contract** (`src/components/theme-preview/showcase-sections.ts`): a
  registry maps each section to the theme draft keys it demonstrates, and
  `tests/int/theme-preview-showcase.int.spec.ts` reads the canonical lever list out
  of `sanitizeThemeDraft` itself — a new lever cannot ship without a showcase slot,
  and the registry cannot claim a key the codec would reject. The same spec renders
  the page and asserts one section per registry id, in order, so the coverage rule
  can never be satisfied by a registry the page ignores. Because a registry entry
  can claim a lever its section never renders, the lever whose consumer is easiest
  to miss — `liquidGlass` — is pinned further: the Colors section must contain a
  real `.glass-surface` panel and a `.glass-button` sample (the demo pair added
  2026-10-04, on a busy artwork backdrop so the blur is actually visible).
- **Preview wiring**: `getThemeLivePreviewUrl` builds
  `${baseUrl}/theme-preview?preview=theme&draft=…&previewToken=…` (the token is what
  unlocks a *draft* studio) and `MobileLivePreviewBridge` appends the same path for
  its "open in new window" — both pinned by the spec above.
- **No route-segment config**: `cacheComponents: true` rejects
  `export const dynamic`, and the route does not need it — `headers()` plus the
  guard's cookie reads make it uncached by default.
- **The preview never navigates away**: `PreviewNavGuard` swallows clicks on
  off-page links inside the showcase (the sample `ShopItem` links to
  `/shop/<slug>`, which has no entry behind it), while leaving the section-nav
  anchors, new-tab links and external URLs alone.
- **Section-nav clicks stay inside the preview** (`src/lib/theme/preview-scroll.ts`):
  in-page anchors scroll through a window-scoped `window.scrollTo` (honouring the
  target's `scroll-mt`) — never `element.scrollIntoView()`, which inside an iframe
  walks ancestor browsing contexts and moves the *hosting admin window*. That was a
  real reported bug (2026-10-04): every click of a showcase nav chip scrolled the
  editor page +310px, forever — 155px from `LivePreviewNavigationLock`'s
  `scrollIntoView` plus 155px from `HashScroll`, whose effect the anchor handler's
  `history.replaceState('#…')` re-ran through the router with an unchanged hash.
  Fixed by sharing the helper between `LivePreviewNavigationLock` and
  `ShowcaseScrollSync`, and by having `HashScroll` remember the last-scrolled hash so
  router re-renders stop re-scrolling. Reproduced and verified against the running
  dev server; pinned by `tests/int/preview-scroll.int.spec.ts`,
  `tests/int/hash-scroll.int.spec.ts` and the navigation-lock spec (all of which
  fail if anything scrolls with `scrollIntoView` again).
- **The split preview shows a real desktop** (2026-10-04): the inline theme
  preview renders the page at *pane ÷ zoom* CSS px and scales it to fit (Payload's
  native iframe zoom). At the default 100% that means a ~1000px "desktop" — the
  Container Width lever had almost nothing to compare and cards needed extra
  scrolling. `ThemeLivePreviewAutoOpen` now drops the theme view to **75% once**
  when the split window activates (≈1400px desktop scaled into the pane); **50%**
  (≈2080px — the whole 1600px container scale) is one click away in the zoom
  dropdown, which is re-enabled in the toolbar next to the Mobile/Desktop toggle
  (the rest of the toolbar simplification — hidden breakpoint select and
  resolution inputs — stands). The default never fires when the control already
  reads anything other than 100%, so an artist's session choice wins. Pinned by
  `tests/int/theme-live-preview-auto.int.spec.ts` (75% at 100%, artist choice
  respected, non-theme routes untouched, stylesheet guard that the zoom control
  stays visible while the verbose controls stay hidden).
- **Category filter — show only what you're looking at** (2026-10-04): the nav
  (`ShowcaseNav`) is now the showcase's category control — **All** (the default:
  the full page), **Header** (the page frame: the real nav chrome + intro, which
  is what the Header style lever changes) and one chip per section. Picking a
  category renders only its section: the state is a `data-showcase-filter`
  attribute on the showcase root, and `showcaseFilterCss(sectionIds)` generates
  the rules from the registry (a new section cannot ship without its hide rule;
  hiding/showing costs no re-render of the heavy sections). While a category is
  selected, the preview **follows the editor**: focusing a theme field in the
  admin switches the filter to that field's section (via the same
  `listenShowcaseScroll` transport the scroll bridge uses) — and it does so from
  **any** state: editing a Colors field selects Colors, editing Typography selects
  Typography. Clicking **All** is a manual gesture — the full page stays until the
  next field edit takes over again. Pinned by
  `tests/int/showcase-filter.int.spec.ts` (generated CSS, chip states, All
  immunity, follow-the-editor) and the style-tag pin in
  `theme-preview-showcase.int.spec.ts`.
- **The preview follows the field you are on**: every theme field name maps to a
  showcase section (`src/lib/theme/theme-showcase-scroll.ts`) — focus or click it and the
  preview scrolls there. Per field, not per group: the `✨ Style` group holds the
  card style, radius, border weight, header style *and* button shape, which live in
  four different sections. A section only claims a lever it renders a consumer for,
  which is why `borderRadius`/`borderWidth` point at the hero/FAQ/socials section
  (`var(--radius)`, `var(--border-width)`) rather than at the button matrix. The
  bridge is `ThemeShowcaseScrollBridge` (admin, theme view only, once per section
  change); the receiving end is `ShowcaseScrollSync`, which scrolls by
  `getElementById` — an id lookup rather than selector syntax, so a hostile
  postMessage cannot smuggle selector operators. The message rides the existing
  preview transports (`sendToPreviewSurfaces`), as a new message type beside the
  block-preview protocol rather than a new block action.
- **The guide yields to the artist between target changes** (2026-10-02, reported
  as "changing Button shape scrolls the preview back to Colors"): the spotlight
  used to re-broadcast `artpack:tour-target-active` on every 400ms re-measurement,
  so the bridge re-announced the guide's section moments after the artist focused a
  field — and every draft-driven preview reload re-broadcast it through
  `PAYLOAD_SHOWCASE_READY`. The event is now dispatched **only** when the step,
  sub-step or selector changes (the spotlight's reset effect is the single
  announcement point; the poll never announces). A preview reload re-broadcasts the
  last announced section, which is then the artist's own field, not the guide's.
  Pinned by `tests/int/tour-target-scroll.int.spec.ts` (mount + polls = exactly one
  announcement; a selector change = exactly one more).
- **The preview's scroll never chains into the admin** (same report: "scrolling the
  showcase scrolls the controls window too"): the showcase page ships
  `html { overscroll-behavior: contain }` (`ThemeShowcase`), so scrolling the
  preview to its top/bottom edge no longer propagates into the parent admin
  document — the left controls pane only moves when the artist scrolls it. Pinned by
  a render assertion in `tests/int/theme-preview-showcase.int.spec.ts`.
- **Section order** (`showcase-sections.ts`) leads with the panels that carry their
  own style attribute — header, cards, buttons, each with its own heading — then
  colors, typography, sections & rhythm, width, forms, prose. The page renders in
  registry order through an id → content map, so reordering a section is one line in
  the registry and the render pin follows automatically.

Not yet verified in a browser: whether each section reads as clearly as intended at
375px in the preview panel, and whether the section spacing lever is visible enough
between sections. Both need a session on a real studio.

**Gotcha — no route-segment config.** The repo runs `cacheComponents: true`
(`next.config.mjs`), where `export const dynamic = 'force-dynamic'` is rejected at
compile time (`Route segment config "dynamic" is not compatible with
nextConfig.cacheComponents`), which took the page down with a 500 until it was
removed. It is not needed either: the page reads `headers()` and the visibility
guard reads cookies, so the route is uncached by default — with Cache Components,
caching is opt-in (`'use cache'`), never the default. The same rejection now
applies to `src/app/robots.ts` as of Next 16.4, where the segment config was
removed for the same reason (it reads `headers()`).

## Other consumer sites

- **Heading sites**: `PageHeader`, `HeroBlockComponent`
  (`getHeroTitleFontSize`), `HomeBentoGrid` (section + tile titles),
  `ProductContent` (both titles), `queue/[id]/page.tsx`, `BundleContentsWrapper`.
  All class strings are written literally so Tailwind's source scanner sees
  them (no runtime-built class names).
- **Section rhythm**: `FAQBlockComponent`, `SocialLinksBlockComponent`,
  `MediaGalleryBlockComponent`, `HomeBentoGrid` sections — each keeps its
  built-in padding as the literal fallback.
- **Container width**: every storefront frame renders through the
  **`Container`** component (`src/components/ui/Container.tsx`) — `w-full mx-auto`
  plus a width preset whose fallback is the surface's pre-lever width:
  `bleed` (none), `3xl` (48rem), `4xl` (56rem), `6xl` (72rem), `7xl` (80rem,
  the default) and `wide` (1400px). Presets exist instead of one hardcoded
  default because the surfaces genuinely differ — a hardcoded
  `max-w-[var(--container-max-width,1400px)]` would stretch the FAQ (896px
  today) and the full-bleed shop catalogue the moment the lever is unset.
  `PageContainer` accepts the same presets as an opt-in `width` prop for
  storefront blocks; its `maxWidth` stays the semantic choice for forms and
  reading pages.
  - Consuming surfaces: hero grid, bento, `priceList` + its `Socials`
    section, commission form, FAQ / `mediaGallery` / `socialLinks` (via
    `PageContainer width`), richText (prose bodies stay full-bleed as before,
    non-prose takes 4xl), `sectionedContent`, `shopCatalog`, `queueView`,
    product carousel blocks, the product page (`ProductContent` +
    `ProductSkeleton`), the prices page, and the queue-detail header + main.
  - Blocks stay full-bleed (backgrounds, ambient glows, hairline borders);
    only their inner frames narrow. There is deliberately no single
    `BlocksRenderer` wrapper — it would clip those decorations whenever a
    width is chosen.
  - Not wired (deliberately): form/reading pages (login, register, reset,
    tip, review submission) keep their intentional widths, the chat page is
    a fixed-height app shell, and `PageHeader`'s centered reading column
    keeps its built-in width.

### Button Shape wiring

Two common components own the lever, so sites never hand-write
`rounded-[var(--button-radius, …)]`:

- **`Button`** (`src/components/ui/button.tsx`): per-size fallbacks plus a
  `shape` prop — `default` (the per-size rectangles), `pill`, and `feature`
  (the tall rounded CTAs). The shape variant is declared last among the cva
  variants so it beats the per-size radius in the class merge, while a
  caller's `className` still wins.

  `shape` is a **cva variant, not a DOM prop**, so it only has an effect if the
  component forwards it into `buttonVariants({ … })`. It did not, at first:
  `ButtonProps extends VariantProps<typeof buttonVariants>`, so TypeScript
  accepted `shape="pill"` at 21 call sites that had just deleted their literal
  `rounded-full` — and the buttons silently kept their per-size rectangle
  radius while the prop leaked onto the DOM as a bogus `shape` attribute.
  `tests/int/button-shape.int.spec.ts` renders the component (the only
  assertion that can catch this class of bug) and locks the contract down.
- **`ThemedLink`** (`src/components/ui/ThemedLink.tsx`): the themed anchor for
  CTA links — `shape="theme" | "pill" | "rect"` (`theme` falls back to the
  studio's global radius) — plus the liquid-glass `glass-surface` hook (inert
  unless the theme enables liquid glass).

Call sites that only duplicated their size's default radius dropped the
override class entirely (tip checkout, form status, reject button,
social-confirm button). One conversion is deliberately *not* byte-identical:
`VerifiedQuoteView` used `rounded-2xl sm:rounded-[3rem]` (1rem / 3rem) and
`ArtistReviewForm` used `rounded-3xl sm:rounded-[3rem]` (1.5rem / 3rem), and
both now share the single `feature` shape — so `VerifiedQuoteView`'s mobile
radius moves 1rem → 1.5rem. One variant cannot express both, and that
1rem/1.5rem split was an accident of the old markup rather than intent.

The remaining inline `var(--button-radius…)` usages
are non-component elements where neither applies:

- `ProductSidebar`'s magnetic buy button is a `motion.button` (framer-motion
  style props + refs); wrapping it in `Button` would concatenate — not merge —
  the base classes, so it keeps its own radius classes.
- Decorative CTA chips and price tags (`ShopItem`, `HeroLayout`,
  `HomeBentoGrid`'s primary-solid span) sit inside card-wide links — anchors
  cannot nest, and these elements are not interactive.

## Verification

- `npx tailwindcss -c tailwind.config.ts -i src/app/globals.css -o /tmp/out.css`
  — confirms every new arbitrary class compiles to real CSS
  (`font-size: calc(… * var(--heading-scale,1))`, `aspect-ratio: var(…)`,
  `border-radius: var(--button-radius,…)`, …). Verified 2026-09-21.
- `npx tsc --noEmit` — clean, on the working tree and on a clean worktree of
  the PR revision.
- `vitest run tests/int/theme-components.int.spec.ts tests/int/theme-draft-codec.int.spec.ts tests/int/ai-theme-generation.int.spec.ts tests/int/button-shape.int.spec.ts tests/int/live-preview-cookie-sanitize.int.spec.ts`
  — 46/46 pass, including the generator/sanitize/script/CSS/AI-contract
  assertions, the tracking-scale monotonicity guard, the `Button shape`
  render contract, and the preview-cookie whitelist guard.

- Verified 2026-10-04 (theme control kit): `npx tsc --noEmit` clean; `sass`
  compiles the admin `custom.scss` including `_admin-fields.scss`; the field/theme
  spec set is green under `yarn test:int` (`option-switcher-field` 18,
  `font-picker-field` 7, `range-slider-field` 4, `switch-field` 4,
  `color-picker-field` 3, `field-help-tooltip` 6, `theme-editor-groups` 6,
  `theme-generator-field` 3, `theme-components` 30, plus the tour and live-preview
  specs); the admin-theme contract spec finds zero offenders in the new kit file.
- Verified live 2026-10-04 (split-pane zoom): against the running editor, the
  theme preview's toolbar read **75%**, the iframe carried `transform: scale(0.75)`
  (a ~1400px desktop scaled into the split pane), the merged Artwork framing
  control rendered as one row of four options with the legacy ratio fields gone,
  the category chips filtered (`All` → five sections, `Colors` → only Colors),
  and focusing a typography field while `Colors` was active switched the filter to
  `typography` — only that section visible.
- Verified live 2026-10-04 (artwork framing merge + showcase categories): the merged
  control renders four stops and writes both scoped keys (division: `option-switcher`
  18, `artwork-framing-field` 5, `showcase-filter` 8, `theme-preview-showcase` 15,
  `theme-editor-groups` 8 — plus the wider theme set, 161/161 across 18 specs);
  `tsc --noEmit` clean; import map and `payload-types` regenerated with verified
  deltas. Live-verified against the running editor (see the zoom entry above —
  the same run covered both features).
- Verified 2026-10-04 (card radius pill): the legacy `'full'` value clamps to
  `1rem` (same as Large) instead of 9999px; the editor field offers only
  Sharp/Small/Medium/Large; the AI schema rejects `'full'`; the slider shows an
  out-of-enum legacy value at the roundest stop. Pinned by `theme-components`
  (clamp), `theme-editor-groups` (options), `ai-theme-generation` (schema
  rejection) and `range-slider-field` (legacy display).
- Verified live 2026-10-04 (showcase nav scroll): before the fix, the same click on
  the Colors chip moved the admin window +310px and then +155px per click
  (0→433→743→1053→1363) while the preview stayed aligned; the timed probe placed
  the jump at ~300ms after the click (the `HashScroll` timeout) and a non-anchor
  click moved nothing. After the fix: admin `scrollY` is 0 after every click and
  stays 0 at +80/400/800/1600/2600ms, while the preview still scrolls between
  sections (`#typography` → `#colors` verified).
- Verified live 2026-10-04 against the running dev server (Playwright probe as the
  seeded artist, read-only; screenshots + raw JSON in `.screenshots-review/after`):
  zero `Switch` pills (was 8), zero native selects / react-selects (was 3 native
  controls), 0 clipped elements in the form at 1440px (live preview open, 287px
  rail), 1000px (492px rail) and 1920px; card grid measured 1 column at 287px and
  2 columns at 492px (container query); 1 switch, 5 sliders, 32 option
  radios, 13 font chips with the kit's computed styles applied (`border-radius`,
  accent-tint selected state, 6px track); group order on screen =
  Colors → Typography → Header → Cards & media → Buttons & links → Layout →
  Effects; the Card Style cycler cycled the selection live
  (`Glassmorphism → Flat → Glassmorphism`) against the real preview. (The probe
  ran before the readout chip was removed 2026-10-04.) One
  pre-existing quirk left untouched: Payload's `.group-field--top-level`
  `padding: 40px var(--gutter-h)` overhangs the rail at ~1000px (its content box
  aligns; present before this pass with the same measurements).

- Verified 2026-10-04 (liquid-glass legibility): `theme-preview-showcase` 14/14 —
  including the new pin that the Colors section must render a real `.glass-surface`
  panel and a `.glass-button` sample — the wider theme set 126/126, `tsc --noEmit`
  clean, `payload-types.ts` regenerated with a verified one-line delta (the new
  description only). The demo's on/off contrast is a one-glance check in the
  running preview: reload the theme editor and watch the panel over the artwork.

- Fixed 2026-10-05 (review pass): legacy values are migrated, not just tolerated
  (`20261005_120000_migrate_legacy_theme_lever_values`: `full` → `large`,
  `none`/`capitalize` → `uppercase` — select validation can no longer block a
  save on a legacy theme); the AI schema's `headingTextTransform` enum is
  `none | uppercase` and `applyThemeToForm`'s omitted-field default is
  `uppercase`, so the model can no longer emit a value the select rejects; the
  admin-theme contract spec is green again (its fifteen pre-existing color/radius
  offenders were replaced with — or had missing steps added to — the token kit,
  and the `!important` floor was re-baselined to the real count, 1461);
  `yarn generate:importmap` now runs as the first step of `build`, so a stale
  committed import map cannot ship a Theme editor missing its field controls.
  Verified: `tsc --noEmit` clean; compiled admin CSS diff shows only the token
  additions and the value-preserving replacements (the one declared change,
  9999px → `--ap-radius-pill` on the 18px help button, renders identically);
  theme set 199/199, tour + adjacent set 241/241, full `test:int` 120/134 files
  (13 of the 14 failing files are sandbox-environment failures — blocked loopback
  services (`EPERM` Postgres connects, Redis hook timeouts); the 14th,
  `status-tokens`, is a pre-existing offender in a component this branch never
  touches).

## Follow-ups (not in scope)

- Onboarding archetypes (`src/features/onboarding/types.ts`) still set only the
  classic fields; new levers default to neutral for studios created from a
  preset (AI-prompt onboarding gets the levers — see above).
- Bento/gallery tiles could opt into the ratio lever if their layout moves to
  aspect-driven boxes.
