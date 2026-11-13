---
status: plan
collections: [pages, products, queue]
features: []
reviewed: true
---

# Caching Strategy & Scaling Plan (Vercel)

Status: decided, implementation pending
Date: 2026-09-09
Scope: Next.js 16 `use cache` (`cacheComponents: true`), multi-tenant 2monke on Vercel

## Decision

Use **Upstash Redis (regional) + `@neshca/cache-handler`** as the shared cache
backend for Next.js `use cache`. Reasons:

- **ElastiCache is architecturally incompatible**: it lives in an AWS VPC and
  Vercel serverless functions are not in that VPC. Revisit only if hosting
  migrates off Vercel.
- **Status quo is a correctness problem**: default handler = 50 MB in-memory
  LRU per lambda, `revalidateTag` is process-local, cache dies on every deploy.
  Multi-tenant staleness + post-deploy DB thundering herd.
- **Upstash fits**: serverless-native (REST, no VPC), already used in the
  project (`src/lib/security/rate-limit.ts`, QStash email queue), region co-locatable
  with Vercel functions, billing scales with traffic.

Before buying anything: verify whether Vercel's platform cache for Next 16
`use cache` is already active for the project (it may make the custom handler
unnecessary). If not set, proceed with Upstash.

## Implementation steps

1. Add `@neshca/cache-handler` (v1.9.0) to `next.config.mjs` under
   `cacheHandlers.default`, using existing env vars
   `UPSTASH_REDIS_REST_URL` / `UPSTASH_REDIS_REST_TOKEN`, key prefix
   `use-cache:`, **tiered mode** (lambda-local L1 + Redis L2) so warm-function
   repeat hits never cost a Redis command.
2. Keep `cacheMaxMemorySize` small — it becomes the L1 only.
3. Create a **regional** (not Global) Upstash DB in the same region as the
   Vercel functions. Go Global only together with multi-region functions +
   DB read replicas.
4. Bump the 9 `cacheLife('seconds')` functions in `src/lib/cache.ts` to
   `minutes` / `hours` — freshness already comes from
   `safeRevalidateTag` / `revalidateTenantTag` on writes; 1s revalidation is
   pure DB churn. Biggest command-volume win.
5. Do not cache long-tail tenants — their entries only evict busy ones and
   burn Redis commands.

## Product sources: catalogue window vs. curated selection

The shop surfaces read products from two different sources on purpose, and mixing them
up is a silent data bug:

- **Catalogue window** — `getCachedProducts()` (`features/shop/cache/shop.cache.ts`):
  100 rows, `sort: '-price'`. Feeds `shopCatalog` and `latestProducts` ("newest N"
only needs a window). It must **not** be used to resolve a hand-picked selection.
- **Curated selection** — a `featuredProducts` block carries its own documents. Pages
  are read at depth 2 (`page.action.ts`), so `block.products` arrives populated, in the
  artist's order, with no cap. `resolveFeaturedProducts` uses those documents first;
  ids that arrive unpopulated (live preview, a shallower read) are fetched by
  `getCachedProductsByIds(ids, subdomain)` — one query, capped by the id set, scoped to
  the studio, cached under `tenant_<id>:products` with the id set as part of the cache
  key (one entry per distinct selection).

The same accessor backs bundle contents (`getProductsByIds`, tenant required), which
previously queried by bare id — ids are only unique inside a studio, so that path could
return another tenant's row.

## Cost levers (order of impact)

1. Profile bump `seconds` -> `minutes`/`hours`.
2. Tiered cache handler (L1 absorbs repeat hits).
3. Skip caching for low-traffic tenants.
4. Watch bandwidth on large cached payloads (avoid caching depth-2 studio
   objects when only a few fields are needed).

## Scaling sequence on Vercel (do in order)

1. Redis cache handler + profile bumps (this doc).
2. Co-locate regions: Vercel functions, Upstash Redis, Postgres.
3. DB connection pooler (Neon/Supabase/PgBouncer) — many lambdas x Postgres
   connections is the real bottleneck. (`cpus: 4` / `workerThreads: false` in
   `next.config.mjs` only apply self-hosted; Vercel ignores them.)
4. Fix uncacheable hot paths (see below).
5. Then scale horizontally: Fluid compute, higher concurrency, `maxDuration`
   for streaming routes, multi-region only with read replicas.

## Uncachable hot paths (separate fixes)

1. **Comments poller** (hot spot #1): 5s `fetch('/api/comments/cache/[id]')`
   per open discussion; unauthenticated; refetches the full list on every
   cycle. **Implemented 2026-09-09**: write-side version stamp in
   `postCommentAction` (`comments:version:<queueItemId>` incr), short-lived
   poll ticket minted by `CommentsWrapper` after auth checks, poll route
   validates ticket + version and returns `{ unchanged }` when nothing
   changed, client skips re-render on unchanged, adaptive backoff
   (5s -> 10s -> 20s), pauses on hidden tab and closed dock, `router.refresh()`
   on ticket expiry. Target: ~80-90% reduction in Redis commands + bandwidth,
   and closes the unauthenticated read.
2. **Deliverable proxy** (`/api/deliverables/[id]`): per-file auth + up to 3
   DB queries + S3 stream through a serverless function. Long streams also
   risk Vercel duration limits. Long-term: presigned S3 URLs (redirect);
   keep the proxy only for auth-gated streaming.
3. **Auth chain** (`getAuthenticatedUser` / `getCachedUser`): DB session
   lookup per call; consider Redis session caching.
