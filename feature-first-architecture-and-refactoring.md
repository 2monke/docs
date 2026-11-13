---
status: spec
collections: [commissions, media, products, queue, tips]
features: [auth, commissions, dashboard, onboarding, pricing, queue, shop, studio-settings, tipping]
---


# Feature-First Architecture & Codebase Refactoring

Comprehensive specification of 2monke's modular Feature-First architecture, feature decomposition, and component colocation.

## Architecture Overview

2monke adopts a **Feature-First Architecture** where domain-specific logic, components, state, server actions, and services are encapsulated within self-contained modules under `src/features/<feature-name>/`. App Router routes (`src/app/`) and Payload CMS admin panels serve as thin consumers and coordinators.

```
src/
├── features/
│   ├── commissions/             # Commission submission, mascot, typewriter, uploads
│   │   ├── components/
│   │   │   ├── CommissionForm.tsx
│   │   │   ├── FileUpload.tsx
│   │   │   ├── FormStatus.tsx
│   │   │   ├── Mascot.tsx
│   │   │   └── Typewriter.tsx
│   │   └── index.ts
│   │
│   ├── dashboard/               # Artist/client dashboard navigation, cards, profile
│   │   ├── actions/
│   │   │   └── profile-actions.ts
│   │   ├── components/
│   │   │   ├── CommissionCard.tsx
│   │   │   ├── DashboardFilters.tsx
│   │   │   ├── DashboardNav.tsx
│   │   │   └── ProfileHeader.tsx
│   │   └── index.ts
│   │
│   ├── onboarding/              # Multi-step artist onboarding modal & generator
│   │   ├── actions/
│   │   │   └── artist-onboarding.action.ts
│   │   ├── components/
│   │   │   ├── OnboardingModal.tsx
│   │   │   └── ...
│   │   └── index.ts
│   │
│   ├── shop/                    # Digital products, character bases, secure PDF viewer
│   │   ├── components/
│   │   │   ├── SecurePDFViewer.tsx
│   │   │   ├── SecurePDFViewerClient.tsx
│   │   │   ├── ShopItem.tsx
│   │   │   └── ShopList.tsx
│   │   └── index.ts
│   │
│   ├── studio-settings/         # Decomposed studio admin settings tabs
│   │   ├── components/
│   │   │   ├── CommissionTypesTab.tsx
│   │   │   ├── GeneralSettingsTab.tsx
│   │   │   ├── IntegrationsTab.tsx
│   │   │   ├── PricingBlocksTab.tsx
│   │   │   ├── SettingsView.tsx
│   │   │   └── ThemeSettingsTab.tsx
│   │   ├── styles.ts
│   │   ├── types.ts
│   │   └── index.ts
│   │
│   └── tipping/                 # Fan tips, micro-transactions, and tip checkout
│       ├── components/
│       │   ├── TipAmountSelector.tsx
│       │   └── TipButton.tsx
│       ├── services/
│       │   └── tipService.ts
│       └── index.ts
│
├── app/                         # Thin Next.js App Router route entrypoints
├── components/                  # Shared primitives (Button, Input, ThemeProvider, etc.)
└── lib/                         # Core infrastructure (cache, tenant, eventBus, stripe)
```

---

## Key Features & Component Breakdown

### 1. Studio Settings (`src/features/studio-settings/`)

Decomposed from the monolithic 1,440-line `SettingsView.tsx` into modular tabs and typed configs:

- **`types.ts`**: Centralized TypeScript definitions for artist data models, commission settings, pricing tiers, layout modes (`asymmetrical` | `bento` | `hero`), and Stripe status.
- **`styles.ts`**: Shared styling tokens (`s` object) for responsive typography, card elevations, toggle switches, and hex color normalizers.
- **`GeneralSettingsTab.tsx`**: Studio identity (name, handle/subdomain, custom domain routing) and social links list management.
- **`ThemeSettingsTab.tsx`**: Quick theme overview card, palette attribute breakdown, and deep link to the live Theme Studio.
- **`CommissionTypesTab.tsx`**: Commission types CRUD, section assignment, and active status toggles.
- **`PricingBlocksTab.tsx`**: Commission availability toggle, slot capacity, currency configuration, AI price estimator prompts, and tier categories with media asset uploaders.
- **`IntegrationsTab.tsx`**: Discord webhook alerts, Stripe Express Connect merchant onboarding, live payout/charge status badges, and express dashboard launcher.
- **`SettingsView.tsx`**: Coordinator shell managing tab active states, tenant resolution, API mutation payloads, and toast alerts.

### 2. Commissions (`src/features/commissions/`)

- **`CommissionForm.tsx`**: Multi-step interactive commission submission wizard with live price estimation and dynamic reference attachment handling.
- **`FileUpload.tsx`**: Drag-and-drop reference uploader with client-side canvas compression for high-resolution artwork references.
- **`FormStatus.tsx`**: Status cards indicating open/closed artist capacity and post-submission feedback.
- **`Mascot.tsx`**: Animated interactive mascot with reactive speech bubbles.
- **`Typewriter.tsx`**: Dynamic prompt typewriter animation.

### 3. Dashboard (`src/features/dashboard/`)

- **`CommissionCard.tsx`**: Double-bezel commission card displaying AI analysis estimates, final artist quotes, complexity indicators, and deep-link review actions.
- **`ProfileHeader.tsx`**: User avatar with crop/upload, alias inline editor, verified artist badges, and Payload CMS admin quick access.
- **`DashboardFilters.tsx`**: URL query-synced filter selects for status, commission type, and multi-criteria sorting.
- **`DashboardNav.tsx`**: Mobile-responsive sliding tab bar (`/dashboard`, `/dashboard/library`, `/dashboard/profile`) with gradient scroll hints.
- **`actions/profile-actions.ts`**: Server actions for profile alias mutations and social login registration completion.

### 4. Digital Shop (`src/features/shop/`)

- **`ShopItem.tsx`**: Digital product card with double-bezel wrapper, badge indicators (`Course` | `Bundle` | `Character Base`), price formatting, and hover transitions.
- **`ShopList.tsx`**: Asymmetrical bento grid product catalog with category filter pills and responsive layout rhythm.
- **`SecurePDFViewer.tsx`**: High-performance multi-page PDF reader with intersection-observer ticketed rendering, RAM management, dynamic DPR scaling, and DRM watermarking.
- **`SecurePDFViewerClient.tsx`**: SSR-disabled dynamic client loader for the PDF engine.

### 5. Tipping (`src/features/tipping/`)

- **`TipButton.tsx`**: Heart-accented action button routing users to the creator tip page.
- **`TipAmountSelector.tsx`**: Preset denomination selector (€3, €5, €10, €20) with custom amount validation (min €1.00) and keyboard state adaptation.
- **`services/tipService.ts`**: Service initiating Stripe destination checkouts with analytics event tracking (`CHECKOUT_STARTED`, `CHECKOUT_FAILED`).

---

## Backward Compatibility & Re-export Strategy

To prevent breaking existing Payload CMS admin import maps (`importMap.js`), legacy tests, or external references, legacy paths maintain clean re-exports pointing to the new feature modules:

| Domain | Directory Path | Key Components & Actions | Legacy Paths Preserved |
|---|---|---|---|
| **Studio Settings** | [`src/features/studio-settings/`](file:///Users/eugenepucko/Projects/artpack/src/features/studio-settings/) | `SettingsView`, `GeneralSettingsTab`, `ThemeSettingsTab`, `CommissionTypesTab`, `PricingBlocksTab`, `IntegrationsTab`, `styles`, `types` | `src/components/admin/SettingsView.tsx` |
| **Commissions** | [`src/features/commissions/`](file:///Users/eugenepucko/Projects/artpack/src/features/commissions/) | `CommissionForm`, `ProjectTierSelector`, `AuthGatePrompt`, `Typewriter`, `FileUpload`, `FormStatus`, `Mascot`, `CommissionDetailContent`, `CommissionHeader`, `CommissionDetailsCard`, `ArtistQuoteBlock`, `AIAnalysisBlock`, `CommissionSidebar`, `ArtistReviewForm`, `VerifiedQuoteView`, `PaymentSuccessHandler`, `RejectButton`, `RetriggerAIButton`, `commission-review.action.ts` | `src/app/(frontend)/CommissionForm.tsx`, `src/components/commission-form/*`, `src/app/(app)/dashboard/commission/[id]/*` |
| **Onboarding** | [`src/features/onboarding/`](file:///Users/eugenepucko/Projects/artpack/src/features/onboarding/) | `OnboardingModal`, `OnboardingProgressBar`, `OnboardingFooter`, `useOnboardingForm`, `StepIdentity`, `StepAesthetic`, `StepMascot`, `StepOfferings`, `StepGenerating`, `StepLaunch`, `artist-onboarding.action.ts` | `src/components/platform/ArtistOnboardingModal.tsx` |
| **Dashboard** | [`src/features/dashboard/`](file:///Users/eugenepucko/Projects/artpack/src/features/dashboard/) | `CommissionCard`, `ProfileHeader`, `DashboardFilters`, `DashboardNav`, `profile-actions.ts` | `src/app/(app)/dashboard/*` |
| **Shop** | [`src/features/shop/`](file:///Users/eugenepucko/Projects/artpack/src/features/shop/) | `ShopItem`, `ShopList`, `SecurePDFViewer`, `SecurePDFViewerClient` | `src/features/shop/components/ShopItem.tsx`, `ShopList.tsx`, `SecurePDFViewer.tsx`, `SecurePDFViewerClient.tsx` |
| **Tipping** | [`src/features/tipping/`](file:///Users/eugenepucko/Projects/artpack/src/features/tipping/) | `TipButton`, `TipAmountSelector`, `tipService.ts` | `src/features/tipping/components/TipButton.tsx`, `TipAmountSelector.tsx`, `src/features/tipping/services/tipService.ts` |
| **Queue** | [`src/features/queue/`](file:///Users/eugenepucko/Projects/artpack/src/features/queue/) | `QueueView`, `KanbanCard`, `DraggableCard`, `SortableColumn`, `CommissionStatusToggle`, `QueueContent`, `CommentsSection`, `CommentsWrapper`, `RoadmapSection`, `NotifyButton`, `queue-actions.ts`, `comment-actions.ts` | `src/app/(frontend)/queue/*`, `src/app/(frontend)/queue/[id]/*` |
| **Auth** | [`src/features/auth/`](file:///Users/eugenepucko/Projects/artpack/src/features/auth/) | `LoginForm`, `RegisterForm`, `SocialConfirmForm`, `SocialLogins`, `ConsentCheckboxes`, `auth-actions.ts` | `src/app/(frontend)/login/*`, `src/app/(frontend)/register/*`, `src/components/SocialLogins.tsx`, `src/components/ConsentCheckboxes.tsx`, `src/app/(frontend)/auth-actions.ts` |
| **Pricing** | [`src/features/pricing/`](file:///Users/eugenepucko/Projects/artpack/src/features/pricing/) | `PriceList` + `layouts/` (`HeroLayout`, `ShowcaseLayout`, `PanelLayout`, `CompactLayout`, `ListLayout`, `CategoryGallery`, `MobileItemCarousel`), `ComplexityBadge`, `ApplySlotButton` | `src/features/pricing/components/PriceList.tsx`, `CharacterDrawingTiers.tsx`, `CustomsCarousel.tsx`, `ComplexityBadge.tsx`, `ApplySlotButton.tsx` |

---

## 4. Next.js Client / Server Boundary Rules in Feature Barrels

To prevent Next.js / Turbopack client bundler errors (such as pulling `node:fs` or `payload` into the browser bundle):
1. **Client-Facing Barrel (`index.ts`)**: Only exports Client Components and client-callable server actions.
2. **Server-Facing Entry (`server.ts`)**: Exports async Server Components that directly invoke `getPayload` or `@payload-config` (e.g., `QueueContent`, `CommentsWrapper`, `CommissionDetailContent`).
3. **Action Re-exports with `'use server'`**: Files using `'use server'` must re-export named actions explicitly (e.g. `export { loginAction, ... } from '...'`) rather than wildcard `export * from '...'` to satisfy Next.js compiler static export analysis.

## Verification & Debug Reference

```bash
# Validate TypeScript correctness across all features
npx tsc --noEmit

# Run integration tests
npm run test:int

# Run Playwright E2E tests
npx playwright test tests/e2e/artist-onboarding.e2e.spec.ts

# Regenerate Payload Types & Import Map (when editing configs)
npm run generate:types
npm run generate:importmap
```
