---
status: shipped
collections: [artists, commission-reviews, commissions, customers, orders, product-reviews, products, queue, raw-media, studios]
features: [queue, reviews, shop]
---


# Customer Reviews & Testimonials (Commissions & Digital Shop)

> **Status**: Shipped

How customers submit reviews for completed commissions (`commission-reviews`) and digital shop purchases (`product-reviews`), how review tokens and verified-purchase checks are enforced, and how artists moderate and feature testimonials.

## Commission Reviews (`commission-reviews`)

Commission reviews are stored in `src/collections/CommissionReviews.ts` (under **My Work** in the admin panel) with a unique index on `commission` so each commission can only be reviewed once.

### Token-Based Review Invitation & Submission
1. **Automatic & Manual Invites**:
   - When an artist finalizes a commission in `finalizeCommissionAction` (`src/features/queue/actions/deliverable-actions.ts`), ArtPack generates a single-use review token (`generateReviewToken` in `src/lib/security/reviewTokens.ts`), stores only its SHA-256 hash (`hashReviewToken`) on `commissions.reviewToken`, sets `reviewStatus: 'invited'` and `reviewRequestSentAt`, and emails `/review/commission/[token]` to the customer.
   - Artists can also send or re-send a review invite for a completed commission via the `/api/commissions/:id/send-review-invite` endpoint (`src/collections/endpoints/sendCommissionReviewInvite.ts`).
2. **Submission Rules (`submitCommissionReviewAction` in `src/features/reviews/actions/create-commission-review.action.ts`)**:
   - **Rate Limiting**: Capped per IP by `reviewRateLimiter`.
   - **Expiration**: Review links expire if `isReviewTokenExpired(commission.reviewRequestSentAt)` returns true, requiring the artist to issue a fresh invite.
   - **Rating & Minimum Length**: Requires a `rating` from `1` to `5` stars and `reviewText` passing `validateReviewText` (`src/lib/security/reviewValidation.ts`, minimum 2 sentences).
   - **Anonymous Option (`isAnonymous`)**: Customers can check **Post Anonymously** (`isAnonymous: true`), which omits `characterName` and `finishedArtwork` (`raw-media` deliverable) from public display.
   - **Immutability**: Once submitted, `rating`, `characterName`, and `reviewText` are read-only in the admin (`update: noOneField`) so artists cannot alter customer feedback.

### Moderation & Featured Testimonials
- New commission reviews default to `status: 'pending'` (**Pending Moderation**).
- Artists can set `status` to `published` or `hidden`, toggle `isAnonymous`, and check `isFeatured` (**Featured Testimonial**) to pin the review to the studio's hero or pricing testimonials carousel (`src/features/reviews/actions/get-reviews.action.ts`).

## Digital Shop Product Reviews (`product-reviews`)

Product reviews are stored in `src/collections/ProductReviews.ts` (under **Shop & Sales** in the admin panel) with a unique compound index on `['product', 'customer']`.

### Verified Purchase Enforcement
Submitted via `submitProductReviewAction` (`src/features/reviews/actions/create-product-review.action.ts`):

1. **Authentication Required**: The reviewer must be logged in (`payload.auth`); anonymous guests receive `"Please log in to your account to review this product."`
2. **Completed Order Check (`isVerifiedPurchase`)**: The action queries `orders` via `getOrders` (`src/features/shop/actions/get-order.action.ts`) for an order where `user.value` equals the customer ID, `products` contains `productId`, and `status` is `'completed'`. If no completed order exists, submission is rejected with `"Only customers who have purchased this product can leave a review."`
3. **Validation**: Requires a `rating` between `1` and `5` stars, optional `title` (headline), and `reviewText` of at least 10 characters.
4. **Moderation**: Product reviews are created with `status: 'pending'` and `isVerifiedPurchase: true`. Artists moderate reviews in the admin panel (`published`, `pending`, or `rejected`); only `published` reviews appear publicly on `/shop/[slug]` (while a logged-in customer can always read their own pending review).
5. **Order Review Invites (`POST /api/orders/:id/send-review-invite`)**: From any completed order (`orders.status === 'completed'`), artists can click the **Send Review Invite** button (`src/components/admin/OrderReviewInviteButton.tsx`, backed by `src/collections/endpoints/sendOrderReviewInvite.ts`) to email the buyer links (`${studioBaseUrl}/shop/${slug}?review=true`) for all unreviewed products in that order and record `orders.reviewRequestSentAt`.
