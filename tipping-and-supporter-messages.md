---
status: shipped
collections: [artists, customers, studios, tips]
features: [auth, studios, tipping]
---


# Studio Tipping & Supporter Messages (`/tip`)

> **Status**: Shipped

How supporters send tips and messages to artists via the studio `/tip` page (`tips` collection), how Stripe Connect destination charges and platform fees are applied, and how artists view received tips.

## Tip Jar Flow & Validation

Each studio storefront exposes a `/tip` route backed by `src/collections/Tips.ts` and `createTipCheckoutSession` (`src/features/tipping/actions/create-tip-checkout.action.ts`):

- **Minimum Amount & Currency**: Tips are denominated in `EUR` (`currency: 'EUR'`, stored in cents on `tips.amount`). The minimum tip is **€1.00** (`amount < 1` is rejected with `"Minimum tip is €1.00"`).
- **Supporter Details**:
  - `userEmail` (`tips.email`) is required so Stripe Checkout and 2monke can send a receipt (`"Email is required"`).
  - `userName` (`tips.name`) and `message` (`tips.message`) are optional supporter fields displayed to the artist in the admin panel.
  - If the supporter is logged in (`src/features/auth`), their `users` or `customers` account is linked on `tips.user`.
- **Stripe Connect Payment Gate**:
  - The target studio (`src/features/studios`) must have both `stripeAccountId` and `stripeChargesEnabled`. Otherwise, checkout fails closed with `"This studio is not currently configured to accept tips. Payouts must be connected in the studio dashboard."`

## Checkout, Fee Calculation & Fulfillment

1. **Pending Record Creation**: `createTipRecord` (`src/features/tipping/actions/create-tip.action.ts`) creates a `tips` document with `status: 'pending'` and `amount` in cents.
2. **Stripe Checkout Session**:
   - `resolveStudioPlatformFee` (`src/lib/stripe/stripe.ts`) computes the platform fee in cents (`platformFeeInCents`), and `createTipCheckoutSession` creates a Stripe Checkout session with `payment_intent_data.application_fee_amount` and `transfer_data.destination` set to the studio's `stripeAccountId`.
   - Redirects return to `${baseUrl}/tip/success?session_id={CHECKOUT_SESSION_ID}` or `${baseUrl}/tip/cancel` (validated against the studio's domains by `sanitizeCheckoutOrigin` in `src/lib/stripe/checkout-origin.ts`).
   - `updateTipStripeSession` (`src/features/tipping/actions/update-tip.action.ts`) records the `stripeSessionId` on the tip document.
3. **Webhook Confirmation & Notifications**:
   - When Stripe confirms payment (`src/lib/stripe/stripeWebhooks.ts`), the tip's `status` is updated to `'completed'` alongside `platformFeeCents` (protected by `adminRoleFieldAccess` so artists cannot edit fee accounting).
   - `sendTipReceivedEmail` (`src/lib/email/index.ts`) sends a branded thank-you email to the supporter, and `revalidateDashboardStatsAfterChange` updates the studio's revenue statistics.
   - Artists review all received tips, supporter names, and messages in the custom admin table view (`src/components/admin/TipsListView.tsx`) under **Shop & Sales → Tips**.
