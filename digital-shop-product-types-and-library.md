---
status: shipped
collections: [commission-settings, customers, digital-assets, media, orders, pages, product-types, products, studios]
features: [auth, shop, studio-settings, studios]
---


# Digital Shop, Product Types, Bundles & Customer Library

> **Status**: Shipped

How studios sell downloadable digital goods (`products`, `product-types`, `digital-assets`), how net payouts and bundles work, and how buyers purchase and download files in their customer library (`/dashboard/library`).

## Products & Completeness Validation (`products`)

Shop items live in `src/collections/Products.ts` and are managed via the custom table view (`src/components/admin/ProductsListView.tsx`).

- **Required Artwork & Deliverables**:
  - `requireAtLeastOneImage`: Every product must have at least one preview image (`media`) so shop cards and product pages always have artwork.
  - `requireAtLeastOneFile`: Every standard product must attach at least one private downloadable file (`digital-assets` in `src/collections/DigitalAssets.ts`), failing validation with `"Add at least one file — buyers download these after paying."`
- **Bundles (`productType === 'bundle'`)**:
  - When `productType` is `'bundle'`, the `digitalFiles` upload field is hidden and skipped by validation; instead, the artist selects existing products in `bundleProducts`. Buyers who purchase a bundle receive access to the downloadable files of every product included in the bundle.
- **Buyer Price vs. Net Payout (`price` & `priceNetCents`)**:
  - `price` is stored in cents (e.g., `5000` for `€50.00`).
  - On save, `applyNetPricing` (`src/collections/hooks/applyNetPricing.ts`) computes `priceNetCents` after deducting the studio's platform fee, and `NetPricePreview` shows the artist their estimated payout alongside the buyer price.

## Product Types & Categories (`product-types`)

Studios organize their shop using `src/collections/ProductTypes.ts`:

- **System vs. Custom Types**: Each studio has seeded system default types (`isSystem: true`, such as standard products and bundles) plus custom categories (`name`, `slug`, `icon`, `order`, `isEnabled`).
- **Deletion Protection**: System types (`isSystem: true`) cannot be deleted. Furthermore, `guardProductTypeDeletion` (`src/collections/hooks/guardProductTypeDeletion.ts`) runs in `beforeDelete` and blocks deleting any product type that is still referenced by one or more products.

## Shop Checkout & Customer Library Fulfillment

1. **Stripe Checkout (`createShopCheckoutSession` in `src/features/shop/actions/create-shop-checkout.action.ts`)**:
   - Requires customer authentication (`status: 401` if logged out).
   - Verifies tenant isolation so a request on a studio host can only purchase products belonging to that studio (`status: 403` on mismatch).
   - Uses the studio's configured currency from `src/collections/CommissionSettings.ts` (`getCommissionSettingsByTenant` in `src/features/studio-settings/actions/update-commission-settings.action.ts`) and requires the studio to have `stripeAccountId` and `stripeChargesEnabled` (`src/features/studios`).
   - Enables Stripe `automatic_tax: { enabled: true }` and `billing_address_collection: 'required'`, routing funds via destination charges and redirecting to `/dashboard/library?success=true`.
2. **Order Fulfillment & Library Downloads**:
   - Orders (`src/collections/Orders.ts`) are fulfilled idempotently by `src/features/shop/actions/fulfill-shop-order.action.ts` on both the `/dashboard/library` success render and the Stripe webhook (`checkout.session.completed` / `checkout.session.async_payment_succeeded`).
   - Customers view owned products in `/dashboard/library` via `src/features/shop/actions/get-owned-library-products.action.ts` (`src/features/auth`) and download files through authenticated presigned S3 URLs.
