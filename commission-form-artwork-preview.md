# Commission Form: Artwork Beside the Questions

The commission form asks a customer to choose a piece of art and then, on a
wide screen, used to show them only text. The form column is capped at 36rem
and the rest of the 80rem page frame was empty (or filled with the mascot).
Worse, the two choices that actually define the purchase — which tier and, for a
YCH round, which seat — were a `<select>` and a list of buttons, with the seat
sheet buried under the upload field.

Since 2026-11-10 the form is two panes from `xl` up: the artwork on the left,
staying put while the questions scroll on the right.

```
/apply (wizard)                     CMS block (inline)
┌───────────────────────────┐      ┌──────────────────────────────┐
│ ① Round ─ ② Project ─ ③   │      │ [round] [type cards]          │
│                           │      ├──────────────┬───────────────┤
│  Which round…?            │      │ artwork      │ questions     │
│  [card] [card]            │      │ 3fr (60%)    │ 2fr (40%)     │
│                           │      │              │               │
│        [Back] [Continue]  │      │ sticky pane  │ page scrolls  │
└───────────────────────────┘      └──────────────┴───────────────┘
        ↓ Continue
┌──────────────────────────────────────────────┐
│ ② …then the application itself:              │
│ ┌──────────────────────────┬───────────────┐ │
│ │ 3fr (60%) artwork        │ 2fr (40%) form│ │
│ └──────────────────────────┴───────────────┘ │
└──────────────────────────────────────────────┘
```

Below `xl` nothing about the flow changes: one column, the seat sheet back
inline above the seat list, exactly as before. The wizard also renders below
`xl` (its steps stack), on the same document scroll as every other width.

## Where the artwork comes from

Everything the pane shows is artwork the artist already uploaded for the prices
page. No new collection, no new upload field, no migration.

| Selection | Pane shows | Owner |
| --- | --- | --- |
| A tier is chosen and has `exampleImages` | those images | `TierItemEditor` → `pricingTiers[key][].exampleImages` |
| A tier is chosen but has no images | the type's category gallery for that section, else for the type | `CommissionSettings` → `commissionTypes[].sections[].images` |
| Nothing chosen yet | the type's category gallery | same |
| No gallery, no images on the chosen tier | the type's first tier example | `exampleImages` |
| Seat round | the round's seat sheet with one marker per seat | `sprints.seatMapImage` + `commission-seats.marker {x,y}` |
| Nothing at all | a labelled placeholder naming the type | — |

The artwork carries **no caption**: the chosen tier is already named by the
form's own selects and repeated in the summary rows beside the picture, so a
name printed over the art was the same information a third time — and it sat on
top of the picture the customer opened the pane to look at.

Its **eyebrow** is mode-dependent, because the two modes make different promises:

| Pane | Eyebrow | Why |
| --- | --- | --- |
| A tier's artwork | **Example work** | Those are past commissions demonstrating style and detail level. The piece is drawn *after* the request, so "what you're buying" would promise an artwork that already exists — a shop product, not a commission slot. |
| A seat round's sheet | **What you're buying** | Claiming Seat 3 buys that seat's pose. The sheet is the inventory, so the stronger claim is literally true. |

One label cannot make both promises honestly; `resolvePreviewEyebrow` picks per mode
(and `PREVIEW_EYEBROW` holds the two strings, so the copy has one home).

The resolution rules live in `src/features/commissions/lib/selection-preview.ts`
(`buildPreviewModel`), deliberately free of React so they are unit-testable.

**Artwork is never borrowed.** A chosen tier with no uploads falls through to
the *type's* gallery, never to another tier's examples: showing the wrong piece
next to a price misrepresents the sale. Unresolved media IDs (a deleted media
document arrives as a bare string in the free-form `pricingTiers` JSON) are
dropped rather than rendered as broken images.

## No frames, full-size artwork

Both panes are plain content on the page surface. The form used to sit in a
`CardBezel` (muted shell, 2px inset, border, shadow) and the pane boxed every
part of itself — the artwork, the thumbnails, the summary, the seat card. That
is a different visual language from the storefront's flat blocks, and on a
two-pane layout it read as two boxes floating in the page.

Removed: the form's bezel, the status cards' border and shadow, the artwork
frame, the thumbnail borders, the summary box, the seat card outline, the
dashed prompt and the chips' outlines. Kept: **form controls** (inputs, textarea,
select, the dropzone and the seat markers) still carry a border — a borderless
text field is an invisible one, and the marker needs its contrast ring over
artwork. Type cards swapped an outlined box for a filled surface with a ring on
the selected one.

**Images take the full column at their own aspect ratio.** The artwork used to
sit in a fixed `aspect-[4/3]` box with `object-contain` and padding, which
letterboxed every portrait or landscape piece into a fraction of the width it
could have used. It now renders at `w-full` inside a frame whose `aspect-ratio`
comes from the media document's own width/height (`imageAspect`, falling back to
4:3 when a reference was never resolved). The seat sheet does the same — its
markers are positioned in % of a box that is now exactly the image, so they stay
pinned as the sheet grows.

There is **no height cap**. One was here while the pane held a third of the width;
at seven tenths it would shrink a portrait back down to a fraction of the column
it has, which is the opposite of the point. A tall piece simply makes the page
taller, and the pane stays pinned beside it.

### Column split: 60 / 40

```
xl+ ┌──────────────────────────────────┬─────────────────┐
    │  3fr (60%)                       │  2fr (40%)      │
    │  artwork                         │  questions      │
    └──────────────────────────────────┴─────────────────┘
```

The pane used to share the frame 50/50 (with the form column capped at 34rem).
The artwork now takes **three fifths**: it is what the customer is choosing, and
a commission's detail level is only legible at size. The questions keep **two
fifths** — they are not a sidebar, and at a third the type cards and tier selects
read as a squeezed column. `xl:grid-cols-[minmax(0,3fr)_minmax(0,2fr)]`
(60 : 40).

### The two page-level choices sit above both panes

"Which round?" and "Project Type" both render above the two-pane grid
(`max-w-3xl`), in that order, and the form column holds the application alone.

They are page-level decisions: the round decides *which commission types exist*,
and the type decides *which tier sections exist and which artwork the pane
shows*. At the top of the form they were the first rows to scroll out of sight,
so a customer editing a tier had to scroll back up to re-check what they were
buying.

Structurally this split `ProjectTierSelector` in two:

| Component | Renders | Lives |
| --- | --- | --- |
| `ProjectTypePicker` | the type cards (with thumbnails) | above both panes |
| `ProjectTierSelector` | the priced tier sections | inside the form |

The type picker still renders nothing below two types — one type is not a
choice — and stays two-across at most.

## The form's own page: `/apply`

The form is still a CMS block (`commissionForm`) — an artist who wants it
mid-page keeps it. It also has its own route, `src/app/(frontend)/(studio)/apply/page.tsx`,
which renders the same component with the form's data needs declared directly
(`getBlockDataNeeds([{ blockType: 'commissionForm' }])`).

Why both: on a composed page the form shares the frame with the hero above and
the gallery below, so the artwork column can never be as large as the screen
allows — which is the whole point of showing the tier you are buying. On its own
route it gets the viewport.

Wiring that has to stay true:

- the wrapper keeps `id="commission"`, so the `#commission` hash slot shortcuts
  append still lands on the form;
- `apply` is in `RESERVED_STUDIO_ROUTE_SLUGS`, because Next resolves a static
  route before `[slug]` — a CMS page saved as "apply" would be silently
  unreachable;
- `COMMISSION_FORM_PATH` (what a slot shortcut navigates to when the current
  page hosts no form) is `/apply`;
- platform hosts still 404 it, like every other single-segment studio route.

## Summary rows, not a fake total

The pane lists every chosen section in the artist's order with the price exactly
as authored — `"120"` formats to the studio currency, while `"+25%"`, `"15 - 40"`
and `"TBD"` survive verbatim (`formatPrice`). It does not sum anything: the
price list mixes flat amounts, surcharges and percentages of a base tier, so a
computed total would be a number the artist never agreed to. The note under the
rows says the artist confirms the final quote.

## Only a round that can take the request is offered

The submission action has always refused a round that is not `active` with
`acceptingCommissions` ("This commission round is no longer accepting
requests."). The *form* did not check, so a paused or planned round could be
listed, chosen and filled in, and the refusal only landed on submit. That is the
observed oddity: a round that was not open appeared in the commission form.

The list is a `any[]` prop from several page paths and crosses a `'use cache'`
slice with a minutes-long window, so it can legitimately be stale. `lib/round-options.ts`
now decides what may be offered:

- `status` present and not `active` → not offered (planned, archived);
- `acceptingCommissions === false` → not offered (paused intake keeps the round
  `active` while the work finishes);
- both absent → offered, because a minimal `{ id, name }` caller should not break
  — the server remains the gate for those.

Consequences, all pinned by `tests/int/round-options.int.spec.ts` and
`tests/int/commission-form-round-liveness.int.spec.ts`:

- a stale or paused round is never rendered as a choice, with or without other
  live rounds in the list;
- when a list arrived but nothing in it is live, the form closes and says
  **"This round has finished taking requests"** — not the artist-is-full copy,
  which is about the studio and would be a lie about a paused round;
- a studio with no rounds at all keeps the original closed copy, because there is
  no round to blame and "finished" would name one that never existed;
- the seat picker tells the two empty states apart: `getSeatsForRoundAction` now
  reports `roundLive`, so a shut round reads "finished taking requests — refresh"
  while a live round with no inventory reads "no seats set up yet".

## Seats — "where is my seat?"

The pane shows the artist's posted seat sheet with a marker per seat at their
`{x,y}` percentages. Markers are real buttons (named `"Seat 3 — available"`,
disabled for taken/held/closed) that write into the same `seatId` state as the
form's radio group — they are a shortcut, never a second source of truth.

**Selecting a seat answers the question the sheet exists for.** At fit-to-pane
scale a marker on a busy YCH sheet is easy to miss, so a claimed seat:

- zooms the sheet toward its marker (`scale(1.9)` with `transform-origin` at the
  marker's percentages — the sheet and its markers scale together, so positions
  stay exact and the frame clips the overflow), with a `Whole sheet` toggle back
  to the overview;
- spotlights it with a radial gradient that pushes the rest of the art back;
- enlarges the marker (ring + check), while the other markers recede to 45%
  opacity — still visible, still clickable, just not competing for attention;
- adds a caption chip: *"Seat 3 is highlighted"*.

Nothing is invented per seat: there is one seat sheet, and the seat's position on
it *is* its picture. The sheet is drawn once per breakpoint — in the form below
`xl` and in the pane at `xl`+, both using the same `SeatSheet` component, so a
phone customer gets the same markers and the same zoom-to-seat behaviour.

## The artist's seat manager is inline, not a modal

`SeatManagerPanel` expands **under the round card** whose `Seats` button was
pressed (the button reads `Hide seats` while open, with `aria-expanded`). It used
to be a portalled modal; a dialog is the wrong shape for this job — the artist
compares the seat sheet against the seat list, the round's capacity and the other
rounds while placing seats, and an overlay hid all of it and trapped focus.

Layout: two flex columns that stack under ~320px (no media query needed) — sheet
+ placement + note on the left, seat list + add-seat form on the right. The
click-on-the-sheet marker placement, `Place` / `Close` / `Delete` actions and the
`Seat map saved` / `Save note` flows are unchanged. `Delete` asks twice: the first
press arms the button (`Confirm delete`), the second removes the seat — inventory
deletion is irreversible and a single mis-aimed click used to be enough.

At the round's cap the add-seat form is replaced by a note naming the way
forward (raise the limit in **Edit**). It is replaced the same way when the round
is bound to no seat-based type: the inputs could only end in the server's
refusal.

## The mascot

The mascot's own desktop column used to occupy up to 440px at `2xl`, which is
the width the artwork needs. So the column yields at `xl`
(`hidden lg:flex xl:hidden`): `lg`–`xl` keeps today's mascot + form row, and at
`xl`+ the same state-driven message follows the form into the pane as a compact
note under the summary.

## Type cards

`ProjectTypePicker`'s cards carry the type's first artwork when the artist has
any (`resolveTypeThumbnail`: category gallery first, then a tier example). The
image is decorative (`alt=""`) — the card already names the type.

The picker is hoisted above the panes (see "The two page-level choices" above),
which is why the component is separate from `ProjectTierSelector`.

**Two across, maximum.** A third column squeezed each card to ~180px: the
thumbnail became a stamp and the label wrapped. The grid is `grid-cols-1
sm:grid-cols-2` regardless of how many types exist — it grows in rows.

**Selection and hover are inset rings** (`ring-2 ring-inset`). A ring is painted
outside the border box, and an ancestor that clips overflow made an outside ring
on the first card of a row read as a cut border. Inset rings live inside the card
and no ancestor can clip them. Hover is `ring-border` (neutral) rather than a
plum `hover:border-primary`: on a tinted surface that read as the card turning
red. The cards are a `radiogroup`/`radio` pair, like the round cards beside them,
so the chosen type is announced rather than merely coloured.

## State

The pane renders only while the form is answerable:

```ts
const showPreview = formOpen && !submitted && Boolean(user) && hasPublishedPricing
```

`formOpen` is `areCommissionsOpen && !roundAvailability.suppliedButNoneLive`, so a
form closed by a finished round hides the pane for the same reason a closed
studio does: there is nothing to preview.

Closed rounds, the submitted state, the signed-out `AuthGatePrompt` and the
"pricing coming soon" state keep a centred single column instead of leaving a
dead half-page beside a narrow card.

## Accessibility

- Every tier control stays in the form, in DOM order, so screen-reader order
  matches visual order. The seat sheet is the exception by layout: below `xl` the
  form shows its own radio group beside the inline sheet, and at `xl`+ the form
  hides that group because the sheet's markers *are* the seat control. Both paths
  write the same `seatId` state, so there is one selection and two ways to reach
  it, never two sources of truth.
- Thumbnails are `aria-pressed` buttons in a labelled group
  (`"Show example 1 of 3"`); the active one is visually ringed.
- Seat markers are `aria-pressed` buttons with the seat name and availability in
  the accessible name, and are disabled exactly where the form's radio is. Each
  marker also carries the seat's short label, so a sheet full of dots never asks
  a customer to click one to find out which seat it is.
- Decorative backdrop imagery is `alt="" aria-hidden`.

## Two layouts: inline block vs `/apply` page

`CommissionForm` takes `layout`:

- **`inline`** (default, the CMS block) — the form is one block among others, the
  page scrolls, and the pane rides along with `position: sticky`. A block that
  hijacked the page's scrolling would fight the hero above and the gallery below.
- **`wizard`** (`/apply`) — the same component, plus the two page-level questions
  become their own steps (below). The scroll model is identical: one document,
  with the artwork sticky beside it. `page` used to be a third value that owned
  the viewport with two inner scrollports; it was removed along with that model,
  which is why the type is now two states rather than three.

### Wizard mode: round → project → application

`/apply` asks three questions in order, one screen each:

```
┌──────────────────────────────────────────────┐
│ ① Round  ─  ② Project  ─  ③ Details           │  <- progress rail
│                                              │
│  Which round are you applying to?            │  <- the step's question
│  [ card ] [ card ]                           │
│                                              │
│                     [ Back ]  [ Continue ]   │
└──────────────────────────────────────────────┘
```

**Steps are dropped when there is nothing to ask** (`lib/wizard-steps.ts`):
`roundCount > 1` gets a round step, `typeCount > 1` gets a type step, `details`
always. A one-round one-type studio therefore opens straight onto the form — two
steps that only confirm the single option are a click tax. A stored index is
clamped (`clampStepIndex`) because switching rounds can remove the type step
under the customer's feet.

**This is what makes the flow scroll-free.** Steps 1–2 are a short card list by
construction — they cannot overflow. Step 3 is the two-pane application, and two
things keep its height down:

- the marketing heading ("Ready to Sona Up?") is dropped when the stepper is
  showing, because the step header already says what the screen is for;
- the round/type cards are not in the column at all — they were steps 1–2.

Honest limit: step 3 is the one screen that can still scroll — the document
scrolls, on any viewport, and the artwork stays pinned beside it (see the sticky
caveat below). On a tall desktop the form fits without scrolling; on a short
laptop, or with many reference images / a long seat list, the page scrolls while
the artwork holds still. Steps 1–2 are short enough that they never scroll at any
width.

Navigation: the rail only jumps **backwards** (a completed step is reachable, a
future one is inert), Back is present on every step including the last, and
Continue disappears on the last step because the application's own submit button
is the way forward. The stepper is deliberately not a `<form>` — nesting one would
let Continue submit the application.

### Page mode

`/apply` is an ordinary scrolling document at every width. What makes it the
better place to view the pane is not a different scroll model — it is that the
page has no hero above the form and no gallery below it, so the artwork column
gets the whole frame, and the sticky `top` offset is the only thing above it.

- The document is the only scrollport. Neither pane has `overflow-y-auto`, so
  there is no inner scrollbar to trap a flick or to hide the artwork.
- The artwork pane is `xl:self-start xl:sticky` with
  `top: calc(var(--header-stack-height, 6rem) + 1.5rem)`, so it holds its place
  while the questions scroll past.
- The route frame uses `overflow-x-clip`, **not** `overflow-x-hidden`: hiding one
  axis makes the other compute to `auto`, which turns the frame into the sticky
  pane's scroll ancestor and silently degrades `sticky` to `static`. That is a
  real regression this page shipped with once — the artwork scrolled away with
  the questions — and `tests/int/apply-page.int.spec.ts` now pins the class.

Below `xl` there is no second pane, so the same page is a single column and the
seat sheet is drawn inline in the form.

### Sticky caveat

The pane is `xl:sticky` with `top: calc(var(--header-stack-height, 6rem) + 1.5rem)`.
A `position: sticky` element cannot escape an ancestor with `overflow` other than
`visible`: on pages whose wrapper sets `overflow-x-hidden` (e.g. the `[slug]` page
frame) it degrades to a normal block and scrolls with the form. The home page has
no such ancestor, so the pane stays put there. `/apply` avoids the trap by using
`overflow-x-clip` on its frame — which contains horizontal overflow without
creating a scroll container — so this is the reliable place to view the pane.

## Verification

- `tests/int/selection-preview.int.spec.ts` — resolution rules, the
  never-borrow property, summary rows, authored-price passthrough.
- `tests/int/selection-preview-render.int.spec.ts` — no caption over the art
  (the tier is stated once, in the summary), thumbnail
  `aria-pressed`, placeholder, seat markers enabled/disabled, sold-out copy, and
  the zoom/transform-origin + "is highlighted" caption for a claimed seat.
- `tests/int/commission-form-preview-layout.int.spec.ts` — the responsive
  contract jsdom cannot see: `hidden xl:sticky … xl:block`, the two-column grid,
  the single-column answerable-state guard, the mascot yield, the one-sheet-per-
  breakpoint rule, type-card thumbnails.
- `tests/int/seat-manager-inline.int.spec.ts` — the admin seat manager renders
  inline under its round (no portal/overlay/dialog semantics, `aria-expanded` on
  the trigger, upcoming rounds get the same control), and the customer's inline sheet is
  the marked `SeatSheet`, not a bare image.
- `tests/int/round-options.int.spec.ts` — the liveness rule (planned/archived and
  paused rounds dropped, minimal `{ id, name }` kept, holes ignored, order kept)
  and the "supplied a list, nothing open" flag.
- `tests/int/commission-form-round-liveness.int.spec.ts` — the rendered form: no
  application form and no round name for a planned or paused round, the
  round-finished copy instead of the artist-is-full copy, only the live round
  when a live and a paused one are supplied, and the ordinary closed copy for a
  studio with no rounds. Mutation-checked: restoring the old unfiltered list fails
  it.
- `tests/int/apply-page.int.spec.ts` — the `/apply` wiring: the route renders the
  real form, keeps `id="commission"`, `apply` is a reserved slug,
  `COMMISSION_FORM_PATH` points there, and — the regression guard — the route
  frame contains horizontal overflow with `overflow-x-clip`, never
  `overflow-x-hidden`, because the hidden axis would make the frame the sticky
  pane's scroll ancestor and un-stick the artwork.
- `tests/int/apply-split-scroll.int.spec.ts` — scroll model: one document, no
  inner `overflow-y-auto`/`overscroll-contain` on either pane, `xl:sticky` on the
  artwork, and no viewport-height trap on the route. (This file previously
  described two independent scrollports and asserted the opposite of the code; it
  now pins what ships.)
- `tests/int/commission-wizard.int.spec.ts` — the wizard as a customer meets it,
  including the two share-link cases: a `?round=` link skips the round step, and
  the **Change round** escape is present on every skipped step so a link to the
  wrong campaign is reversible.
- `tests/int/seat-capacity.int.spec.ts` — the cap policy, `resolveSeatLimit`
  parsing, `changesSlotLimit` (so a resubmitted, unchanged limit never triggers
  the guards), and both write paths at source level: `createSeat` refuses past
  the limit, `updateRound` refuses a lowering below the existing seats.
- `tests/int/selection-preview-render.int.spec.ts` — every seat marker carries
  the seat's short label, so a busy sheet is never a field of anonymous dots.
- `tests/int/commission-form-mobile-borderless.int.spec.ts` — updated: the status
  cards now assert **no** border class at all (not `border-0` on mobile +
  `sm:border`), since the form no longer renders a card frame.
- `npx tsc --noEmit` clean; the existing commission suites (`commission-form-*`,
  `commission-rounds`, `dynamic-pricing-tiers`, `commission-mobile-sticky-dock`)
  stay green.

Not verified in a browser: the visual result at 1024/1280/1536px still needs a
look on a dev server (the sticky behaviour in particular — no source-level
assertion can prove it). No schema change is involved in this feature — seats
store only their marker.
