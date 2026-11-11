# Multi-Round Commissions & Seat Maps

## What this is

A studio can run several commission campaigns at the same time — a YCH batch,
an acrylic special, a general illustration opening — with each campaign selling
its own commission types and owning its own capacity.

```text
Studio
├─ Commission Settings            ← the type + price library
│   ├─ illustration  (open)       drawing tiers, backgrounds, extras
│   ├─ refsheet      (open)       custom tiers, extras
│   ├─ ych           (seats)      add-on categories only
│   └─ acrylic       (open)       sizes, finishes
│
└─ Commission Rounds              ← many may be live at once
    ├─ General Illustrations → [illustration]  maxSlots 10
    ├─ YCH: Autumn Batch     → [ych]           seats: 5 (seat map image)
    └─ Acrylic Special       → [acrylic]       maxSlots 3
```

The customer picks the round in the form when more than one is live. The
selection travels as `?round=<id>` from the price list's “Request a slot”
shortcut. The server never guesses: a submission names its round, and the round
must be live, belong to that studio, and offer the submitted type.

## Commission type templates

A studio's type library is built from **starter templates**
(`src/features/studio-settings/lib/commission-type-templates.ts`). A template is
app-level content, versioned with the code and never edited by an artist;
creating a type copies its sections and default prices into the studio's own
library, where everything is editable afterwards.

| Template | Slot style | Sections |
| --- | --- | --- |
| Illustration | open | Drawing Tier (cards), Backgrounds (panel), Extras (guide only) |
| Custom & Ref Sheet | open | Ref Sheet Tier (showcase), Extras (guide only) |
| YCH | **seats** | Add-ons (compact) — the seat carries the base price |
| Emotes & Badges | open | Emote Packs (compact) |
| VTuber & Live2D | open | Model Art (showcase) |
| Acrylic & Traditional | open | Size (cards), Finish (panel) |

`slotMode` is the part a name cannot convey: YCH is the seat-map template, so a
type created from it sells round seats rather than `maxSlots` capacity.

### Two ways to create a type

- **Rounds → Plan a Round → “Start from a template”.** Persists the type
  immediately (`createCommissionTypeFromTemplate`), then selects it for the
  round. This is the round-first setup: name the campaign, click YCH, open it,
  add seats. Idempotent — if the studio already has that type (same key, or the
  same label from the old timestamped presets) it is reused instead of
  duplicated.
- **Types & Pricing → Add Type.** The same templates, appended to the local
  editor and written on Save. This flow deliberately allows adding the same
  template twice (two illustration price lists), hence its timestamped keys.

A type created in the round drawer is also pushed into the settings editor's
local library, so a later “Save” of unrelated settings cannot write a type list
that drops it.

A round's type keys are validated against the **saved** settings document, which
is why the drawer's template path persists immediately — a type that only lived
in the settings form would fail with “Commission types not found in your saved
settings”.

## Data model

| Where | Field | Meaning |
| --- | --- | --- |
| `sprints` | `commissionTypeKeys` (`text`, hasMany) | Types this round sells. Empty = every type (legacy). |
| `sprints` | `seatMapImage` (upload → media) | The sheet customers pick seats from. |
| `sprints` | `seatMapNote` | Instructions above the seat picker. |
| `commission-settings.commissionTypes[]` | `slotMode` (`open` \| `seats`) | How this type sells capacity. |
| `commissions` | `commissionType` (`text`) | Was an enum (`illustration`/`custom`); studios define their own keys now. |
| `commissions` | `seat` (relationship) | The claimed seat. |
| `commissions` | `seatSnapshot` (group) | `label` and `price` copied at claim time. |
| `commission-seats` | — | One row per seat (one character): round, type key, label, price, note, marker %, status, claim token, held-until, commission. |

### Seat lifecycle

```text
open ──claim(conditional write)──▶ held ──finalize(token)──▶ taken
 ▲                                    │                        │
 └──── release (token / commission) ──┴────────────────────────┘
```

- **Claim** is one SQL `UPDATE ... WHERE status = 'open' OR (status = 'held' AND
  held_until < now)`. Two customers racing for one seat produce exactly one
  winner; the loser gets `409` and the form reloads the map.
- The hold expires after 10 minutes, so a crash between claim and finalize
  self-heals.
- `finalize` runs *before* the commission is created; a failure after that
  releases the seat by its claim token.
- `releaseRoundSlotForCommission` (artist decline, customer withdrawal) puts the
  seat back on the map and releases the customer's per-round allowance. It does
  **not** touch the `maxSlots` counter, which a seat round never reserved.
- A `taken` seat can only be freed through its commission. The seat manager can
  close/reopen `open`/`closed` seats, but deleting sold seats is refused.

### Round invariants

- A round needs at least one type to go live.
- Multiple rounds may be live concurrently, including rounds offering the same commission types.
- The type set is locked while live; name and opening date are locked too.
- A round may not mix `open` and `seats` types — capacity would be half a
  counter and half inventory, and neither could tell the truth.
- Dates may overlap.

## Submission flow

`src/features/commissions/actions/submit-commission.action.ts`:

1. Resolve tenant + authenticated user.
2. Resolve the round: `roundId` must be present and live; when absent, exactly
   one live round is required (compatibility for a cached form). Several live
   rounds without a choice → “choose which round you are applying to”.
3. Validate the type is offered by that round.
4. Read the studio type catalog to learn the type's `slotMode` (fail closed).
5. Seat rounds: claim the seat (`round` + `type` + `open` in the WHERE).
6. Reserve the customer's per-round allowance.
7. Open-slot rounds: reserve the round counter (`maxSlots`).
8. Finalize the seat (seat rounds) before slow work.
9. Ingest media, compute the price breakdown (seat price + selected add-ons for
   seats; tier analysis otherwise), create the commission with `batch: round.id`
   and either `pending_ai` (open) or `needs_artist_review` (seat).
10. Attach the seat back-link, send the confirmation email.
11. Any failure releases the seat, the customer allowance and the round counter.

Payment: the Stripe webhook attaches the queue item to `commission.batch` — the
round the customer chose — never to “whichever round is accepting now”.

## Public UX

- **Open-rounds showcase** (`OpenRoundsCarousel`): **one card per commission
  type within a live round**, not one card per round. Illustration + Acrylic in
  one round is two offers at two price lists, so it renders as two cards — each
  with its own artwork (a seat type shows the round's sheet, an open type shows
  its first category image), the round name as context above the type name, its
  own availability, and its own CTA (`?round=<id>&commissionType=<key>`) that
  opens the form already set to that round and type. The artwork opens a
  full-screen **Preview**. Motion is a marquee — see below.
- **A block, not a page section.** `openRounds` (`src/blocks/OpenRounds.ts`) can
  be dropped on the home page, `/prices` or any custom page. It has exactly two
  fields — **title** and **subtitle** — because presentation is derived, not
  configured: card width follows the viewport (one on phones, two from 768px),
  and the row only moves on its own when the cards overflow. `/prices` renders the
  same component above its price list (not via the block), so the page always
  leads with it.
- **Motion, and the rule that keeps it honest.** A seamless marquee needs two
  identical passes translated by half the track, so the second pass must start
  off-screen or the same campaign appears twice. `resolveRoundRowLayout`
  (`src/features/pricing/lib/marquee.ts`, pure) therefore requires **both**:
  - at least `MARQUEE_MIN_ROUNDS` (3) rounds — one or two are a set you take in at
    a glance, and sliding them is noise;
  - a card pass wider than the row — the measured check that keeps the repeat
    off-screen.

  Cards are 420px wide in marquee mode precisely so three of them (1320px) exceed
  the widest container the block can sit in (`7xl` → 1232px usable). If an artist
  raises the container cap through the theme lever until three cards fit, the row
  falls back to still rather than showing a duplicate. Two rounds never move, at
  any width. `prefers-reduced-motion` stops the loop and lets the row scroll by
  hand.
- **No pause button; the row pauses itself.** The motion stops while the pointer
  is over the row and while anything inside it has focus (`:focus-within`), so a
  card is never a moving target for a click or a Tab. `prefers-reduced-motion`
  stops it outright, hides the duplicate copy, and lets the row scroll by hand.
- **Card imagery** (`buildOpenRoundCards`, pure): a seat round shows its sheet;
  an open round borrows the first artwork of the categories it sells; a bound
  round with no imagery of its own falls back to a text-only treatment rather
  than borrowing another campaign's art (a legacy unbound round may borrow
  anything, since it sells everything). `toShowcaseRounds` joins live rounds with
  their seat inventory, shared by the block and the page.
- **Form** (`CommissionForm`): a round switcher appears only when ≥2 rounds are
  live; switching rounds clears the type and every tier/seat selection. A
  seat-based type renders the seat map (image + note) and numbered seats; taken
  seats are disabled. The submit payload carries `roundId` and `seatId`.
- **Price list** (`PriceList`): each commission type's “Request a slot” CTA
  carries the live round that sells it (`?round=<id>&commissionType=<key>`), so
  the form opens on the right campaign.
- **Queue** (`QueueView`): round tabs (when >1), a round badge on every card, and
  drag-and-drop that preserves the card's round.

## Admin UX

- **Commission Settings → Types & Pricing**: each type has a “How this type is
  sold” select (`Open slots` / `Seat map`). Deleting a type bound to a live or
  planned round is refused with the round's name.
- **Commission Settings → Rounds**:
  - Plan/Edit drawer: type checkboxes (at least one), limits, dates. Types are
    read-only while the round is live.
  - Round cards: type chips, `Slots: n/max` or `Seats: taken/total`, per-round
    Close / Reopen / Archive.
  - **Seats** button (seat rounds): upload/replace the seat map image, save the
    instructions, then click the sheet to place each seat. A seat is **one
    character** — label, price and an optional note. Placement is click-only:
    markers are stored as percentages of the sheet, so the map looks right at
    any width. Close/open/delete from the seat list.
- Several rounds may be live; opening one no longer archives another.

## Files changed

- `src/blocks/OpenRounds.ts` (the `openRounds` block — registered in
  `Pages.blocks` and classified in `block-data-needs.ts` as
  `['pricing','activeSprints','seatRounds']`)
- `src/features/pricing/lib/open-round-cards.ts` (pure card builder +
  `toShowcaseRounds`), `lib/marquee.ts` (loop geometry),
  `components/OpenRoundsCarousel.tsx`
- `src/lib/blocks/page-data.ts` (the `seatRounds` need)
- `src/collections/Sprints.ts`, `src/collections/CommissionSeats.ts`,
  `src/collections/Commissions.ts`, `src/collections/CommissionSettings.ts`
- `src/collections/hooks/enforceRoundTypeExclusivity.ts` (replaces
  `enforceSingleActiveSprint.ts`)
- `src/features/queue/lib/round-rules.ts`, `src/features/queue/lib/round-edit.ts`
- `src/features/queue/actions/` — `get-sprint`, `get-live-rounds` (inside
  `get-sprint`), `commission-seat`, `manage-seats`, `get-seats-for-round`,
  `get-round-type-catalog`, `create-round`, `update-round`, `open-round`,
  `get-round-stats`, `release-round-slot`, `queue-actions`
- `src/features/studio-settings/lib/commission-type-templates.ts` (template
  catalog + pure build/reuse/merge helpers),
  `src/features/studio-settings/actions/create-commission-type.action.ts`
  (persist a template as a studio type)
- `src/components/admin/commission-settings/default-preset-types.ts` (editor's
  view of the catalog), `PlanRoundDrawer.tsx` (template picker)
- `src/features/commissions/actions/submit-commission.action.ts`,
  `src/features/commissions/components/CommissionForm.tsx`,
  `src/features/commissions/lib/commission-form-submit.ts`,
  `src/features/commissions/lib/commission-sections.ts`,
  `src/features/pricing/lib/slot-request.ts`,
  `src/features/pricing/components/PriceList.tsx`
- `src/components/BlocksRenderer.tsx`,
  `src/components/admin/CommissionRoundsPanel.tsx`,
  `src/components/admin/CommissionSettingsAdminView.tsx`,
  `src/components/admin/commission-settings/{PlanRoundDrawer,SeatManagerPanel,CommissionTypesTab,types,settings-helpers}.tsx`
- `src/features/queue/components/{QueueView,KanbanCard}.tsx`
- `src/lib/stripe/stripeWebhooks.ts` (queue item follows `commission.batch`)
- `src/migrations/20261106_112627.{ts,json}`

## Migration notes

- `commissions.commission_type` is converted from the `illustration`/`custom`
  enum to `varchar`, preserving values.
- `sprints_texts` backs the hasMany `commissionTypeKeys`.
- `commission_seats` is a new collection with FKs to studios, sprints and
  commissions; the seat/circular commission FK is added after both columns exist.
- Existing rounds get an empty binding → they keep “all types” semantics. They
  must be bound to at least one type before they can be opened again.
- A single migration, `20261106_112627`, carries everything: rounds/seats,
  commission-type changes, and the additive `pages_blocks_open_rounds` block
  table (generated by `payload migrate:create`).
- `invites.email`/`status`/`expires_at` lose their `NOT NULL` in that migration
  on purpose: those fields carry `admin.condition` (the create view hides them
  behind the composer) and Payload's generator emits conditional fields as
  nullable columns. Presence is enforced by the invites collection's own hooks,
  and NULL invites fail closed in every consumer.
- `commissions.commission_type` conversion order matters: the enum default is
  dropped *before* `SET DATA TYPE varchar`, and `down()` drops the seat FKs
  *before* `DROP TABLE … CASCADE` (CASCADE removes them, so a later
  `DROP CONSTRAINT` would fail). `down()` also remaps `ych`/`acrylic` → `custom`
  so the narrowing cast cannot abort.

## Verification

- `npx tsc --noEmit`
- `npx vitest run --config ./vitest.config.mts --project=unit tests/int/commission-type-templates.int.spec.ts tests/int/commission-rounds.int.spec.ts tests/int/commission-status-toggle.int.spec.ts tests/int/round-slot-accounting.int.spec.ts tests/int/slot-request.int.spec.ts tests/int/commission-form-submit.int.spec.ts`
- DB suites (require Postgres): `round-slot-accounting-db`, `toggle-commissions-db`,
  `customer-close-slot-db`.
- Manual: two live rounds → home form shows the switcher → pick YCH → seat map
  renders → claim a seat → queue shows the round badge → decline the commission
  → seat reopens.

## Known gaps

- Per-round price overrides are not supported; prices live on the type library.
- `estimateCommission` still applies the illustration complexity rubric to any
  non-`custom` open-slot type. Seat rounds skip AI entirely.
- The customer dashboard type filter dropdown lists only legacy types (URL
  filtering works).
- Seat map markers are numeric percentages; there is no click-to-place editor on
  the image yet.
- Mixed slot styles inside one round are refused by design; split them into two
  rounds.
