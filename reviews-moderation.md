# Reviews: Moderation From the Parent Document

## Why

Reviews used to be managed from two collections in the admin nav
(`commission-reviews`, `product-reviews`). The artist never thinks in those
terms — a review belongs to one commission or one product — and a separate list
invites bulk edits that the review flow was never designed for.

Now both collections are **hidden from the admin nav** and moderation lives
inside the document that owns the review:

| Parent | Relation | Panel |
| --- | --- | --- |
| Commission | 1:1 (`unique` on the relationship + DB index) | `ReviewModerationPanel kind="commission"` |
| Product | 1:N (unique only per `product + customer`) | `ReviewModerationPanel kind="product"` |

No schema change was needed: the uniqueness constraints already existed.

## Public visibility

New reviews are forced to `pending` by the create flows
(`create-commission-review` sets it; `verifyPurchaseBeforeReviewCreate` forces
it for products). Every public fetcher filters `status = 'published'`, and the
anonymous read access rule does too — so a review is invisible until an artist
publishes it. That rule predates the panels; the panels are simply where the
artist makes the decision.

## Status vocabulary

- `commission-reviews`: `pending | published | hidden`
- `product-reviews`: `pending | published | rejected` (hide = `rejected`)

The panel renders both `hidden` and `rejected` as **Hidden**. "Publish" and
"Hide" are the whole vocabulary — there is no delete, edit, or feature action in
the panel (deliberate for now).

## Authorization

`src/features/reviews/actions/moderate-review.action.ts`:

1. Session must be a `users` (artist/admin) account.
2. The review is read first; its tenant is the boundary — the caller never
   supplies one.
3. `userHasTenantAccess(user, review.tenant)` must pass.
4. The write uses the **session user** (`overrideAccess: false`), so the
   collection's own `ownTenantOnly` rule stays in force as a second gate.
5. Revalidation runs for the review family (the collection hooks also do it).

`get-admin-reviews.action.ts` resolves the parent document (commission/product)
with the session, checks its tenant, and only then lists its reviews — a review
of another studio's document is unreachable by id.

## Files

- `src/components/admin/ReviewModerationPanel.tsx` — the card, plus the
  `CommissionReviewPanel` / `ProductReviewPanel` field components.
- `src/features/reviews/actions/get-admin-reviews.action.ts` — tenant-scoped read.
- `src/features/reviews/actions/moderate-review.action.ts` — publish/hide.
- `src/collections/Commissions.ts` / `src/collections/Products.ts` — `ui` field
  that mounts the panel at the end of the main column.
- `src/collections/CommissionReviews.ts` / `src/collections/ProductReviews.ts` —
  `admin.hidden: true`.

## Tests

- `tests/int/admin-review-moderation.int.spec.ts` — panel rendering: empty
  state, publish, hide, anonymous names, product list + `rejected` → Hidden.
- `tests/int/review-moderation-action.int.spec.ts` — tenant refusal, role
  refusal, invalid action, status mapping, anonymous name suppression.
- `tests/int/review-moderation-db.int.spec.ts` — real database: an artist of the
  review's studio publishes through Payload access control; another studio is
  refused; the panel then reads the hidden review.
- `tests/int/artist-admin-visibility.int.spec.ts` — both collections are hidden.

## Known gaps

- No delete / bulk moderation / feature toggle in the panel.
- Platform admins no longer see the collections in the nav either (they can
  still reach `/admin/collections/...` by URL). Several other collections are
  candidates for the same treatment — a separate nav audit.
