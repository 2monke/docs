---
status: shipped
collections: [artists, commission-settings, commissions, customers, notification-settings, pages, products, queue, sprints, studios, themes]
features: [auth, onboarding, studios]
---


# Authentication, Dual Account Model & Artist Onboarding

> **Status**: Shipped

How customer (`customers`) and artist/admin (`users` + `artists`) accounts authenticate, verify email addresses, reset passwords, transfer sessions across subdomains, and provision new studios through `/onboarding`.

## Dual Account Architecture (`customers` vs. `users` & `artists`)

2monke separates buyers and studio members into distinct Payload auth collections:

- **Customers (`src/collections/Customers.ts`)**: Buyers who submit commissions, purchase digital shop products, join chat rooms, and leave reviews.
- **Studio Users (`src/collections/Users.ts`) & Artist Profiles (`src/collections/Artists.ts`)**: Studio owners and team members authenticate against `users` (which carries `roles` and `tenants`) while public bio, avatar, and social links live in `artists`, linked to `src/collections/Studios.ts` (`src/features/studios`).

## Login, Turnstile & Cross-Host Session Transfer

`loginAction` (`src/features/auth/actions/auth-actions.ts`) handles sign-in for both account types from a single login form:

1. **Security Guards**: Every login attempt runs `apiRateLimiter.limit(ip)` and Cloudflare Turnstile verification (`verifyTurnstileToken` in `src/features/auth/services/turnstile.service.ts`) in parallel before touching the database.
2. **Collection Fallback**: Attempts `loginUser` against `customers` first; if that fails, tries `users`. On success, sets the HTTP-only `payload-token` cookie (7-day expiry) scoped via `resolveAuthCookieDomain` (`src/lib/tenant/cookie-domain.ts`).
3. **Post-Login Routing & Cross-Subdomain Transfer**:
   - Customers redirect to their validated `redirect` query parameter or `/dashboard`.
   - Artists with multiple studios redirect to `/studios`.
   - Artists with one studio who log in from the platform host (`artpack.me`) are redirected via `buildStudioSessionTransferUrl` (`src/features/auth/actions/session-transfer-token.ts`) to their studio subdomain (`<studio>.artpack.me` or custom domain). Draft studios (`profileStatus === 'draft'`) land directly on `/admin` (`resolveSessionTransferLandingPath` in `src/features/auth/actions/session-transfer-landing.ts`).

## Email Verification & Password Reset

- **Email Verification (`/verify`)**: New customer signups receive a verification token verified at `/verify?token=...`.
- **Password Recovery (`/forgot-password` & `/reset-password`)**: Handled by `forgotPasswordOperation` and `resetPasswordOperation` (`src/features/auth/actions/password-reset.action.ts`), which check both `customers` and `users` so either account type can reset their password from studio or platform hosts.

## Artist Onboarding Wizard (`/onboarding`)

New artists create their studio through the `/onboarding` flow orchestrated by `src/features/onboarding/actions/artist-onboarding.action.ts`:

- Validates studio name, handle (`subdomain` against `RESERVED_SUBDOMAINS`), and Turnstile token.
- Creates or links the `users` account and provisions the tenant ecosystem in a coordinated sequence:
  1. `studios` record initialized with `profileStatus: 'draft'` (published automatically once Stripe Connect reports `charges_enabled`).
  2. `artists` public profile (`src/collections/Artists.ts`) and default `themes` document (`src/collections/Themes.ts`).
  3. `commission-settings` (`src/collections/CommissionSettings.ts`) with initial categories/tiers and `notification-settings` (`src/collections/NotificationSettings.ts`).
  4. System `pages` (`src/collections/Pages.ts`: `home`, `prices`, `shop`, `queue`, `tos`, `privacy-policy`) and an initial `sprints` commission round (`src/collections/Sprints.ts`).
