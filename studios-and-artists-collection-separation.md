---
status: spec
collections: [announcements, artists, commission-settings, commissions, media, orders, pages, products, queue, sprints, studios, themes, tips]
features: [onboarding, studio-settings]
---


# Studios & Artists Collection Separation Architecture

Architectural specification for segregating **`studios`** as the primary multi-tenant organization entity and **`artists`** as creator/staff member profiles within studios, eliminating backward-compatibility legacy fallbacks in favor of a clean, robust multi-tenant model.

---

## 1. Architecture Overview

```
                               ┌───────────────────────────┐
                               │           Users           │
                               │  (Authentication & Auth)  │
                               └─────────────┬─────────────┘
                                             │
                       ┌─────────────────────┴─────────────────────┐
                       │ (owner)                                   │ (user)
                       ▼                                           ▼
          ┌─────────────────────────┐                  ┌───────────────────────┐
          │         Studios         │◄─── [artists] ───│        Artists        │
          │   (Multi-Tenant Unit)   │                  │   (Creator Profile)   │
          └────────────┬────────────┘                  └───────────────────────┘
                       │
       ┌───────────────┼───────────────────────────┬───────────────────────┐
       │ (tenant)      │ (tenant)                  │ (tenant)              │ (tenant)
       ▼               ▼                           ▼                       ▼
┌──────────────┐┌──────────────┐             ┌───────────┐           ┌───────────┐
│ Commissions  ││ Sprints      │             │ Products  │           │ Orders    │
│ & Queue      ││ & Batches    │             │ & Assets  │           │ & Tips    │
└──────────────┘└──────────────┘             └───────────┘           └───────────┘
       │               │                           │                       │
       ▼               ▼                           ▼                       ▼
┌──────────────┐┌──────────────┐             ┌───────────┐           ┌───────────┐
│ Themes       ││ Settings     │             │ Pages     │           │ Media     │
│ (Branding)   ││ (Pricing/AI) │             │ (Content) │           │ (Storage) │
└──────────────┘└──────────────┘             └───────────┘           └───────────┘
```

---

## 2. Core Entity Modeling

### `Studios` Collection (`src/collections/Studios.ts`)
The central tenant entity representing an artist shop / studio storefront:
- **Identifier**: `subdomain` (e.g., `vronyakat`), `customDomain` (optional).
- **Ownership**: `owner` (`relationship` -> `users`, required).
- **Affiliated Creators**: `artists` (`relationship` -> `artists`, `hasMany: true`).
- **Configuration**:
  - `theme` (`relationship` -> `themes`).
  - `commissionSettings` (`relationship` -> `commission-settings`).
  - `mascot` (`relationship` -> `media`).
- **Stripe Connect Integration**:
  - `stripeAccountId`, `stripeChargesEnabled`, `stripePayoutsEnabled`, `stripeDetailsSubmitted`.
- **Social Links & Metadata**: `socialLinks` array (`platform`, `url`, `label`).

### `Artists` Collection (`src/collections/Artists.ts`)
The profile entity representing individual creator identities:
- **User Link**: `user` (`relationship` -> `users`, required).
- **Profile Info**: `name`, `bio`, `avatar` (`relationship` -> `media`), `specialties` array.
- **Affiliations**: `studios` (`relationship` -> `studios`, `hasMany: true`).
- **Portfolio Showcase**: `portfolio` (`relationship` -> `media`, `hasMany: true`).
- **Socials**: `socialLinks` array (`platform`, `url`, `label`).

### `Users` Collection (`src/collections/Users.ts`)
- **Tenant Scope**: `tenants` (`relationship` -> `studios`, `hasMany: true`, `saveToJWT: true`).
- **Roles**: `['admin', 'artist']`.

---

## 3. Tenant Scoping Across Collections

All tenant-scoped collections in ArtPack maintain their primary tenant foreign key pointing directly to `studios` (`relationTo: 'studios'`):

| Collection | Tenant Relation Field | Description |
| :--- | :--- | :--- |
| **`Announcements`** | `tenant` (`relationTo: 'studios'`) | Studio banner announcements |
| **`Commissions`** | `tenant` (`relationTo: 'studios'`) | Client commission requests |
| **`CommissionSettings`** | `tenant` (`relationTo: 'studios'`) | Pricing tiers, AI guidelines, webhook config |
| **`DigitalAssets`** | `tenant` (`relationTo: 'studios'`) | Digital download files & courses |
| **`Media`** | `tenant` (`relationTo: 'studios'`) | Uploaded images, reference sheets, banners |
| **`Orders`** | `tenant` (`relationTo: 'studios'`) | Digital store transactions & checkout records |
| **`Pages`** | `tenant` (`relationTo: 'studios'`) | CMS layout pages (`home`, `shop`, `queue`, etc.) |
| **`Products`** | `tenant` (`relationTo: 'studios'`) | Storefront digital & physical products |
| **`Queue`** | `tenant` (`relationTo: 'studios'`) | Production pipeline cards & checklist items |
| **`QueueComments`** | `tenant` (`relationTo: 'studios'`) | Client & creator thread comments |
| **`RawMedia`** | `tenant` (`relationTo: 'studios'`) | Presigned direct S3 upload registry |
| **`Sprints`** | `tenant` (`relationTo: 'studios'`) | Commission batch periods |
| **`Themes`** | `tenant` (`relationTo: 'studios'`) | Styling tokens, colors, typography, header layout |
| **`Tips`** | `tenant` (`relationTo: 'studios'`) | Artist tipping & supporter contributions |

---

## 4. Key Workflows & Lifecycle Integration

### 1. Auto-Tenant Creation on Signup (`createTenantOnArtistSignup.ts`)
When a user with the `artist` role is registered:
1. Validates and generates a unique `subdomain` against `studios`.
2. Creates an `artists` creator profile record (`user: user.id`).
3. Creates the `studios` tenant record (`owner: user.id, artists: [artist.id]`).
4. Creates default `themes` and `commission-settings` linked to `studio.id`.
5. Links `theme` and `commissionSettings` to `studios`, and updates `artists.studios = [studio.id]`.
6. Assigns `users.tenants = [studio.id]`.

### 2. Multi-Step Onboarding Action (`artist-onboarding.action.ts`)
- **Step 1 Validation**: Queries `studios` collection for subdomain availability.
- **Atomic Creation**: Creates user, artist profile, studio tenant, theme, and commission settings in order.
- **Rollback Resilience**: In case of failures, automatically rolls back all partially created resources including `createdStudioId`.
- **Tenant Context**: Sets `payload-tenant-context` cookie to `studio.id`.

### 3. Payment Processing & Stripe Connect
- **Tips Route (`src/app/api/tips/checkout/route.ts`)**: Resolves studio tenant directly via subdomain/ID and routes destination charges via `studio.stripeAccountId`.
- **Store Checkout (`src/app/api/checkout/route.ts`)**: Resolves product tenant from `studios` and retrieves `studio.stripeAccountId`.
- **Commission Review (`commission-review.action.ts`)**: Creates Stripe checkout session using `studio.stripeAccountId` with platform fee split.

### 4. Admin Panel & Settings
- **Tenant Selector (`TenantSelector.tsx`)**: Fetches active directory from `/api/studios?limit=100` and displays subdomains.
- **Studio Settings View (`SettingsView.tsx`)**: Directly interacts with `/api/studios/${studioId}` and `/api/commission-settings` without legacy fallback queries.

---

## 5. Files Changed Reference

| File | Nature of Changes |
| :--- | :--- |
| `src/collections/Studios.ts` | **New collection** defining primary studio tenant schema with relations and Stripe Connect fields. |
| `src/collections/Artists.ts` | Updated to artist profile entity with `studios` array relationship. |
| `src/collections/Users.ts` | Updated `tenants` field `relationTo: 'studios'`. |
| `src/collections/*.ts` (all collections) | Updated all `tenant` relation fields from `'artists'` to `'studios'`. |
| `src/collections/hooks/createTenantOnArtistSignup.ts` | Creates studio + artist bundle and assigns studio ID to `user.tenants`. |
| `src/features/onboarding/actions/artist-onboarding.action.ts` | Subdomain uniqueness on `studios`, `studio.id` tenant cookie, and studio rollback. |
| `src/features/studio-settings/components/SettingsView.tsx` | Switched from `/api/artists` to `/api/studios`. |
| `src/components/TenantSelector.tsx` | Switched directory fetch to `/api/studios`. |
| `src/components/admin/AdminHeaderActions.tsx` | Switched live preview URL resolution to query `studios`. |
| `src/lib/cache.ts` | Direct studio resolution for `fetchStudio` and `getStudio`. |
| `src/lib/infra/cleanup.ts` | Cleans up `studios` owned by user along with all tenant-scoped data and `artists` profiles. |
| `src/lib/stripe/stripeConnectAuth.ts` | Validates studio ownership and Stripe account details directly from `studios`. |
| `src/app/api/tips/checkout/route.ts` | Studio tenant resolution and Stripe destination transfer routing. |
| `src/app/api/checkout/route.ts` | Studio tenant resolution for product payments. |
| `src/app/api/submit-commission/route.ts` | Associates commissions and uploaded reference media directly with `studio.id`. |
| `src/seedGlobals.ts` & `src/app/(frontend)/api/seed/route.ts` | Seeds studio tenants, themes, commission settings, pages, and linked artist profiles from scratch. |

---

## 6. Debug & Troubleshooting Reference

### 1. "Cannot find tenant" on public subdomain
- **Check**: Verify the record exists in `studios` collection with matching `subdomain` value.
- **Check**: Verify `studios.theme` and `studios.commissionSettings` are populated.

### 2. Tenant Context Cookie Mismatch
- **Check**: The `payload-tenant-context` cookie should hold the ID of the `studios` document (not the `artists` profile ID).
- **Check**: The active user's `tenants` array in `users` should contain the matching `studio.id`.

### 3. Stripe Connect Payouts not transferring
- **Check**: Verify `studio.stripeAccountId`, `studio.stripeChargesEnabled = true`, and `studio.stripePayoutsEnabled = true`.
