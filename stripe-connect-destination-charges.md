---
status: shipped
collections: [artists, commissions, products, tips]
features: []
---


# Stripe Connect & Destination Charges

How ArtPack routes multi-tenant marketplace payments to connected artists using Stripe Connect Express and Destination Charges with platform split fees.

## Architecture

```
Buyer Checkout (Commission / Shop / Tip)
             │
             ▼ (e.g. 100.00 EUR)
    ┌──────────────────────────────┐
    │    Platform Stripe Account   │
    └──────────────┬───────────────┘
                   │
         ┌─────────┴─────────┐
         │                   │
         ▼                   ▼
┌──────────────────┐  ┌───────────────────────────────────┐
│   Platform Fee   │  │   Destination Transfer            │
│  (e.g. 5.00 EUR) │  │  (95.00 EUR)                      │
└──────────────────┘  └──────────────┬────────────────────┘
                                     │
                                     ▼
                      ┌───────────────────────────────────┐
                      │   Connected Artist Stripe Account │
                      │   (stripeAccountId: acct_xxx)     │
                      └──────────────┬────────────────────┘
                                     │
                                     ▼
                      ┌───────────────────────────────────┐
                      │   Artist Bank Account / Payout    │
                      └───────────────────────────────────┘
```

## Key Components

### 1. Stripe Helper Client & Utilities (`src/lib/stripe/stripe.ts`)

- **`calculatePlatformFee(amountInCents, feePercent?, feeFixedCents?)`**: Computes platform commission (5% launch rate, 7-10% steady state, minimum 0.50 EUR; resolved per studio by `resolveStudioPlatformFee`).
- **`createStripeExpressAccount(artist)`**: Creates a Stripe Express Connected Account configured with `mcc: '5818'` (Digital Goods/Artist) and `transfers` + `card_payments` capabilities.
- **`createStripeAccountLink(params)`**: Generates Stripe-hosted KYC onboarding link (`type: 'account_onboarding'`).
- **`createStripeLoginLink(accountId)`**: Generates single-sign-on Express Dashboard link.
- **`getStripeAccountStatus(accountId)`**: Fetches live verification status (`charges_enabled`, `payouts_enabled`, `details_submitted`, `requirements`).

### 2. Onboarding & Account Management Endpoints

- **`POST /api/stripe/connect/onboard`**:
  - Authenticates request and validates tenant ownership.
  - Creates Stripe Express account on first invocation and stores `stripeAccountId` on `artists` record.
  - Returns `{ url }` to redirect user to Stripe's onboarding flow.
- **`POST /api/stripe/connect/dashboard`**:
  - Authenticates artist and generates a secure Stripe Express Dashboard login link.
- **`GET /api/stripe/connect/status`**:
  - Fetches live status from Stripe API and synchronizes `stripeChargesEnabled`, `stripePayoutsEnabled`, and `stripeDetailsSubmitted` in the database.

### 3. Payment Flow & Destination Charges

All checkout sessions check if the artist has a verified Stripe account (`stripeChargesEnabled === true`):

```typescript
const isConnectedArtist = Boolean(artist?.stripeAccountId && artist?.stripeChargesEnabled)
const platformFee = isConnectedArtist ? calculatePlatformFee(unitAmount) : 0

const session = await stripe.checkout.sessions.create({
  payment_method_types: ['card', 'paypal', 'revolut_pay', 'klarna'],
  line_items: [...],
  mode: 'payment',
  ...(isConnectedArtist && artist?.stripeAccountId
    ? {
        payment_intent_data: {
          application_fee_amount: platformFee,
          transfer_data: {
            destination: artist.stripeAccountId,
          },
        },
      }
    : {}),
  metadata: {
    tenantId: String(artist.id),
    artistStripeAccountId: artist.stripeAccountId,
    platformFeeInCents: String(platformFee),
  },
})
```

**Affected Gateways:**
- **Commissions**: `src/app/(app)/dashboard/commission/[id]/actions.ts`
- **Digital Shop Products**: `src/app/api/checkout/route.ts`
- **Artist Tips / Direct Support**: `src/app/api/tips/checkout/route.ts`

### 4. Real-time Webhook Synchronization (`src/lib/stripe/stripeWebhooks.ts`)

- Listens for `account.updated` events from Stripe.
- Matches `event.data.object.id` against `artists.stripeAccountId`.
- Automatically syncs `stripeChargesEnabled`, `stripePayoutsEnabled`, and `stripeDetailsSubmitted` in Payload CMS.

### 5. Artist Studio Settings UI (`src/components/admin/SettingsView.tsx`)

- **"💳 Payouts & Stripe" Tab**:
  - **Not Connected**: One-click *"Connect with Stripe Express ↗"* button.
  - **Action Required / Pending KYC**: Warning badge with *"Complete Verification ⚠️"* button.
  - **Active & Verified**: Green status indicators + *"Open Stripe Dashboard ↗"* for tax forms, bank settings, and payout history.

## Debug & Verification Reference

1. **Verify Connect Onboarding URL Generation**:
   ```bash
   curl -X POST http://localhost:3000/api/stripe/connect/onboard \
     -H "Content-Type: application/json" \
     -H "Cookie: payload-token=..." \
     -d '{"artistId": 1}'
   ```
2. **Verify Account Status Synchronization**:
   ```bash
   curl http://localhost:3000/api/stripe/connect/status?artistId=1 \
     -H "Cookie: payload-token=..."
   ```
3. **Verify Webhook Logs**:
   Look for `[WEBHOOK-CONNECT] Processing account.updated for acct_...` in server console logs.
