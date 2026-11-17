# Admin data locks — what an artist may change on a money record

Four collections expose information that belongs to one of three parties: the
customer, the platform, or the studio. Only the studio's own work product is the
artist's to edit. This note is the policy, and where each rule is enforced.

Payload enforces all of it **server-side** in `payload/dist/fields/hooks/beforeValidate`:
a denied field is deleted from the incoming data, so REST and GraphQL clients are
held to the same rule as the admin form.

The UI tells a narrower story, and it is worth being precise about it:

- a **read** denial removes the field from the form entirely;
- a **collection-level** update denial makes the whole document read-only;
- a **field-level** update denial does **not** make that field read-only. Payload
deletes the value and restores the stored one, so the save *succeeds* and the
field silently reverts on reload. `admin.readOnly` is a static boolean, so it
cannot express “locked once this document is paid”.

That last case is why Commissions carries `rejectFrozenCommissionEdits`
(`src/collections/hooks/rejectFrozenCommissionEdits.ts`): a `beforeChange` hook
that compares the raw submitted body (`req.data`, which the update pipeline never
rewrites) against the stored document and throws an error naming the locked
fields. The write was already refused by field access; the hook makes the refusal
visible instead of letting the artist believe it saved.

System writers — Stripe webhooks, checkout fulfillment, migrations — pass
`overrideAccess: true`, which bypasses field access. That is deliberate: payment
confirmation must never be blocked by a UI-level lock.

## Orders — a receipt, not a draft

`src/collections/Orders.ts`

| Field | Rule | Why |
|---|---|---|
| `status` | admin-only update | `refunded`/`completed` are money events recorded atomically with `refundedAt` and library access. A hand-set status desyncs them. |
| `totalAmount` | admin-only update | The amount actually charged. |
| `products` | admin-only update | Decides what the buyer owns and can download. |
| `user` | admin-only update | Reassigning moves library access to another account. |
| `stripeSessionID` | admin-only read + update | Payment plumbing; also no longer the document title. |
| delete (collection) | admin only | Deleting a sale erases revenue from reporting and revokes access. |
| update (collection) | admin only | Every artist-editable field is already locked, so a writable record only produced a form whose saves silently reverted. |

`useAsTitle` is `id`, not `stripeSessionID`. The title is the document *label* —
it appears in the edit header, breadcrumbs and every relationship cell — so a
Stripe session id there leaked payment plumbing into places a field-level rule
could not reach.

`sendOrderReviewInvite` still writes `reviewRequestSentAt` for artists. That is
the one artist-triggered write, so the endpoint performs its own authorization
(users session, tenant membership, `completed` status, unreviewed items) and then
writes with `overrideAccess: true` as the system. With that exception, orders are
admin-writable only: an artist opens the receipt read-only.

## Tips — a payment record

`src/collections/Tips.ts`

Every recorded value (`amount`, `currency`, `email`, `name`, `message`, `user`,
`stripeSessionId`) is admin-writable only, `status` included: it previously
checked only "is a users-collection session", which every artist satisfies, so
an artist could mark a pending tip `completed`. Delete is admin-only.

`createTipCheckoutSession` writes the pending tip with `overrideAccess: true` —
the platform's checkout path, not an artist write — and does so **explicitly**.
Relying on `tenantPayload`'s “no user means override” default would 403 the tip
flow the day a caller starts passing `user`.

## Commissions — the order facts freeze at payment

`src/access/immutableAfterPayment.ts`

Once the commission reaches `paid` or `completed`, these are refused for artists:

`customer`, `commissionType`, `seat`, `seatSnapshot`, `characterName`,
`referenceImages`, `customerNotes`, `drawingTier`, `customTier`, `background`,
`tierSelections`, `selectedTiers`, `aiAnalysis`.

They describe what the customer bought and the quote they were shown. Letting the
artist rewrite them afterwards changes the purchase without changing a number in
Stripe.

Still artist-writable after payment — the studio's own work product: `status`
(within the transition rule below), `batch`, `artistReview` (final price, notes,
breakdown), `deliverableFiles`, and the review fields.

The check reads the **stored** document first. A single request cannot flip
`status` to an unpaid value and edit the facts alongside it.

### `status` is a transition rule, not a role check

`commissionStatusFieldAccess` (`src/access/immutableAfterPayment.ts`)

Reading only the stored document was not enough, because it left a two-request
bypass: step 1 flips `status` from `paid` back to `approved` (allowed by the old
role check), step 2 edits the facts (the helper now sees an unpaid stored
status and allows them).

The transition itself is now the gate:

- admins keep full control (support and repair);
- artists drive the workflow freely before payment (`needs_artist_review` →
  `approved` / `rejected`, …);
- once `paid` or `completed`, the only allowed next statuses are the paid ones —
  the studio can still finish the job (`paid` → `completed`), but cannot undo the
  sale;
- an update that omits `status` is not a transition and passes.

Webhooks and server actions write with `overrideAccess`, so the payment
confirmation that *creates* the paid state is never blocked by this rule.
`rejectFrozenCommissionEdits` reports a refused transition with the same message
as a refused field.

## Products — platform plumbing stays with the platform

`src/collections/Products.ts`

- `meta` (SEO title/description/image) is admin-only. The storefront already
  derives all three from the product itself, so hiding the override loses
  nothing and stops two titles drifting apart.
- `stripeID`, `stripePriceID`, `docUrl` are admin-only. Products live on the
  **platform's** Stripe account (checkout charges are destination charges to the
  studio), so the ids and the dashboard link are dead ends for an artist.

Neither change rewrites stored data: a hidden field is not submitted, and
Payload's update merges, so admin-set values survive an artist's edit.

## Tests

`tests/int/admin-field-locks.int.spec.ts` pins the collection configs (which
fields carry a lock, and what the lock answers for an artist vs an admin), the
status transition matrix in both directions, the rejection hook (including the
“unchanged whole-document save” case), and the Orders collection gate; plus
`orders-list-view` / `tips-list-view` for the delete affordance: an artist sees
the row and the view action, never a delete button that would 403.

Not covered by a test: the admin form's *appearance* once a commission is paid.
The fields stay visually editable and the save is refused with an error, because
Payload offers no per-document `readOnly`. If that proves confusing in practice,
the next step is a small custom `Field` wrapper rather than a second policy.
