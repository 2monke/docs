# Admin data locks — what an artist may change on a money record

Four collections expose information that belongs to one of three parties: the
customer, the platform, or the studio. Only the studio's own work product is the
artist's to edit. This note is the policy, and where each rule is enforced.

Payload enforces all of it **server-side** in `payload/dist/fields/hooks/beforeValidate`:
a denied field is deleted from the incoming data, so REST and GraphQL clients are
held to the same rule as the admin form. The admin UI then reflects it — a field
the session cannot update renders read-only, and a field it cannot read is not
rendered at all (`RenderFields` → `getFieldPermissions`). The rules below
therefore do not depend on anyone hiding a widget.

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

`useAsTitle` is `id`, not `stripeSessionID`. The title is the document *label* —
it appears in the edit header, breadcrumbs and every relationship cell — so a
Stripe session id there leaked payment plumbing into places a field-level rule
could not reach.

`sendOrderReviewInvite` still writes `reviewRequestSentAt` for artists, so
collection-level update stays `ownTenantOnly`; the lock is per field.

## Tips — a payment record

`src/collections/Tips.ts`

Every recorded value (`amount`, `currency`, `email`, `name`, `message`, `user`,
`stripeSessionId`) is admin-writable only, `status` included: it previously
checked only "is a users-collection session", which every artist satisfies, so
an artist could mark a pending tip `completed`. Delete is admin-only.

`createTipCheckoutSession` writes the Stripe session id back onto the pending tip
with `overrideAccess: true` — the customer's checkout path, not an artist write.

## Commissions — the order facts freeze at payment

`src/access/immutableAfterPayment.ts`

Once the commission reaches `paid` or `completed`, these are refused for artists:

`customer`, `commissionType`, `seat`, `seatSnapshot`, `characterName`,
`referenceImages`, `customerNotes`, `drawingTier`, `customTier`, `background`,
`tierSelections`, `selectedTiers`, `aiAnalysis`.

They describe what the customer bought and the quote they were shown. Letting the
artist rewrite them afterwards changes the purchase without changing a number in
Stripe.

Still artist-writable after payment — the studio's own work product: `status`,
`batch`, `artistReview` (final price, notes, breakdown), `deliverableFiles`, and
the review fields.

The check reads the **stored** document first. A single request cannot flip
`status` to an unpaid value and edit the facts alongside it.

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
fields carry a lock, and what the lock answers for an artist vs an admin), plus
`orders-list-view` / `tips-list-view` for the delete affordance: an artist sees
the row and the view action, never a delete button that would 403.
