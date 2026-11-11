---
status: shipped
collections: [help-gaps]
features: [help]
---

# Customer Help Assistant, Audience Isolation & Gap Queue

Customer-facing AI help assistant for artists (studio owners) and buyers (commissioners and collectors), featuring strict audience isolation, verbatim citation verification, and an automated gap queue for missing documentation.

## Scope & Purpose

ArtPack provides two distinct customer journeys:
1. **Buyers / Art Collectors**: Requesting commissions, monitoring progress, purchasing digital goods from studio shops, leaving tips, and submitting verified reviews.
2. **Artists / Studio Owners**: Managing commission sprints, setting pricing tiers, onboarding Stripe Connect for bank payouts, moderating reviews, and customizing themes.

The customer help system answers questions for both audiences from a curated, public-safe markdown corpus in `help/` (~3,000 tokens), structurally decoupled from the team's internal documentation in `docs/` (~200,000 tokens). Any unanswered or low-confidence inquiry is caught by the system, gracefully offered support handoff, and logged into the `help-gaps` collection for triage and documentation expansion.

## Architecture

```
User Query (In-App Widget, Public Discord, or CLI)
  │
  ▼
src/features/help/actions/ask-help.action.ts
  │
  ├─ 1. Account-Specific Guard (Regex / Keyword Classifier)
  │     ├─ Matches: "where is my order", "my payout", "cm_8f9x...", "refund my"
  │     └─ Route: Returns polite support / dashboard lookup guidance immediately
  │               Logs gap with reason 'account_specific'
  │
  ├─ 2. Audience-Scoped Help Retrieval (src/lib/ai/help)
  │     ├─ Target: 'buyer'  → Loads help/buyers/*.md + help/shared/*.md
  │     └─ Target: 'artist' → Loads help/artists/*.md + help/shared/*.md
  │
  ├─ 3. Lexical TF-IDF Pre-Search (search.ts)
  │     └─ Evaluates query terms & identifiers; top hits attached as evidence
  │
  ├─ 4. Generative Answering (engine.ts & prompt.ts)
  │     └─ Calls Gemini with CustomerAnswerSchema
  │        Enforces: friendly 2-4 sentences, verbatim quote citations, zero code leaks
  │
  ├─ 5. Answer Validation & Leak Prevention (validate.ts)
  │     ├─ Verifies: citations resolve, quotes are verbatim substrings in cited section
  │     ├─ Blocks: src/, payload.config, collections/, .ts, internal table names
  │     └─ On Failure: Downgrades to honest "I don't know" + support recommendation
  │
  └─ 6. Automated Gap Queue Logging (log-help-gap.action.ts)
        ├─ If grade !== 'documented' (absent, unknown, downgraded, account_specific)
        ├─ Normalizes text & computes SHA-256 questionHash for deduplication
        ├─ Strips PII (emails & phone numbers)
        └─ Creates or increments count on matching record in 'help-gaps' collection
```

## Audience Isolation Model

The help corpus is physically partitioned to prevent leaking artist configuration rules to buyers or buyer instructions to artists:

| Folder | Audience | Topics Covered | Visible To |
|---|---|---|---|
| `help/buyers/` | `buyer` | Commission lifecycle, digital downloads, payments, refunds, tipping, leaving reviews | Buyers only |
| `help/artists/` | `artist` | Studio settings, Stripe Connect onboarding, sprint quotas, pricing tiers, review moderation | Artists only |
| `help/shared/` | `shared` | Dual-account model, password recovery, contacting platform support | Both audiences |

Audience resolution occurs on the server side:
- Logged-in `customers` and anonymous studio storefront visitors resolve to `buyer`.
- Logged-in `users` with artist/admin roles resolve to `artist`.
- The Discord bot resolves audience based on the active channel (`#artist-help` vs. `#buyer-help`).

## Verification & Leak Prevention

Unlike general chat bots that speculate or produce plausible hallucinations, every answer delivered by the help assistant must satisfy strict verification invariants:

1. **Verbatim Quotation**: For any `documented` answer, the cited quote must be an exact substring of the cited section (checked via `quoteAppearsIn`).
2. **Reverse Leak Defense**: The validator screens answers against internal technical patterns (`src/`, `payload.config`, `collections/`, `.ts`, `.tsx`, database query symbols). If any technical identifier appears, the answer is immediately downgraded.
3. **Escalation Mandate**: Answers graded `absent`, `unknown`, `downgraded`, or `account_specific` must set `canEscalate: true` and provide a customer-facing support reason.

## Payload `help-gaps` Collection

Unanswered inquiries are stored in the database for team triage:

- **Slug**: `help-gaps`
- **Primary Key**: Text prefixed ID (`gap_...`)
- **Deduplication**: `questionHash` (SHA-256 of lowercase, stripped question text).
- **Frequency Counter**: `count` (incremented every time a client asks the same or semantically identical question).
- **Access Control**: Admin-only read, update, delete (`isAdmin`). Server Actions write with `overrideAccess: true`.
- **Fields**:
  - `question`: Sanitized question string with emails replaced by `[email]` and phone numbers by `[phone]`.
  - `questionHash`: Unique indexed hash.
  - `audience`: `'buyer' | 'artist' | 'shared'`.
  - `reason`: `'absent' | 'unknown' | 'downgraded' | 'account_specific'`.
  - `docsGap`: Suggested article topic identified by the model.
  - `searchHits`: Top matching search sections at time of query.
  - `count`: Number of occurrences.
  - `status`: `'new' | 'drafting' | 'in_review' | 'resolved' | 'wont_fix'`.
  - `contactEmail`: Optional customer email provided for support followup.
  - `source`: `'widget' | 'discord' | 'cli'`.
  - `lastAskedAt`: Timestamp of the latest occurrence.

## CLI Commands

```bash
# Verify integrity of all help articles, frontmatter, and token counts
yarn help:check

# Test a buyer query with dry-run evaluation
yarn help:ask "how do I download my purchased files?" --as buyer --dry-run

# Test an artist query
yarn help:ask "how do I connect stripe for payouts?" --as artist --dry-run
```

## Component Breakdown & Files Changed

| Path | Purpose |
|---|---|
| `help/{buyers,artists,shared}/*.md` | Customer-facing help articles with frontmatter metadata |
| `src/collections/HelpGaps.ts` | Payload CMS collection config for logging documentation gaps |
| `src/lib/utils/id.ts` | Added `helpGap: 'gap_'` ID prefix |
| `src/payload.config.ts` | Registered `HelpGaps` in `withAdminLeaveGuard` |
| `src/lib/ai/help/types.ts` | Type definitions for chunks, citations, answers, and search hits |
| `src/lib/ai/help/text.ts` | Token estimation, heading normalization, and quote substring matching |
| `src/lib/ai/help/corpus.ts` | Help markdown parser, frontmatter validator, and audience filtering |
| `src/lib/ai/help/search.ts` | Lexical TF-IDF search engine scoped by audience |
| `src/lib/ai/help/prompt.ts` | Customer system prompt, Zod schema, and context serializer |
| `src/lib/ai/help/validate.ts` | Verbatim quote validator, internal leak defense, and downgrader |
| `src/lib/ai/help/engine.ts` | Answering pipeline connecting corpus, search, model, and validation |
| `src/lib/ai/help/index.ts` | Barrel export for help library |
| `src/lib/ai/index.ts` | Domain barrel export for `src/lib/ai` |
| `src/features/help/actions/log-help-gap.action.ts` | Deduplicating, PII-scrubbed gap logging action |
| `src/features/help/actions/get-help-gaps.action.ts` | Admin query action for help gaps |
| `src/features/help/actions/ask-help.action.ts` | Orchestrator with account-specific filter and gap logging |
| `src/features/help/index.ts` | Barrel export for help feature actions |
| `scripts/help.ts` | CLI runner for `yarn help:check` and `yarn help:ask` |
| `package.json` | Added `help:check` and `help:ask` run scripts |
| `tests/int/help-assistant.int.spec.ts` | 12 automated integration tests verifying audience isolation, search, validation, and actions |

## Debug & Verification Reference

```bash
# Run unit and integration tests for customer help assistant
yarn test:int tests/int/help-assistant.int.spec.ts

# Ensure repository-wide ask pipeline tests remain green
yarn test:int tests/int/ask-pipeline.int.spec.ts

# Verify type safety across the project
./node_modules/.bin/tsc --noEmit
```
