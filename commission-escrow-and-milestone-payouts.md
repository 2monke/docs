# Commission Escrow & Milestone Payouts (SPEC)

> **Status**: DESIGN — decisions confirmed 2026-11-05, not yet implemented.
> **Decisions locked by the user**:
> 1. Hold **100%** of the commission on the platform until approval milestones.
> 2. Release a **deposit after sketch approval**, the **balance after final approval**
>    (do both in one build; a single-release studio config is just `depositPercent: 0`).
> 3. **No auto-approve.** Commissions can run for months; a clock must never
>    release money without a human decision.
> 4. **Admins can always force-release or refund** from the panel.
> **Related**: `docs/stripe-connect-destination-charges.md` (today's rail),
> `docs/commission-rounds.md` (slot accounting — unchanged by this work),
> `docs/monetization-listing-gates-and-subscriptions.md` (platform fee).

---

## 1. Why the current rail cannot do this

Commissions today are **destination charges**:
`payment_intent_data.transfer_data.destination = artist.stripeAccountId`
(`src/features/commissions/lib/checkout.ts` → `buildCommissionCheckoutParams`).

The artist's Stripe balance is credited **at the moment the customer pays**. There
is no "deliver later" flag on `transfer_data`. The three ways to delay artist
access, and why two of them are rejected:

- **Delayed payouts on the artist's account.** The money still lands in the
  artist's own balance and is spendable on their Express card. If they spend it,
  a later reversal fails and the platform eats the refund. Slows the bank
  transfer, does not protect the customer. Rejected.
- **Manual capture** (`payment_intent_data.capture_method: 'manual'`, supported
  by Checkout). Online card authorizations expire in **~7 days** (Visa card-not-present
  is 7 for customer-initiated; the card-present 5-day window does not apply here).
  Klarna allows ~28 days, PayPal ~20. A commission takes weeks to months. Rejected —
  but this *is* the right tool if a small paid "reservation deposit" is ever added,
  because a hold is cheaper than a charge + refund.
- **Separate charges and transfers.** Charge on the **platform** account with a
  `transfer_group`; create the transfer to the artist only when a milestone is
  approved, using `source_transaction: chargeId`. This is the pattern Stripe
  documents for paying a seller after the service is delivered. **Chosen.**

The merchant of record is **changing**, and that is a deliberate part of this
work: today the platform is the seller of record (destination charges without
`on_behalf_of`); under this spec the artist becomes the settlement merchant and
tax-liable party, while the platform is liable only for its own fee. See §2.1 —
it is a supported Stripe configuration, but it is not free. Dispute liability and
Stripe's processing fees stay with the platform either way.

---

## 2. Money model

The commission price is captured once, up front, on the platform. Nothing is
transferred until an approval milestone fires. **The artist's share depends on who
is tax-liable**, so the split is computed per studio mode rather than fixed.

```
captured total   = price + tax        (Stripe Tax, tax_behavior: 'exclusive')
platform fee     = resolveStudioPlatformFee(price, studio)   // src/lib/stripe/fee-schedule.ts

taxLiability = 'artist'   (the decision — artist is seller of record)
  artist net     = captured total − platform fee       // tax belongs to the artist to remit

taxLiability = 'platform' (fallback)
  artist net     = price − platform fee

sketch release   = round(artist net × depositPercent / 100)
final release    = artist net − sketch release        // remainder, no drift
```

**Why the tax moves with the liability.** The party that remits the tax must hold
the tax. Artist-liable means the artist is the seller: the customer's €63 VAT is
the artist's to remit, so it must be in the artist's payout. Leaving it on the
platform would make the artist pay it out of pocket. The inverse holds for
platform-liable. Getting this backwards is the whole failure mode of a
half-migrated tax model.

Worked example — €300 commission, 5% launch fee, 50% deposit, no tax configured:

```
price                30000 c
platform fee          1500 c      (5%)
artist net           28500 c
sketch release       14250 c
final release        14250 c
```

Same commission with €63 VAT, artist-liable:

```
captured total       36300 c
platform fee          1500 c      (5% of pre-tax price)
artist net           34800 c      → sketch 17400 c + final 17400 c
artist remits         6300 c      to the tax authority
```

Same commission with €63 VAT, platform-liable:

```
artist net           28500 c      → sketch 14250 c + final 14250 c
platform holds        6300 c      tax + 1500 c fee, remits the tax itself
```

Invariants that must never break: **the sum of all transfers equals `artist net`,
never the captured total**, and **`artist net` uses the tax mode persisted on the
commission**, not the studio's current setting.

### 2.1 Who owes the tax

**Decision: the artist is tax-liable; the platform is liable only for its own
fee.** This is a supported Stripe configuration, not a contract-only arrangement —
but it is a *different* configuration from what ships today, and it has strings.

**The mechanism.** Set the connected account as the settlement merchant:

```ts
payment_intent_data: {
  on_behalf_of: studio.stripeAccountId,
  transfer_group: `commission_${commission.id}`,
},
automatic_tax: {
  enabled: true,
  liability: { type: 'account', account: studio.stripeAccountId },
},
```

Stripe's merchant-of-record table lists *separate charges and transfers **with**
`on_behalf_of`* as a connected-account-MoR arrangement. Escrow therefore survives:
the charge still lands on the platform balance, the transfer to the artist is
still deferred until approval, but the artist is the seller of record.

**What each party owes under this model.**

- **Artist:** VAT/sales tax on the artwork, and income tax on the transfer.
  Transfers are gross earnings — the platform does not withhold.
- **Platform:** tax on its own fee. If the artist is an EU/UK VAT-registered
  business, the fee is B2B and reverse charge may apply; if the artist is a
  private individual, the platform charges VAT on the fee where it is registered.
  This requires knowing the artist's tax status, so onboarding must collect
  country, business-vs-individual, and VAT ID — one more step in the studio setup
  tour, and a field on `Studios` (`taxId`, `taxCountry`, `taxStatus`).

**Four things that break the clean story.**

1. **Stripe Tax reads the *artist's* registrations.** `automatic_tax` with no
   registration configured computes **zero**. Most individual artists have no
   registrations and never will. So "the artist pays the tax" in practice means
   "the tax is the artist's unfilled obligation", not "the tax is collected and
   remitted." That may be legally fine — many small creators sit under thresholds
   — but it is a deliberate outcome you are choosing, not a neutral one.
2. **You must collect registrations to compute their tax.** Stripe requires
   platforms to gather connected accounts' registrations (there is a Connect
   embedded component for exactly this). That is a real onboarding surface:
   per-jurisdiction registration entry, kept current. Without it, `automatic_tax`
   on the artist's behalf is decoration.
3. **`automatic_tax.liability: account` is documented for subscriptions only.**
   Every Stripe doc path that attributes tax liability to a connected account uses
   `mode: 'subscription'`; the one-time-payment (`mode: 'payment'`) examples only
   show `type: 'self'`. Whether the parameter is accepted on a one-time Checkout
   Session is **unverified** — sandbox check §10, before committing to this model.
   If it is rejected, the options are platform-liable tax (contradicts the
   decision), direct charges (kills the escrow hold), or invoicing the commission
   as a separate object (new surface, and the customer flow changes).
4. **Marketplace-facilitator laws can override the contract.** Several
   jurisdictions make the *platform* the collector regardless of who the parties
   agree is the seller, and you cannot contract out of a statutory duty. For
   commissions (services, not goods) this is jurisdiction-by-jurisdiction; it is a
   question for the accountant, with the specific list of countries your artists
   sell into.
5. **`on_behalf_of` has country and currency rules.** The connected account must
   be in a Stripe-supported country for the combination, and settlement happens in
   **the artist's** settlement currency — a EUR charge settled to a USD account
   incurs conversion on every commission. Check the artist's country support at
   onboarding, not at checkout, and decide whether to price commissions in the
   artist's currency or accept the FX.

**Unchanged:** the platform pays Stripe's processing fees and is ultimately liable
for chargebacks and dispute rates, `on_behalf_of` or not. That is why the fee floor
(`PROCESSING_FLOOR_RATE`) exists, and it still must.

**Keep it switchable.** `CommissionSettings.taxLiability: 'artist' | 'platform'`,
default `'artist'`. The platform-liable branch is the fallback if the sandbox check
fails or if a studio needs it, and it makes the difference testable rather than
assumed.

`depositPercent` lives on `CommissionSettings` (per studio), default **50**,
range **0–90**. `0` means a single release at final approval; `90` caps the
sketch payout so the final transfer is never zero. A value above 90 is clamped,
not rejected, so a studio edit can never strand the balance.

Exception path: if `artist net` is smaller than the fee floor (a €0.50 charge at
a 3.5% floor), the fee is capped at the amount and the artist net can be 0. The
release code must treat "artist net = 0" as a completed-with-nothing-to-transfer
milestone, not an error.

---

## 3. State machine

Commission workflow statuses do **not** change. Adding a `delivered` status would
ripple through `QUOTE_VIEW_STATUSES`, `CUSTOMER_PAYABLE_STATUSES`, slot rules,
notification copy and the admin panel; the milestone record carries that state
instead.

```
pending_ai → needs_artist_review → approved → awaiting_payment
                                                   │
                                    checkout.session.completed (payment_status = paid)
                                                   ▼
                                                  paid        payout: held
                                                   │
                          ┌────────────────────────┼─────────────────────────┐
                          ▼                        ▼                         ▼
                 sketch submitted          final submitted            admin refund
                          │                        │                         │
                 customer approves        customer approves          payout: refunded
                          ▼                        ▼                    (no status change)
                 payout: partially_released   completed
                   (sketch transfer)          payout: released
                                                    │
                                          changes requested → back to paid
                                          (no money moves)
```

Milestone status (`commission-milestones.status`):

```
pending → submitted → approved
             ↑   │
             └───┴── changes_requested        (customer asks for revisions)
```

Payout status on the commission (`payout.status`):

```
not_applicable  legacy destination-charge rows, money already with the artist
unfunded        checkout created, money not captured yet
held            captured, nothing transferred
partially_released  sketch transfer done
released        artist net fully transferred
partially_refunded / refunded
disputed        a chargeback is open — releases blocked
```

**The approval decision and the money movement are separate writes.**
Customer approval flips the milestone to `approved` immediately (it is the
customer's decision and must never be lost to a Stripe error). The transfer runs
afterwards with a stable idempotency key and is retryable. If it fails, the
milestone stays approved, `payout.status` stays `held`/`partially_released`, and
`payout.lastError` records why with an admin alert.

---

## 4. What changes in the artist's flow

This is the behavioral core, and the part that will surprise people:

- **Queue "Completed" no longer completes the commission.** Dragging a queue card
  to Completed (`src/features/queue/actions/queue-actions.ts` → `updateQueueStatus`,
  `updateQueueItem`) and `finalizeCommissionAction`
  (`src/features/queue/actions/deliverable-actions.ts`) currently write
  `status: 'completed'` on the commission. They must instead **submit the final
  milestone** (`submitted`) and leave the commission `paid`.
- The commission page renders a derived badge **"Delivered — awaiting client
  approval"** when `status = paid` and the final milestone is `submitted`.
- `completed` becomes reachable only by: customer approval, or an admin
  force-release/complete.
- **The email changes role.** `finalizeCommissionAction` today sends
  "commission finished — leave a review" (`sendCommissionFinishedReviewEmail`).
  Under escrow, final submission sends **"ready for your approval"** with the
  approve / request-changes link; the star-rating invite moves to *after*
  approval, on the same `reviewStatus` machinery. The sketch submission sends its
  own approval email with the sketch files attached as links.
- The sketch milestone is submitted from the same deliverable uploader
  (`createRawMediaDeliverableAction` already produces private `raw-media`), just
  with `key: 'sketch'`.
- The artist sees "held for release" amounts in the queue/dashboard. Their
  Express dashboard will show €0 until the first approval; without this the
  feature reads as "I am not being paid".

---

## 5. Data model

### 5.1 `commissions` — new fields

| Field | Type | Notes |
| --- | --- | --- |
| `paymentRail` | select `destination` / `separate` | Written at checkout creation. **Refunds branch on this, not on the env flag** — flipping the flag must never change how an old charge is unwound. |
| `taxLiability` | select `artist` / `platform` | Copied from the studio at checkout. **The payout split reads this row, never live config.** |
| `payout.status` | select (see §3) | Group, admin-readable, sidebar. |
| `payout.amountCapturedCents` | number | From the webhook's session amount. |
| `payout.amountTaxCents` | number | `session.total_details.amount_tax` — the tax component as calculated. |
| `payout.amountRefundedCents` | number | Incremented by refund actions/webhook. |
| `payout.amountReleasedCents` | number | Recomputed from milestone rows, never incremented blindly. |
| `payout.lastError` | text | Release failure detail for admin retry. |
| `stripe.chargeID` | text | `session.payment_intent.latest_charge`; the `source_transaction` for every transfer. |
| `stripe.transferGroup` | text | `commission_<id>`. |

Existing `platformFeeCents` stays the single fee record. For separate charges it
means "fee retained", not "fee collected" — same number, and the reporting source
of truth for platform revenue.

### 5.2 New collection `commission-milestones`

Slug `commission-milestones`, ID prefix `cml_` (add to `src/lib/utils/id.ts`),
admin group `MY_WORK`, `hidden: true` in nav (managed from the commission view,
like `commission-reviews`).

Fields: `tenant` (relationship, `associateTenant` hook), `commission`
(relationship, indexed), `key` (`sketch` | `final`), `status`, `tokenHash`,
`tokenSentAt`, `submittedAt`, `approvedAt`, `changesRequestedAt`,
`deliverableFiles` (relationship → `raw-media`, hasMany), `transferID`,
`transferAmountCents`, `releasedAt`, `customerNote`.

Indexes: unique on (`commission`, `key`) — one sketch and one final per commission.

Access: create/update admin-only; read tenant-scoped. The customer-facing approval
page reads through a token action with `overrideAccess`, exactly like
`getCommissionByReviewToken`.

Tokens: reuse `src/lib/security/reviewTokens.ts`
(`generateReviewToken` / `hashReviewToken` / `isReviewTokenExpired`, 30-day TTL).
**Only the hash is stored.** Re-submitting a revised sketch mints a fresh token and
invalidates the old one (the write replaces `tokenHash`).

### 5.3 Migration

One Payload migration adds the fields + collection and backfills in-flight rows:

- every existing commission → `paymentRail: 'destination'`
- every commission with `status in (paid, completed)` → `payout.status: 'released'`,
  `amountCapturedCents = round(artistReview.finalPrice × 100)`,
  `amountReleasedCents = amountCapturedCents − (platformFeeCents ?? 0)`
- everything else → `payout.status: 'not_applicable'`

Without this backfill the panel would claim that money already transferred under
the old rail is "held", and an admin might release it twice.

---

## 6. Stripe call shapes

### 6.1 Checkout (separate charge)

`buildCommissionCheckoutParams` gains a branch; `transfer_data` and
`application_fee_amount` are **omitted**:

```ts
{
  mode: 'payment',
  line_items: [{ price_data: { currency: 'eur', unit_amount: unitAmount, tax_behavior: 'exclusive', product_data: {...} }, quantity: 1 }],
  automatic_tax:
    taxLiability === 'artist'
      ? { enabled: true, liability: { type: 'account', account: studio.stripeAccountId } }
      : { enabled: true, liability: { type: 'self' } },
  payment_intent_data: {
    transfer_group: `commission_${commission.id}`,
    ...(taxLiability === 'artist' ? { on_behalf_of: studio.stripeAccountId } : {}),
  },
  metadata: { commissionId, orderType: 'commission', paymentRail: 'separate', taxLiability, tenantId, ... },
}
```

The `taxLiability` value is persisted on the commission. Refunds and reports read
it per row — a config change must never reinterpret a historical sale.

### 6.2 Release (per milestone)

```ts
await stripe.transfers.create(
  {
    amount: releaseCents,
    currency: 'eur',
    destination: studio.stripeAccountId,
    source_transaction: chargeId,        // ties the transfer to this sale's funds
    transfer_group: `commission_${id}`,
    metadata: { commissionId, milestone: key, tenantId },
  },
  { idempotencyKey: `commission_${id}_${key}_release` }
)
```

`source_transaction` is the important safety property: Stripe accepts the transfer
request and **defers execution until that charge's funds are available**, instead
of failing or drawing on unrelated platform balance. The second transfer from the
same charge is capped by the charge's remaining untransferred amount — this is
verified in the sandbox suite (§10), not assumed.

### 6.3 Refund

```ts
await stripe.refunds.create({
  payment_intent: paymentIntentId,
  amount: refundCents,          // omitted = full
  reverse_transfer: true,       // pulls the money back from the artist, proportionally
})
```

With separate charges there is **no application fee to refund** — the platform fee
was never transferred, it is simply retained. `reverse_transfer: true` reverses
transfers proportionally to the refunded amount. If the artist has already spent
the released deposit, the reversal can leave their account negative or partially
unwound; the result must be surfaced to the admin, never swallowed.

`src/lib/stripe/sale-refund.ts` (`refundSaleFully`) currently assumes a destination
charge and one transfer. It gains a rail-aware path:

- `paymentRail: 'destination'` → existing behaviour unchanged (reverse one
  transfer, refund the application fee).
- `paymentRail: 'separate'` → refund with `reverse_transfer`, no fee refund,
  reverse each milestone transfer as needed.

### 6.4 Webhooks

`handleCheckoutSessionCompleted` (`src/lib/stripe/stripeWebhooks.ts`):
- **Fix an existing defect**: it marks a commission `paid` without checking
  `session.payment_status === 'paid'`. With async methods (Klarna, PayPal) the
  session completes *unpaid*, so today a commission can be marked paid before the
  money settles. Gate on `payment_status` and let
  `checkout.session.async_payment_succeeded` do the rest — it currently only
  fulfills shop orders and explicitly logs "nothing to fulfill" for commissions.
- Record `paymentRail`, `chargeID`, `transferGroup`, `amountCapturedCents`,
  `payout.status = 'held'`.

New handlers:

- `charge.refunded` (exists) → extend to update `payout.amountRefundedCents` and
  `payout.status`, and to handle a partially refunded separate charge.
- `charge.dispute.created` / `charge.dispute.closed` (**new** — no dispute
  handling exists today) → `payout.status = 'disputed'`, block releases, alert
  admin; on `lost`, reverse transfers; on `won`, restore.

### 6.5 Customer approval endpoints

New route group under the existing token-page pattern:
`/review/commission/[token]` keeps collecting the star review; add
`/approve/commission/[token]` (or `?milestone=sketch|final` on the same route)
with two buttons — **Approve** and **Request changes** — and an optional note.
Both actions:

1. hash the token, find the milestone (`overrideAccess`, limit 1),
2. reject if `status !== 'submitted'` or the token is older than 30 days,
3. conditional write `submitted → approved` / `submitted → changes_requested`,
4. only the winner proceeds to release/notify,
5. rate-limited per IP (reuse `apiRateLimiter`, as the invite claim page does).

---

## 7. Admin controls

Commission edit view gains a **Payout** panel (visible to `users` with tenant
access, i.e. `userHasTenantAccess`):

- money summary: captured / fee / released / remaining, milestone table with
  transfer ids and timestamps,
- **Release** on any `submitted` or `approved` milestone whose transfer failed
  (same idempotency key — a retry can never double-pay),
- **Refund** with an amount, a `reverse transfers` toggle (default on) and a
  required note; writes `refundedAt`, updates `payout`,
- **Complete without approval** (force-release + `completed`) — the escape hatch
  for a dead customer, logged with the admin's user id.

All admin money actions are conditional writes with `throwOnError: true`, check
the returned document, and re-read the commission before acting. A swallowed write
must never move money.

---

## 8. Failure modes and invariants

1. **One transfer per milestone, ever.** DB conditional claim first, then a Stripe
   idempotency key derived only from `(commission, milestone)`. A retry, a double
   click and a webhook replay all converge on the same transfer.
2. **Released total ≤ artist net.** Recomputed from milestone rows before every
   transfer; a mismatch is an error, not a rounding adjustment.
3. **Refunded total ≤ captured.**
4. **No release while `payout.status` is `refunded`, `partially_refunded` or
   `disputed`.**
5. **No release to an account with `stripeChargesEnabled === false`** at release
   time (the artist can lose the capability after paying). Hold + alert.
6. **Approval is never rolled back because Stripe failed.**
7. **An abandoned commission is never silently completed.** No auto-approve; see §9.
8. **Tax mode is read from the commission row, never from live config.** A studio
   switching `taxLiability` mid-flight must not change the split of an existing
   sale — the money would be transferred to the wrong party's benefit.
9. **Transfer total equals `artist net` for the persisted tax mode** — recomputed
   before every release, including after a partial refund.

---

## 9. The deadlock problem (no auto-approve)

Money held for months is the point — but a customer who simply stops answering
would freeze the artist's balance forever. Since auto-approve is rejected, use a
**reminder ladder with a human at the end**:

- day 3 and day 10 after final submission → reminder email to the customer,
- day 14 → admin dashboard flag "awaiting approval > 14 days" and a notification,
- the admin can force-release (§7) after reviewing the delivered files.

Same ladder for a submitted sketch (shorter: 3 / 7 days). Nothing moves money
automatically; the clock only produces notifications and an admin queue.

---

## 10. Testing

**Unit** (`tests/int/commission-payout-split.int.spec.ts`) — pure math:
€300/5%/50% split with no tax; the same sale with €63 tax under each tax mode
(artist-liable: artist net = captured − fee; platform-liable: artist net =
price − fee); rounding at odd cents; `depositPercent` 0 and 90; fee floor at a
€0.50 charge; artist net 0; released total equals artist net exactly for 1000
randomized amounts (property-style loop); the split is unchanged when the studio's
`taxLiability` is flipped after checkout.

**State machine** (`tests/int/commission-milestones.int.spec.ts`) — transition
table, token expiry, wrong-token rejection, double approval, approval after refund,
`changes_requested` then re-submit mints a new token.

**DB-backed** (`tests/int/commission-payout-db.int.spec.ts`, real Postgres) —
double release transfers once; release blocked by `disputed`; refund cannot exceed
captured; ledger recompute after a failed transfer leaves the milestone approved.

**Sandbox, opt-in** (`tests/int/payout-sandbox.int.spec.ts`, gated exactly like
`platform-fee-sandbox`: `sk_test_` + `RUN_STRIPE_SANDBOX_TESTS=1`) — real
separate charge, `source_transaction` sketch transfer, second transfer from the
same charge, full refund after both releases with `reverse_transfer`, and a retry
with the same idempotency key returning the same transfer id. Cleanup unwinds
charges, transfers and refunds, and the suite must not leak charges into an
artist's payout history (the reason the existing sandbox suite is opt-in).

**Sandbox gate before Phase 2** (`tests/int/tax-liability-sandbox.int.spec.ts`,
**written**, included in `yarn test:int:sandbox`) — the check that decides the tax
model, in five probes:

- **T1a** create a `mode: 'payment'` Checkout Session with
  `automatic_tax.liability: { type: 'account', account }` and
  `payment_intent_data.on_behalf_of`. A rejection fails with the API's own message
  and the three fallbacks, because it is the blocking question.
- **T1b** the same shape with `liability: { type: 'self' }`, so a rejection in T1a
  is not confused with a broken key or account.
- **T2** no registration on the artist's account → `amount_tax == 0`. This is the
  expected default for most individual artists, not a bug. Skips when the test
  account already carries a registration in the customer's country.
- **T3** a registration created on the **artist's** account → `amount_tax > 0` and
  `amount_total == subtotal + tax`. This is what proves the liability switch is
  read from the artist rather than the platform. Best-effort: if the shared test
  account cannot hold a registration, it skips with the reason.
- **T4** a real separate charge with `on_behalf_of` → `transfer_data` is null and
  no transfer references the charge (escrow survives artist-MoR), the charge's
  `on_behalf_of` is the artist, and an explicit `source_transaction` transfer then
  lands on the artist's account.

Cleanup reverses the transfer, refunds the charge, expires the sessions,
deactivates the registration and deletes the test customers. **Status: not yet
executed** — the design session had no outbound HTTPS, and the suite now skips
with an explicit message rather than a red failure in that case. The T1a result is
the one that unblocks Phase 2.

**Reconciliation** (`scripts/commission-payout-report.ts`) — per commission:
captured, fee, released, remaining, transfer ids, and a mismatch flag. Read-only;
run after the flag flips and before any accounting close.

---

## 11. Phased rollout

**Phase 1 — ledger only, no behavior change.**
`paymentRail` + `payout.*` + `stripe.chargeID/transferGroup`; backfill; payout
panel read-only; fix the `payment_status` defect; correct the "95% transferred
immediately" copy in `src/features/studio-settings/components/IntegrationsTab.tsx`.
Ships safely with the flag off.

**Phase 2 — milestones + approvals behind `COMMISSION_ESCROW_ENABLED`.**
New collection, sketch/final submission, approval pages, release/refund actions,
admin controls, `depositPercent` on `CommissionSettings`, sandbox suite. Flag off =
today's destination-charge checkout, unchanged.

**Phase 3 — flip the flag.**
New checkouts use separate charges. Old commissions keep `paymentRail:
'destination'` and their existing refund path. Monitor the reconciliation report
weekly for the first month.

**Rollback**: turn the flag off. In-flight separate-charge commissions continue to
work (their rail is recorded per row), and no data is lost.

---

## 12. Risks

- **Platform balance dependency.** Transfers debit the platform balance; Stripe's
  automatic payouts can sweep it. `source_transaction` defers a transfer until the
  charge's funds are available and caps it at that sale's amount, but the platform
  still needs the funds. Mitigations: set the platform payout schedule to
  manual/weekly, keep a reserve, and watch the reconciliation report.
  Stripe's real fix — funds segregation (`allocated_funds`) — is **preview only**
  (requires a `Stripe-Version` preview header); do not build on it yet, revisit
  when it is GA.
- **Artist experience.** Express shows €0 until the first approval. Needs explicit
  "held for release" UI and updated onboarding/settings copy, or artists will read
  it as non-payment.
- **Tax ownership is a business decision with a technical prerequisite.** §2.1
  records the decision (artist-liable, platform liable only for its fee) and its
  four caveats. The blocking unknown is whether
  `automatic_tax.liability: account` is accepted on a one-time Checkout Session —
  documented for subscriptions only, verified in the Phase-2 sandbox gate.
  Fallback is `taxLiability: 'platform'` per studio.
- **Most artists will collect no tax.** With no registrations configured,
  `automatic_tax` computes zero and the artist carries an unfilled obligation.
  Onboarding must make that visible rather than silent: a setup-tour step that
  collects country, business status, VAT ID, and registrations, plus a studio
  settings panel showing "tax collection: off — no registrations".
- **Today's tax money goes to the artist.** `automatic_tax` is enabled but
  `transfer_data` carries no `amount`, so the artist receives charge − fee, tax
  included. That happens to match the artist-liable decision, so the *money flow*
  is already right; what is wrong is that the platform, not the artist, is named
  as seller and tax calculator. Escrow + `on_behalf_of` makes the paperwork match
  the money. The shop keeps its destination charge and needs the same
  `on_behalf_of` + liability treatment as a separate fix.
- **Disputes.** Unchanged liability, but now the platform holds the funds and can
  actually reverse them — an improvement. Requires the new dispute handlers.
- **Regulatory.** Holding customer money against delivery can look like escrow or
  payment services in some jurisdictions. This is a business/legal question, not a
  code one; worth a review before volume grows. Flagged, not answered here.
- **Price changes mid-work** (scope creep) stay out of scope: `finalPrice` is
  locked at checkout and one charge funds the whole job.

---

## 13. Proposals to confirm before Phase 2

These are defaults I picked so the spec is buildable; they are not locked:

- **`depositPercent` default 50**, per-studio configurable, range 0–90.
- **`taxLiability` default `'artist'`**, per-studio configurable. This is the
  decision you stated; the flag exists so platform-liable studios are a config,
  not a fork.
- **Reminder cadence** 3 / 10 days then an admin flag at 14 for the final, 3 / 7
  for the sketch. No automatic release at any point.
- **Sketch approval is a real customer gate**, not the artist ticking the Sketch
  checklist item. It mints its own token and is what releases the deposit.
- **Studio can opt out** of the split with `depositPercent: 0` (single release at
  final approval); there is no per-commission override in v1.

## 14. Files

**New**
- `src/collections/CommissionMilestones.ts`
- `src/features/commissions/lib/payout-split.ts` (pure math + transition table)
- `src/features/commissions/actions/submit-milestone.action.ts`
- `src/features/commissions/actions/approve-milestone.action.ts`
- `src/features/commissions/actions/release-payout.action.ts`
- `src/features/commissions/actions/admin-payout.action.ts`
- `src/components/admin/CommissionPayoutPanel.tsx`
- `src/components/admin/StudioTaxPanel.tsx` — country, business status, VAT ID,
  registration list, and the "no registrations → tax collects 0" warning
- `src/app/(frontend)/(studio)/approve/commission/[token]/page.tsx`
- `src/migrations/<ts>_commission_escrow.ts|.json`
- `scripts/commission-payout-report.ts`
- `tests/int/tax-liability-sandbox.int.spec.ts` — the gate that decides §2.1

**Changed**
- `src/collections/Commissions.ts` — rail + payout group + charge fields
- `src/collections/CommissionSettings.ts` — `depositPercent`, `taxLiability`
- `src/collections/Studios.ts` — `taxCountry`, `taxStatus`, `taxId` (onboarding-collected)
- `src/features/commissions/lib/checkout.ts` — separate-charge payload, `on_behalf_of`, tax liability
- `src/features/commissions/actions/create-commission-checkout.action.ts`
- `src/lib/stripe/stripeWebhooks.ts` — payment_status gate, ledger writes, disputes
- `src/lib/stripe/sale-refund.ts` — rail-aware unwind
- `src/features/queue/actions/deliverable-actions.ts` — submit final, don't complete
- `src/features/queue/actions/queue-actions.ts` — same
- `src/lib/security/flags.ts` — `COMMISSION_ESCROW_ENABLED`
- `src/lib/utils/id.ts` — `cml_` prefix
- `src/features/studio-settings/components/IntegrationsTab.tsx` — payout copy
- `docs/stripe-connect-destination-charges.md` — note the commission exception

**Unchanged by design**
- Round slot accounting (`src/features/queue/lib/round-rules.ts`): a slot is still
  held from submission through `completed`. Payment timing never touched it.
- Shop and tips: destination charges stay as they are.
