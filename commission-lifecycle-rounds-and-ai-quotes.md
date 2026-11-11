---
status: shipped
collections: [artists, commission-settings, commissions, media, queue, raw-media, sprints, studios]
features: [commissions, queue]
---


# Commission Lifecycle, Rounds, AI Quotes & Deliverables

> **Status**: Shipped

End-to-end reference for how commission rounds (`sprints`), customer submissions (`commissions`), AI price estimation, artist quote reviews, queue progress tracking (`queue`), and final artwork delivery operate in ArtPack.

## Commission Rounds (`sprints`) & Slot Limits

Commission availability on a studio storefront is controlled by the `sprints` collection (`src/collections/Sprints.ts`), managed by artists in `/admin/commission-settings` under the **Rounds** tab.

- **Single Active Round Invariant**: Each studio can have at most one round with `status: 'active'` at a time (`upcoming`, `active`, `completed`).
- **Public Gate (`acceptingCommissions`)**: A customer can only submit a commission when the studio has an `active` round with `acceptingCommissions: true`. Otherwise, `submitCommissionAction` (`src/features/commissions/actions/submit-commission.action.ts`) rejects submissions with `"Commissions are currently closed. No active sprint is accepting new requests."`
- **Slot Reservations (`maxSlots` & `maxRequestsPerCustomer`)**:
  - `maxRequestsPerCustomer` limits how many active requests a single customer can hold in the round (defaults to `1` on the collection schema, or `3` fallback in `submitCommissionAction` when unset).
  - `maxSlots` caps total capacity for the round (defaults to `5`).
  - Both limits are reserved atomically via `reserveCustomerSlot` and `reserveSprintSlot` (`src/features/queue/actions/reserve-slot.action.ts`) after reference media keys are validated, and are rolled back automatically if submission fails or when an artist declines a request (`rejectCommission`).

## Commission Status Lifecycle

Every document in `src/collections/Commissions.ts` transitions through the following `status` values:

1. `pending_ai` — Initial state on customer submission. Reference images are uploaded and initial tier selections are recorded while background AI complexity analysis runs.
2. `needs_artist_review` — Set once `estimateCommission` (`src/lib/ai/estimateCommission.ts`) finishes populating `aiAnalysis` (`estimatedPrice`, `complexityLevel`, `reasoning`, `costBreakdown`).
3. `approved` (and legacy `awaiting_payment`) — The artist has reviewed the request via `saveArtistReview` (`src/features/commissions/actions/commission-review.action.ts`), finalized `artistReview.finalPrice` and `artistReview.costBreakdown`, and sent the quote to the customer.
4. `paid` — The customer has paid the approved quote via Stripe Checkout (`src/features/commissions/actions/create-commission-checkout.action.ts`), placing the commission on the active work queue.
5. `completed` — The artist has uploaded the finished artwork deliverables and finalized the commission (`finalizeCommissionAction` in `src/features/queue/actions/deliverable-actions.ts`).
6. `rejected` — Declined by the artist (`rejectCommission`); releases the customer and round slot reservations and removes any linked queue item.
7. `closed_by_customer` — Closed by the customer from their waiting-for-payment reminder link (`/dashboard/commission/[id]/close`), recording an optional `cancelReason` visible to the artist.

## AI Price Estimation & Artist Quote Review

- **Configured Currency & Pricing Layouts (`src/collections/CommissionSettings.ts`)**:
  - Studios choose their operating `currency` in Commission Settings: `USD ($)` (default), `EUR (€)`, `GBP (£)`, `JPY (¥)`, `CAD (C$)`, or `AUD (A$)`.
  - Each commission category (`commissionTypes`) defines pricing `sections` with an optional `/prices` page `layout`: `cards` (Big cards), `showcase` (List with gallery), `panel` (Color panel), `compact` (Compact grid), or `list` (Simple list). Leaving `layout` empty hides that section from `/prices` while keeping it selectable in the commission form.
  - Artists can also customize `defaultMaxSlots` (default `5`), `defaultMaxRequestsPerCustomer` (default `1`), `checklistTemplate`, and per-type `aiPromptGuidelines`.
- **AI Analysis**: When a commission is created, `triggerAIAnalysisAfterChange` evaluates the customer's reference images and selected tiers (`tierSelections`) against the studio's `src/collections/CommissionSettings.ts` pricing rules.
- **Manual Retrigger Quota**: Artists can re-run the AI estimate from the review screen via `retriggerAIEstimate` (`src/features/commissions/actions/commission-review.action.ts`) only while `status` is `pending_ai` or `needs_artist_review`. Retriggers are rate-limited to **15 calls per hour per artist** (`aiEstimateRetriggerRateLimiter`).
- **Quote Validation**: In `saveArtistReview`, `finalPrice` must be between `0.01` and `1,000,000`, and every line item in `costBreakdown` requires a non-empty `item` description and a non-negative `cost`. Once saved, the status moves to `approved` and the customer receives a quote ready email.

## Queue Progress, Checklists & Final Deliverables

- **Queue Tracking (`src/collections/Queue.ts`)**: Paid commissions appear on the studio's `/queue` page with status `waiting`, `wip` (Work In Progress), or `completed`.
- **Milestone Checklist**: Each queue card tracks `progress` (`0`–`100%`), auto-calculated from its `checklist` steps (defaults: `Sketch`, `Line`, `Color`, `Shading`, `Shadows`, `Background`, `Render`).
- **Deliverable Finalization (`src/features/queue/actions/deliverable-actions.ts`)**:
  - Artists upload high-resolution deliverables into the private `raw-media` collection (`createRawMediaDeliverableAction`, stored under `tenants/{id}/` in S3 with `isPrivate: true`).
  - Calling `finalizeCommissionAction` atomically sets `commissions.status = 'completed'`, attaches `deliverableFiles`, sets `queue.status = 'completed'` and `progress = 100`, generates a single-use SHA-256 `reviewToken` (`reviewStatus: 'invited'`), and emails the customer their download and review link.
