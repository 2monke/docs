# Legal Consent (Terms & Privacy)

Two different policy relationships exist on the platform, and the code keeps
them apart on purpose:

- **Platform documents** — 2monke's own Terms of Service and Privacy Policy,
  stored in the `terms-of-service` / `privacy-policy` globals and served at
  `/tos` and `/privacy-policy` on the platform root.
- **Studio documents** — each studio's own Terms and Privacy pages in the
  `pages` collection, served at `/tos` and `/privacy-policy` on that studio's
  host.

A customer who creates an account accepts the platform documents. A customer
who applies for a commission or buys from a shop accepts the studio's documents
— or, when the studio never published them, the platform ones.

## The rule: a consent record names the document that existed

`src/features/auth/lib/legal-consent.ts` is the single contract:

```ts
interface LegalConsentRecord {
  type: 'terms' | 'privacy'
  scope: 'platform' | 'studio' // which document was accepted
  version: string | null       // the document's own last-published stamp
  acceptedAt: string
}
```

- `version` is **never invented**. A missing version is `null`, not the
  acceptance date: "we recorded a version" must mean a version existed.
- `scope` is per document, not per form. A studio that published Terms but not
  a Privacy page produces one `studio` record and one `platform` fallback.

`resolveConsentDocuments(tenantId)` (`src/features/studios/actions/get-consent-documents.action.ts`)
decides the documents for a transaction: the studio's page `updatedAt` when the
page is published and visible, otherwise the platform global's `lastUpdated`,
with `scope` set accordingly. Every purchase/application surface uses the same
resolver, so the checkbox links and the audit record cannot disagree about which
document was accepted.

## Where records land

| Flow | Written to | Built by |
|---|---|---|
| Customer registration | `customers.consents` | `buildPlatformConsentRecords` |
| Artist onboarding | `users.consents` | `buildPlatformConsentRecords` |
| Social sign-up / profile name | `users.consents` / `customers.consents` | `buildPlatformConsentRecords` |
| Commission application | `commissions.studioConsents` | `buildStudioConsentRecords` |
| Shop purchase | `orders.studioConsents` | `buildStudioConsentRecords` |

Both `studioConsents` columns are `jsonb` holding `LegalConsentRecord[]` and are
admin-only fields. The writes are **system writes** (`overrideAccess: true`):
`submitCommissionAction` passes it explicitly, and `fulfillShopOrderFromSession`
already did. Tightening either to `overrideAccess: false` would silently strip
the audit record — `tests/int/shop-fulfillment-consent.int.spec.ts` pins the
order path and the explicit option.

### Why shop consent travels through Stripe

Two paths fulfil a paid order — the success-page render and the
`checkout.session.completed` webhook — and either may win the race. The consent
the customer gave at the moment of purchase is packed into the Checkout session
metadata (`studioLegalAcceptedAt`, `studioTermsVersion`, `studioPrivacyVersion`,
`studioTermsScope`, `studioPrivacyScope`), and the winning fulfilment stamps it
onto the order. A retry finds the existing order and never overwrites the
record. Sessions created before consent tracking simply produce no field.

## UI

`ConsentCheckboxes` (`src/features/auth/components/ConsentCheckboxes.tsx`) is
shared by registration, onboarding, social confirm, the commission form, the
product sidebar and the buy-now dialog.

- `scope` selects the link targets: platform URLs are absolute (platform root),
  studio URLs are relative to the current studio host.
- Server surfaces decide the scope from published pages
  (`getCachedPage('tos')` / `getCachedPage('privacy-policy')`) so an
  unconfigured studio's checkbox links to the platform policy it actually falls
  back to. The `commissionForm` **block** cannot await those reads (it renders in
  the client tree) and defaults to `studio`; the dedicated `/apply` route passes
  the real scope.
- The box is a `role="checkbox"` container with the policy link nested inside.
  Keydown is owned by the element that received it: Enter/Space on the link
  follows the link, never toggles consent
  (`tests/int/consent-checkboxes-keyboard.int.spec.ts`).
- `compact` tightens spacing for the shop sidebar and dialog; a generic `error`
  marks both boxes and is announced with `role="alert"`.

## Tips are deliberately exempt

A tip buys no goods and grants no licence, so there is no studio policy to
accept. The reasoning is recorded at `createTipCheckoutSession`; if a tip ever
delivers something, it needs the same resolver as shop orders.

## Verification

```bash
yarn test:int tests/int/legal-consent.int.spec.ts \
  tests/int/consent-checkboxes-keyboard.int.spec.ts \
  tests/int/consent-documents.int.spec.ts \
  tests/int/shop-fulfillment-consent.int.spec.ts \
  tests/int/platform-fee-checkout-wiring.int.spec.ts
./node_modules/.bin/tsc --noEmit
```

## Migration

`20261122_143000_add_studio_consents` adds `commissions.studio_consents` and
`orders.studio_consents` (jsonb, `IF NOT EXISTS` on both directions, so it is
idempotent). Applied to a database that already has the columns it is a no-op.
