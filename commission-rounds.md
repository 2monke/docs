# Commission Rounds — Type Binding, Slot Accounting & Seats

## Why this exists

A round (collection `sprints`, artist-facing name “Commission Round”) is one
campaign: a YCH batch, an acrylic special, a general illustration opening. A
studio may run several at once, so a round declares **which commission types it
sells** (`commissionTypeKeys`) and the customer (or the price-list shortcut)
names the round they are applying to.

A round has capacity in one of two styles, decided by its types:

- **Open slots** — `maxSlots` is the cap; a slot is held from submission.
- **Seat map** — the round owns named seats (`commission-seats`), each with its
  own price. **The slot limit is the seat cap**: an artist who set 4 slots can
  create 4 seats, and the 5th is refused until they raise the limit. Existing
  rounds that already hold more seats are left alone — they just cannot add
  another (`resolveSeatCapacity` in `src/features/queue/lib/round-rules.ts`).
  `updateRound` mirrors it: a limit below the seats that already exist is
  refused, so the invariant cannot be broken from either direction.

The public submission gate, the per-customer limit, the admin rounds panel and
the max-slot guard must all agree on **how many slots are taken** — and, for a
seat round, on **which seats are still open**. They used to disagree:

- The gate counted **commissions** with `batch = round` and `status != rejected`.
- The panel counted **queue rows** with `sprint = round`. A queue row only
  exists once a commission is paid, so a round holding “2 paid + 1 pending AI +
  2 withdrawn” displayed **1/10** while the gate saw 3.
- `closed_by_customer` never released its reservation, so withdrawals
  permanently ate capacity.

## Round invariants

`src/collections/hooks/enforceRoundTypeExclusivity.ts` replaces the old
“one active round per tenant” rule:

1. A round can only become live (`active` + `acceptingCommissions`) with **at
   least one** bound type.
2. Several rounds may be live concurrently, including multiple batches for the same
   commission types (e.g. Batch #1 and Batch #2). Customers choose which round they
   are applying to.
3. The type binding is **locked while the round is live**. Name and opening date
   are locked too (`round-edit.ts`); pause the round to change them.
4. An empty binding means “every type” — that is what rounds created before type
   binding already mean, so it must keep working.
5. Round dates may overlap. Overlapping campaigns are the point of multi-round
   support; the old “dates clash” rejection is gone.

The pure rules live in `src/features/queue/lib/round-rules.ts`
(`findLiveTypeConflict`, `validateRoundTypeKeys`, `isTypeOfferedByRound`,
`splitRoundTypeModes`, `planToggleOpen`) and are unit-tested without a database.

## Slot accounting (open-slot rounds)

`src/features/queue/lib/round-rules.ts`:

- A slot is **held** from submission (`pending_ai`) through review, payment and
  delivery (`completed`).
- A slot is **released** only by `rejected` (artist declined) or
  `closed_by_customer` (customer withdrew).

```ts
ROUND_SLOT_RELEASING_STATUSES = ['rejected', 'closed_by_customer']
buildRoundSlotWhere(roundId)          // commissions occupying a slot
buildCustomerRoundSlotWhere(roundId, customerId) // one customer's slots
```

Anything that measures round capacity must go through these builders — never
through queue rows. `countQueueItemsForSprint` was deleted for that reason.

## Seats (seat-map rounds)

A seat is inventory, not a tier. See `docs/multi-round-commissions.md` for the
full flow; the accounting contract is:

- Submission claims a seat with a **conditional write** (`WHERE status = 'open'`
  or an expired hold). Zero rows updated means another customer won; the API
  answers `409` and the form refreshes the map.
- The seat is finalized before the commission is created; any failure releases
  it by claim token.
- A seat round does **not** reserve `sprint:slots:<id>` — the seat was its slot.
  The per-customer allowance is still reserved and released.
- **Seat inventory is capped by the round's slot limit.** Creating a seat past
  `maxSlots` is refused (`resolveSeatCapacity`), and lowering the limit below
  the seats that already exist is refused by `updateRound`. At the cap the seat
  manager shows no add-seat form at all — the label/price/notes inputs are
  replaced by a note saying to raise the limit in **Edit** — so the artist never
  fills in a seat the server would reject. Management stays available: closing
  or deleting a seat is how room is made again. A round that
  predates the rule and holds more seats than its limit keeps them; it just
  cannot grow until the artist raises the limit.
- `releaseRoundSlotForCommission` frees the seat (decline or withdrawal) and the
  customer allowance. `taken` seats are never freed by an admin status edit —
  only by the commission's own release path.

## Counters and consumers

`src/features/queue/actions/count-round-commissions.action.ts` (internal module,
**not** a `'use server'` action):

| Function | Meaning | Consumers |
| --- | --- | --- |
| `countRoundSlotCommissions` | Commissions holding a slot | gate, panel, max-slot guard |
| `countCustomerRoundSlotCommissions` | One customer’s slots | `maxRequestsPerCustomer` |
| `countRoundCommissions` | Every attached commission | delete guard, archive history |

`getRoundsWithStats` enriches each round with `slotCount` (panel “Slots: n/max”),
`seatTotal`/`seatOpen`/`seatTaken` (panel “Seats: n/m”), `commissionCount` (only
counted for archived rounds) and `openItemCount` (unfinished queue items).

## Release paths

`src/features/queue/actions/release-round-slot.action.ts` —
`releaseRoundSlotForCommission(commission, options)` frees the round counter (open
rounds), the seat (seat rounds) and the customer allowance. Called by:

- `closeCommissionSlot` — customer withdraws.
- `rejectCommission` — artist declines.

Both transitions are **conditional writes** (`status: { in: [reviewable
statuses] }`). The write is the claim: a second decline, a retry, or two
concurrent closes only match once, and only the winner releases. Releasing
unconditionally after a status update would decrement the Redis counter once
per call for a single slot and let the round accept more than `maxSlots`.

Both also use `throwOnError: true` and check the returned document — a swallowed
write must never free a slot.

## Failure modes

- `countCommissions` returns `0` on error by default (display use). The
  reservation gate passes `throwOnError: true` and **fails closed**: if the
  count cannot be read, the submission is refused with a retry message instead
  of being read as “the round is empty”.
- `getLiveRoundsForTenant` fails closed when asked (`throwOnError: true`); the
  round-invariant hook uses it, the storefront accepts “closed” on an outage.
- With Upstash configured, the round counter lives at `sprint:slots:<id>`
  (`sprint:customer:<id>:<customerId>` per customer). A missing key re-syncs
  from the DB count, which already excludes released statuses.
- The quick studio-wide toggle pauses **every** live round and resumes **every**
  paused one (`planToggleOpen`). It never creates a typeless round; with nothing
  to open it returns a message pointing at the Rounds panel.

## Queue

- A card carries its round badge, and the board groups by round (`sprint`).
- Dragging a card between status columns **keeps its round**. In the “All
  rounds” view a backlog card cannot be dropped into a status column (the round
  is ambiguous) — pick a round tab first.
- Changing a commission's round is an explicit edit, not a drag side effect.

## Files

- `src/features/queue/lib/round-rules.ts` — rules, type binding, toggle plan.
- `src/collections/hooks/enforceRoundTypeExclusivity.ts` — live-round invariants.
- `src/features/queue/actions/count-round-commissions.action.ts` — counters.
- `src/features/queue/actions/release-round-slot.action.ts` — release helper.
- `src/features/queue/actions/reserve-slot.action.ts` — Upstash gate + DB fallback.
- `src/features/queue/actions/get-round-stats.action.ts` — panel data.
- `src/features/queue/actions/commission-seat.action.ts` — seat claim/finalize/release.
- `src/features/queue/actions/manage-seats.action.ts` — artist seat CRUD.
- `src/features/commissions/actions/commission-slot.action.ts` — customer close.
- `src/features/commissions/actions/commission-review.action.ts` — artist decline.
- `src/components/admin/CommissionRoundsPanel.tsx` / `commission-settings/PlanRoundDrawer.tsx` / `commission-settings/SeatManagerPanel.tsx` (the seat manager expands inline under its round; the modal-era `SeatManagerDrawer.tsx` is gone).

## Tests

- `tests/int/commission-rounds.int.spec.ts` — type binding, exclusivity, slots.
- `tests/int/round-slot-accounting.int.spec.ts` — rule, panel mapping, fail-closed gate.
- `tests/int/round-slot-accounting-db.int.spec.ts` — real Postgres: 5 attached → 3 slots.
- `tests/int/customer-close-slot-db.int.spec.ts` — real action + DB: round full → withdraw → gate accepts.
- `tests/int/release-round-slot.int.spec.ts` — release wiring, status guard, lost-race guard.
- `tests/int/commission-rounds.int.spec.ts` — Redis/DB reservation behaviour.
- `tests/int/commission-status-toggle.int.spec.ts` — toggle planning (unit) + switch UI.
- `tests/int/toggle-commissions-db.int.spec.ts` — toggle against real rounds (multi-round resume/pause).
- `tests/int/commission-rounds-modal-dismiss.int.spec.ts` — drawer dismissal rules.

## Known gaps

- Admin-side status edits (Payload edit view) to `rejected` / `closed_by_customer`
  do not release the Upstash counter; the DB count is correct, but a Redis
  counter can stay inflated until its 24 h TTL or the next re-sync.
- The archive/open guards still count unfinished **queue items**, so a round can
  be archived while a `pending_ai` request is attached.
- The customer dashboard's type filter still lists only the legacy
  `illustration` / `custom` options; a URL `?type=ych` filters correctly, but
  the dropdown does not offer studio-defined types yet.
