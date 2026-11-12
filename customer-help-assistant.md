---
status: shipped
features: [help]
---

# Customer Help Assistant

Customer-facing AI help assistant for artists (studio owners) and buyers
(commissioners and collectors), with strict audience isolation and verbatim
citation verification.

**Answering lives in a separate service.** The corpus, retrieval, model call, and
validation are in [`2monke/help-service`](https://github.com/2monke/help-service),
a Cloudflare Worker. This repository keeps only a proxy and the UI.

## Scope & Purpose

ArtPack provides two distinct customer journeys:

1. **Buyers / Art Collectors**: requesting commissions, monitoring progress,
   purchasing digital goods from studio shops, leaving tips, and submitting
   verified reviews.
2. **Artists / Studio Owners**: managing commission sprints, setting pricing
   tiers, onboarding Stripe Connect for bank payouts, moderating reviews, and
   customizing themes.

The assistant answers for both audiences from a curated, public-safe markdown
corpus (~3,000 tokens), structurally decoupled from the team's internal
documentation (~200,000 tokens). The separation matters: everything in `docs/`
is written for engineers and is not safe to quote to a customer.

## Architecture

```
browser (in-app widget) ─┐
                         ├─▶ POST /api/help/ask/service ─▶ Gemini
discord-gateway ─────────┘        (HMAC service token)
                              [2monke/help-service, a Worker]
        ▲
        │ POST /api/help/ask  (proxy)
        │
2monke-app
```

Inside the Worker:

```
  ├─ 1. Account-Specific Guard (regex classifier)
  │     ├─ Matches: "where is my order", "my payout", "cm_8f9x…", "refund my"
  │     └─ Returns support/dashboard guidance immediately, without a model call
  │
  ├─ 2. Audience-Scoped Retrieval
  │     ├─ Target: 'buyer'  → buyers/*.md + shared/*.md
  │     └─ Target: 'artist' → artists/*.md + shared/*.md
  │
  ├─ 3. Lexical TF-IDF Pre-Search
  │     └─ Query terms & identifiers; top hits attached as evidence
  │
  ├─ 4. Generative Answering
  │     └─ Gemini with a Zod schema. 2–4 sentences, verbatim quotes, no code leaks
  │
  ├─ 5. Validation & Leak Prevention
  │     ├─ Citations must resolve; quotes must be verbatim substrings
  │     ├─ Blocks: src/, payload.config, collections/, .ts, internal table names
  │     └─ On failure: downgrades to an honest "I don't know" + support handoff
  │
  └─ 6. Unanswered questions logged as structured JSON
```

## Why answering is not in this application

It used to be. The corpus was a `docs` submodule read with `fs` at request time
and traced into the deployment artifact, so fixing a sentence in a help article
meant a docs commit, a submodule bump, and a full application rebuild. A
submodule pins; pinning is the opposite of independent deploys.

Moving answering out made the corpus build input for one worker. Editing an
article is one deploy of that service, and this application is untouched.

## What stays here, and why

| Path | Why it cannot move |
|---|---|
| `src/app/api/help/ask/route.ts` | Proxy. The HMAC secret must not reach a browser, and the audience comes from the Payload session, which the service cannot see. Also the only component that knows *who* is asking, so the per-user rate limit lives here. |
| `src/app/api/discord/interactions/route.ts` | Proxy. Keeps Ed25519 signature verification in the app; Discord calls this URL directly. |
| `src/features/help/components/HelpWidget.tsx` | React UI inside the dashboard and storefront layouts. |
| `src/lib/help/client.ts` | The signed client. Classifies failures: an outage degrades to a "contact support" answer, while a rejected question stays visible instead of hiding a bug. |
| `src/lib/security/service-token.ts` | Mints the HMAC envelope, and holds the shared test vector that keeps three deployments in step. |
| `scripts/help-draft.ts` | Reads *this* repository's admin routes and screen labels, which is why a draft beats a cold model call. |

## Audience Isolation Model

The corpus is physically partitioned to prevent leaking artist configuration
rules to buyers, or buyer instructions to artists:

| Folder | Audience | Topics Covered | Visible To |
|---|---|---|---|
| `corpus/buyers/` | `buyer` | Commission lifecycle, digital downloads, payments, refunds, tipping, reviews | Buyers only |
| `corpus/artists/` | `artist` | Studio settings, Stripe Connect onboarding, sprint quotas, pricing tiers, review moderation | Artists only |
| `corpus/shared/` | `shared` | Dual-account model, password recovery, contacting support | Both audiences |

Audience is resolved **server-side, never by the caller**:

- The in-app proxy derives it from the session: `customers` and anonymous
  storefront visitors are `buyer`; `users` are `artist`.
- The Discord gateway derives it from channel and role, and sends it explicitly.
- The service endpoint **requires** the field and rejects a request without it.
  A default is how artist questions silently start receiving buyer answers.

Nothing in a request body can influence the audience — there is a test for that.

## Verification & Leak Prevention

Every answer must satisfy these invariants:

1. **Verbatim Quotation** — for a `documented` answer, the cited quote must be an
   exact substring of the cited section.
2. **Reverse Leak Defense** — answers are screened against internal technical
   patterns (`src/`, `payload.config`, `collections/`, `.ts`, `.tsx`, database
   query symbols). Any match downgrades the answer immediately.
3. **Escalation Mandate** — `absent`, `unknown`, and `account_specific` answers
   must offer human support.
4. **No model text on failure** — a failed verification returns an honest "I
   don't know", never the model's original answer.

## Unanswered questions are logged, not stored

When the corpus cannot answer, the question is logged as one structured line:

```json
{"event":"help.gap","question":"…","audience":"buyer","reason":"absent",
 "source":"discord","discord":{"channelId":"…"},"searchHits":[/* top 3 */]}
```

Workers Logs and `wrangler tail` collect it. The question is PII-scrubbed first —
emails to `[email]`, phone numbers to `[phone]`.

**There is no gap queue in the current version.** A queue earns its keep when
strangers ask questions and nobody is watching; this project has one test studio
whose owner is the person asking. What the queue cost was real: a database, a
driver, a schema, a migration step, a dedicated role, a bot token, and weekly
cron work — the only deploy blocker the service had.

The version with it is tagged `v0.1-with-gaps` in that repository: a Postgres
store, a triage API with separate intake and admin capabilities, a 15-minute
cron that re-asked old Discord questions and replied in the thread once the
corpus caught up, and a CLI. It was working and tested.

**Restore it** when someone who is not you starts asking questions, or when you
want the list and the logs are no longer good enough.

## Drafting articles

Drafts are generated from this repository's own code facts — real admin routes,
collections, and visible button labels — so they describe the product as it is,
not as a model imagines it.

```bash
# Preview an article without writing anything
yarn help:draft "how do refunds work for a cancelled commission" --as buyer --dry-run

# Write it into the service's corpus (HELP_CORPUS_DIR, default ../help-service/corpus)
yarn help:draft "how do refunds work for a cancelled commission" --as buyer --write
```

The writer is constrained by a prompt and checked by a leak guard before the file
is written: no file paths, no database symbols, no infrastructure names. A draft
lands with `status: generated`, and is only served to customers once a human
changes it to `status: published`.

## Debug & Verification Reference

```bash
# Proxy and client behaviour: audience resolution, degradation, failure classes
yarn test:int tests/int/help-ask-proxy.int.spec.ts tests/int/help-service-client.int.spec.ts

# The HMAC envelope shared by this app, the service, and the gateway
yarn test:int tests/int/help-service-token.int.spec.ts

# Discord interaction handling, including the proxy hop
yarn test:int tests/int/discord-interactions.int.spec.ts

# Is the service up, and which corpus is live?
curl -s https://2monke-help-service.2monke.workers.dev/health

# Type safety
./node_modules/.bin/tsc --noEmit
```

The service's own suite (pipeline, verification, corpus integrity) lives in that
repository.
