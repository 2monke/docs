# Platform Live-Studios Cards — the discovery row's theme contract

The platform home (`/platform-home`) renders a **Live Studios** row: one card per
published, Stripe-ready studio. It is the platform's shelf of artists, and the cards
are the first thing a visitor sees of anyone's work.

The card has two halves, and the split is the whole design:

- **The frame is the platform's.** Square plate, full-bleed square layout, platform
  surface (`bg-card`, `--border`), one name, one Open link. A visitor is browsing
  studios, not buying a commission; twelve differently rounded, differently shadowed
  plates would read as twelve websites in a blender.
- **The interior follows the studio's theme.** The artist's avatar, its crest, the
  ambient glow behind it, the Open button's colour and shape, and the name's
  typeface. That is what makes a row of cards read as a row of *people*.

> The first pass shipped a **banner strip** behind the avatar (duo-tone gradient or an
> uploaded image). In review it read as a second, arbitrary surface pinned to the
> card — a band with edges that could always look placed wrong. It was replaced
> before release by the **ambient glow**: the same two colours, bleeding in as light
> with no edge. The banner fields were removed from the schema with it; there is no
> dead column to migrate around.

## Why this is a projection, not the theme

`getFeaturedStudios` reads studios at **depth 2**, because the card needs the populated
theme, the artists, and each artist's avatar. It then **destructures the theme off the
studio** and returns a narrow `cardLook` instead:

```ts
export interface FeaturedStudio extends Omit<Studio, 'logo' | 'artists' | 'theme'> {
  artists?: Array<FeaturedStudioArtist | Artist | string> | null
  cardLook: StudioCardLook
  avatar: StudioAvatar          // { url } from the artist, or { url: null }
  isCommissionsOpen: boolean
}
```

The landing page is a **public** RSC payload. Spreading the theme document into it
would ship the studio's page background, header style, container width, section
rhythm, brand media and the theme's tenant back-reference to every visitor — for the
sake of a handful of values. `tests/int/platform-landing-studios.int.spec.ts` pins the
absence: the serialized row props must not contain the background colour or a `theme`
key.

## The look (`src/lib/theme/platform-card.ts`)

Pure and client-safe — plain data in, plain data out — so the server projection, the
card component and the tests share one implementation.

```ts
export interface StudioCardLook {
  crest: 'minimal' | 'gilded' | 'cyberpunk' | 'holo' | 'custom'
  crestImageUrl: string | null   // only while crest === 'custom'
  primaryColor: string   // normalized #rrggbb
  accentColor: string    // normalized #rrggbb
  buttonInk: string      // hsl(...) contrast ink for the Open button
  hoverColor: string     // accent when it reads on the card, else primary
  buttonShape: 'default' | 'pill' | 'sharp'
  fontFamily: string | null  // Google catalog name, or null
  fontStack: string | null
}
```

Every branch is total. Unknown crests fall back to `minimal`; a `custom` crest with no
populated upload falls back to the minimal ring; invalid colours fall back to the
platform palette (`PLATFORM_BRAND.primary` / `accent`); a three-digit hex is expanded
to six so `${colour}59` alpha suffixes stay valid CSS. A single bad field can never
blank a card or throw during a server render.

### Crests

| Value | Treatment |
|---|---|
| `minimal` | Borderless circle (`bg-muted/20`), nothing else. |
| `gilded` | Fixed champagne-gold ring (`#c9a24a` gradient) plus four corner trims. **The metal is fixed** — the studio's accent is the inner detail, not the frame, so a gold crest is recognisably gold on every studio. |
| `cyberpunk` | One hairline seam in the studio's primary, a soft neon glow, and four angular corner brackets. |
| `holo` | Conic foil ring built from the board's lavender family (`accentSoft → primaryLight → accent → surfaceAlt`) with a slow hue drift — pearlescent, not a rainbow. Motion only under `prefers-reduced-motion: no-preference`; reduced motion keeps the ring, drops the animation (`.live-studio-crest--holo` in `globals.css`). |
| `custom` | The artist's own artwork, drawn `object-contain` over the avatar window (a transparent PNG/WebP with an empty centre works best). It is never cropped; no upload resolves to `minimal`. |

All decorative layers are `aria-hidden` and `pointer-events-none`; the Open link is
the card's only control. The avatar window itself is never cropped differently by a
crest — a crest decorates *around* (or over) one circle.

### The square-image rule

The avatar and the custom crest are both framed in a circle, so both use the same
rule: **square, at least 400 × 400 px** (`AVATAR_IMAGE_RULE` / `CREST_IMAGE_RULE` in
`src/lib/utils/image-dimensions.ts`, ±2 px tolerance for export rounding). 400 px is
the display size ×2 — the portrait renders at up to ~200 px in a card, and a 2× asset
is what stays crisp on a phone.

Three layers, one implementation:

1. **Upload pre-flight** (`checkSquareImageFile`) — the artist profile avatar and the
   crest artwork refuse a non-square or too-small file with a message naming the
   label, the rule and the actual size. Fails *open* if the browser cannot decode the
   file; the server gate below is the closed half.
2. **Collection validator** (`squareImageUploadValidator`) — on `Artists.avatar` and
   `Themes.platformCardCrestImage`, reading the stored media's `width`/`height` and
   mime. This covers every write path, admin REST included.
3. **Field copy** — the hint under each control is derived from the same rule
   (`squareImageHint`), so the text cannot drift from the check.

Two deliberate exemptions: an **unchanged** value is not re-judged (a legacy
non-square avatar must not block an unrelated profile edit), and onboarding's mascot
passes `context: { skipAvatarDimensionCheck: true }` — that image is free-form
peeping-mascot art, not a portrait upload, and the flag is a Local API context value
a REST client cannot set. Known limit: the admin's crop drawer can re-frame an
already-uploaded avatar after the fact; the rule governs uploads, and the card's
`object-cover` keeps a non-square derivative presentable.

### Ambient glow

Two radial washes built from the studio's own colours: a wide soft light behind the
avatar (`radial-gradient(closest-side, primary-35%, accent-18% at 58%, transparent 78%)`
under `blur-2xl`) and a low accent glow near the action. Both are clipped by the
card's `overflow-hidden`, are `aria-hidden` + `pointer-events-none`, and carry no
motion — the glow is light, so a reduced-motion user loses nothing. It replaces the
banner band: no edge, no placement to get wrong, and twelve cards still read as twelve
palettes.

### The frame uses the platform palette

The card's frame and glow run the **brand board's plum → lavender ramp**, not the
studio's colours ("overcooked" was the right word: a five-stop studio-hued glow plus a
rainbow conic sweep made a curated page look like a Christmas tree, and every hue
competed with the brand).

- travelling frame → one band of the ramp, 2px, blurred 1px, rotating every 7s;
- hover/focus frame → same ramp held still, widened to 5px, blurred 3px;
- outer glow → a plum ring, one plum drop shadow, one lavender bloom (three stops,
  down from five);
- the single-hue studio tint behind the avatar lifts from 75% to full opacity.

`PLATFORM_GRADIENTS` (`src/lib/theme/platform-brand.ts`) is the one source:
`primary` = `#50324C → #B69ACF`, `soft` = `#EBD8F8 → #FAF6FC`, transcribed from the
brand board. `StudioCard` sets `--frame-start` / `--frame-end` / `--frame-start-glow`
/ `--frame-end-glow` inline from that constant, so the stylesheet never holds a second
copy. `tests/int/platform-brand.int.spec.ts` pins the gradients against the palette and
the CSS defaults against the gradients.

**Studio colour still reaches the card in exactly two places:** the action button
(filled with the studio primary) and one low-alpha ambient tint behind the avatar
(`hoverColor` at ~14%). That is identity without a light show.

### Palette coverage on platform surfaces

Where the board actually reaches, audited 2026-11-14:

| Surface | Source |
|---|---|
| `globals.css` `:root` | `PLATFORM_THEME_COLORS`, pinned token-by-token |
| `globals.css` `.dark` | the board's `darkMode` rung (`#1E1220`, `#2B1A2A`, `#3A2438`, `#B69ACF`, `#DABEFF`, `#F8F4FC`, `#C9B6D4`) |
| Landing feature tiles | `PLATFORM_TILE_HUES` — the six Additional Accents |
| Landing CTA band / highlighted fact | `darkSurface` + `darkTextPrimary`/`darkTextSecondary` + `accentYellow` |
| Landing artwork rail + shop mockups | `PLATFORM_BRAND` and `PLATFORM_GRADIENTS` |
| Card frame, glow, ambient tint | `PLATFORM_GRADIENTS` + one studio hue |
| Nav header, auth screens, onboarding, invite | theme tokens (`--primary`, `--accent`, `--muted`…) |

Two guard tests keep it there: no off-board Tailwind colour family on a platform
file, and no bare hex literal in `PlatformLandingPage` (the page builds gradients by
interpolation, so hexes hide in template strings). The artwork rail used to be
Tailwind's stock `rose-300` / `amber-200` / `orange-100` — the last off-board palette
on a curated page.

**Deliberate exceptions, not gaps:**

- **Semantic colours** — the board's pastels (`#6DBF8C`, `#E57373`, `#F7C86B`,
  `#7AA7FF`) are illustration fills; as text on white they measure 1.2–2.9:1. The
  status tokens stay on AA-tested rungs and are reached through `--success` /
  `--warning` / `--info`, never a raw `text-emerald-700`.
- **The `gilded` crest** — fixed champagne gold. Gold is a material metaphor, not a
  brand colour, and the board has no metal; the user chose it explicitly.
- **The `holo` crest** — no longer an exception. It was a literal rainbow; it now runs the board's lavender family, so the only off-board colour left on a platform surface is the gilded gold, which is a material rather than a hue. `tests/int/studio-card.int.spec.ts` asserts every hex in `StudioCard.tsx` is a `PLATFORM_BRAND` value or one of gold's four stops.

**The travelling frame.** `StudioCard` renders one `aria-hidden` span
(`.live-studio-sweep`). It is masked down to a frame with the standard two-layer
recipe (`mask: linear-gradient(#000 0 0) content-box, linear-gradient(#000 0 0)`
+ `mask-composite: exclude`), and its `::before` is a 210% square carrying a **conic
gradient** whose `rotate` animates on the compositor — the band runs around the plate
like a snake. It has to be a real element rather than a pseudo-element because the
mask must clip a rotating *child*, and a pseudo cannot mask a sibling. Animating
`--angle` on a conic-gradient would repaint every card every frame; `rotate` does not.
It is `pointer-events-none`, so the action links underneath stay clickable.

The padding *is* the frame width, so widening on hover is one `padding` transition
(2px → 5px, 240ms) — the gradient never changes size, only the ring that shows it. On
hover/focus the band stops (`animation-play-state: paused`) and switches to a held
`linear-gradient(120deg, …)`, blurred to 3px so it reads as light rather than a
painted border.

Two guards keep it sane:

- **Touch.** The hover rules sit inside `@media (hover: hover) and (pointer: fine)` so
  a tap does not leave a card lit. `:focus-within` carries the same state for keyboard
  users.
- **Reduced motion.** The sweep stops travelling and renders as a static dim ramp;
  hover still widens it. The card never depends on motion to be usable.

### No size jump on load

The row is server-rendered, so the first paint has no measured width and the marquee
layout is `pending`. That state used to borrow the caller's *still* classes
(`flex-1 max-w-[340px]`), then jump to the marquee width (container ÷ 2.4, often
~500px) on the first `ResizeObserver` callback — the "cards start small and grow a
moment later" flash. `MarqueeRow` now reserves the final width in CSS at pending time:
`width: calc(100% / visibleCards)` plus a 280px `min-width` floor, with inline
`flex: 0 0 auto` / `max-width: none` to defeat the still-mode classes. `visibleCards`
is the row's own column count (2.4 by default, **2.9** for the live-studios shelf), so
the reservation always matches what the measurement will produce. The gaps are
**not** subtracted: `resolveMarqueeCardWidth` divides by the column count and lets the
gaps sit on top, so subtracting them (a first attempt) reserved cards ~12px too narrow
and left a small jump of its own.

Two invariants make the reservation safe, and both exist because breaking them
produced an **endless refresh**:

- **The percentage needs a definite containing block.** The pass stays `w-full` while
  pending. A `shrink-0` pass sizes from its content while the content sizes from the
  pass — circular — and with `aspect-square` cards the row's height then oscillates,
  which toggles the page scrollbar, which re-fires the observer.
- **No scrollbar may appear mid-measurement.** The viewport is `overflow-hidden` while
  pending, not `overflow-x-auto`: the reserved widths deliberately overflow, and a bar
  appearing would change `clientWidth` and re-fire the observer.

`measure` also compares before it writes, so a repeated identical observation is a
true no-op instead of a re-render.

Applies to both rows (live studios and open rounds); pinned by
`tests/int/marquee-pending-width.int.spec.ts`, including a test that fires the
observer ten times with the same width and asserts nothing moves.

### The same plate on `/studios`

The workspace hub renders `StudioCard` too, with three switches flipped:

| Prop | Discovery row | `/studios` |
|---|---|---|
| `actions` | default single **Open** | **Admin** (primary) + **Storefront** (secondary, new tab) |
| `showCommissionsStatus` | `true` | `false` |
| `animated` | `true` (travelling band) | `false` (held gradient frame) |

The address is gone: identity is the avatar and the studio name. Two actions sit
side by side (`grid-cols-2`) rather than stacked — stacked buttons would have eaten
the avatar on a square plate — and both keep the 44px touch target.

The static variant is not a different look: it is the same brand ramp held still
(`[data-sweep='static'] .live-studio-sweep::before { animation: none }`), so a studio
looks like itself on both surfaces, and hover still widens and blurs the frame.

The hub also shares the platform's container width (`max-w-7xl`, matching the nav
header and the landing page) and gives the **Launch a New Studio** tile the same square,
sharp border language as the cards beside it — it used to be a `rounded-3xl` box, which
made the one grid on the page read as two different design systems.

### Typography

- Google-sourced themes carry the family (validated against `GOOGLE_FONT_NAMES`) and
  its stack; the row builds **one** stylesheet request via
  `buildGoogleFontsUrl(families)` — deduplicated, unknown names dropped, single-weight
  families (`Bebas Neue`, `Patrick Hand`) without a `wght` axis, `null` when nothing
  survives.
- Custom uploaded / external-stylesheet fonts are **not** loaded on the platform
  origin: the production CSP allows `fonts.googleapis.com` / `fonts.gstatic.com` but
  not arbitrary stylesheet hosts. The card stays on the platform display face instead
  of rendering a wrong fallback.

## The avatar — the artist, and nobody else

```
artist.avatar  →  initial
```

Every studio here is run by one artist, and the avatar is how that artist is
recognised across the site. It is the card's **only** image.

There is deliberately no logo fallback. Studio logos — the theme's `logo` /
`logoMobile` and the legacy `studios.logo` — still exist and still run the storefront
header, but they are brand assets: a wordmark sitting between two portraits reads as
a retail company that wandered onto an artist shelf. A studio with no avatar yet shows
its tinted initial instead, which is honest about the gap rather than borrowing
a brand face for a person.

The avatar renders `object-cover` into the circular window (the square rule means a
portrait is never cropped badly). `resolveStudioAvatar` ignores bare ids and
malformed values — a populated media document with a non-empty `url` is the only
thing that counts.

## Cache and invalidation

`getCachedFeaturedStudios` caches the row for hours under `FEATURED_STUDIOS_TAG`
(depth 2). Five writers already carry that tag, so a card refresh needs no new
invalidator:

- themes → crest / crest artwork / colours / font (`revalidateThemeCache`)
- media → a replaced avatar or crest file (`revalidateMediaCache`)
- artists → avatar change (`revalidateArtistCache`)
- studios → name / publication / Stripe readiness (`revalidateStudioCache`)
- sprints → open-commissions dot (`revalidateSprintsCache`)

## Files

- `src/lib/utils/image-dimensions.ts` — the square/size rule, the browser pre-flight
  and the Payload field validator factory.
- `src/lib/theme/platform-card.ts` — enums, defaults, hex normalization, look and
  avatar resolvers.
- `src/lib/theme/google-fonts.ts` — `buildGoogleFontsUrl`, `SINGLE_WEIGHT_GOOGLE_FONTS`.
- `src/features/studios/actions/get-studio.action.ts` — projection (`cardLook`,
  `avatar`), theme destructured off the returned studio.
- `src/components/platform/PlatformLandingPage.tsx` — maps `FeaturedStudio` →
  `LiveStudioCard`, builds the row's single font URL.
- `src/components/platform/StudioCard.tsx` — the shared plate: `StudioCard`,
  `AvatarCrest`, `CrestCorners`, `AmbientGlow`, and the action-button renderer.
  Used by the discovery row (defaults) and by `/studios` (static, two actions, no
  commissions dot). Props: `actions`, `showCommissionsStatus`, `animated`.
- `src/components/platform/LiveStudiosRow.tsx` — the marquee row; re-exports the
  card and its types so existing import paths keep working.
- `src/app/(frontend)/(platform)/studios/page.tsx` — the workspace hub, now the
  shared card: avatar + name + **Admin** / **Storefront**, no address, no dot, no
  travelling frame.
- `src/collections/Themes.ts` — the **Platform card** group.
- `src/components/admin/PlatformCardCrestField.tsx` — crest picker (icon specimens
  through the shared `ChoiceControl`).
- `src/components/admin/ThemeBrandImageField.tsx` — now serves any theme image slot
  (brand logo, mobile mark, crest artwork); only the header brand slots broadcast to
  the storefront live preview.
- `src/migrations/20261114_133500_platform_card_look.ts` — crest enum, the crest
  column, its media FK and index (idempotent).
- `src/app/globals.css` — `.live-studio-crest--holo` drift + reduced-motion guard.

## Tests

- `tests/int/image-dimensions.int.spec.ts` — the pure rule, the pre-flight (fail-open
  on an undecodable file) and the validator's exemptions.
- `tests/int/image-rule-wiring.int.spec.ts` — the rule is actually attached to
  `Artists.avatar` and `Themes.platformCardCrestImage`, the copy states it, and the
  onboarding opt-out exists.
- `tests/int/platform-card-look.int.spec.ts` — defaults, unknown values, custom-crest
  fallbacks, contrast ink, custom-font fallback, and that the avatar resolves to the
  artist's picture or the initial — never a studio logo.
- `tests/int/google-fonts-url.int.spec.ts` — dedupe, catalog filter, single-weight
  families, injection-shaped names.
- `tests/int/live-studios-row.int.spec.ts` — frame contract, every crest (including
  custom artwork), ambient glow, avatar framing, themed button, one stylesheet,
  marquee behaviour.
- `tests/int/platform-landing-studios.int.spec.ts` — projection (no theme document,
  no page background), avatar forwarding, font URL.
- `tests/int/theme-default-values.int.spec.ts` — schema defaults, the null-on-update
  restore, the crest upload's condition, and the absence of any banner field; the
  fields stay out of `sanitizeThemeDraft`.
- `tests/int/theme-editor-groups.int.spec.ts` — the eighth group, its anchor, and the
  absence of a showcase mapping.

## Debug reference

- **Card shows the initial instead of an avatar** — the studio has no artist avatar.
  Check the artist row's `avatar` media is populated at depth 2 and has a `url`. The
  card deliberately does not fall back to a studio logo.
- **Custom crest shows the minimal ring** — `platformCardCrest` is `custom` but
  `platformCardCrestImage` holds an id rather than a populated media document with a
  URL. The resolver falls back by design; check the relation and the media row.
- **Glow is very faint** — the wash is drawn at low alpha under `blur-2xl`; a card
  using the platform fallback palette shows the platform plum rather than a studio
  colour, which is correct.
- **Open button unreadable** — `buttonInk` comes from `getContrastForeground`; an
  invalid `primaryColor` is replaced by the platform plum before the ink is chosen.
- **Font not applied** — a `custom_upload`/`custom_url` theme is served the platform
  face on purpose (CSP). A Google family must be in `GOOGLE_FONT_NAMES`; the row
  requests one stylesheet for all cards.
- **Crest animation missing** — expected under `prefers-reduced-motion: reduce`; the
  foil ring stays, the drift does not.
