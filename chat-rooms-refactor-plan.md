---
status: shipped
collections: [artists, chat-messages, chat-rooms, commissions, customers, pages, queue, studios]
features: [chat, commissions, queue]
---


# Chat Rooms Refactor — Implementation Plan

Status: **Implemented 2026-09-09** — collections, actions, poll route v2,
room-based UI, legacy redirects, and the DDL migration are in place. The
legacy `queue-comments` system was dropped entirely (never used on test) —
no two-release rollout, no backfill needed; the migration recreates an empty
legacy table only in `down()`. Remaining follow-ups: dashboard chat list with
unread badges, commission context chip wiring in the UI, room-scoped customer
notify.
Date: 2026-09-09
Goal: Replace per-commission comment threads (`queue-comments`) with two-party
chat **rooms** — one room per (studio, customer) pair — so discussion history
persists across commissions instead of resetting with each new project.

## 1. Why

Today every `queue-comments` document hangs off a `queueItem`, which is 1:1
with a commission. A new commission = a new queue item = an empty thread, so
the customer and artist lose all previous conversation context. The data model
is "per-project comments", not "chat".

Target model:

- **Room** = one studio (tenant) × one customer. A customer can have one room
  per studio they commissioned. "Two persons" in practice means **two
  parties**: the customer + the studio team (all artists/admins of that
  tenant), consistent with `userHasTenantAccess`.
- **Commissions become context, not partitions**: messages live in the room;
  a message MAY carry an optional `commission` reference used only as a
  display chip ("re: your Octo character") and as an anchor when opening the
  chat from a commission page.

## 2. Current state (verified)

- `QueueComments` collection (hidden): `tenant`, `commentId` (stable client
  id), `queueItem` (→ queue, required), `parent` (→ queue-comments), `author`
  group (id/name/avatar/role), `content` richText (Lexical).
- Access: read via `queueItem.commission.customer` (customers) /
  `ownTenantOnlyQuery` (artists); create — customers verified through
  queueItem → commission.customer; update `false`; delete `ownTenantOnly`.
- Actions: `postCommentAction`, `getCachedComments`, `notifyCustomerAction`,
  `getQueueComments`, `createQueueCommentRecord`, `deleteQueueComments`,
  `deleteQueueCommentsByQueueItem` (wired into Queue `beforeDelete` hook).
- Redis keys: `comments:cache:<queueItemId>`, `comments:version:<queueItemId>`,
  `comments:ticket:<uuid>` (poller fix, implemented 2026-09-09).
- UI: `CommentsWrapper` (SSR merge DB+Redis, mints poll ticket),
  `CommentsSection` (client poll + post), `DiscussionDock` (desktop drawer +
  phone link), `/queue/[id]/discussion` full page, `NotifyButton` →
  `notifyCustomerAction`. Used from `queue/[id]/page.tsx` and
  `CommissionDetailContent.tsx` (dashboard commission review).
- Rate limiting: `commentRateLimiter` (3/60s per IP), link sanitization via
  `src/lib/utils/commentContent.ts` — both stay as-is.

## 3. Target schema

### `chat-rooms` (new collection, slug `chat-rooms`, id prefix `chrm_`)

| Field | Type | Notes |
|---|---|---|
| `tenant` | relationship → studios | required, index; `associateTenant` hook |
| `customer` | relationship → customers | required, index |
| `lastMessageAt` | date | sort key for chat list, set by afterChange hook |
| `unreadCustomer` / `unreadArtist` | number | incremented by hook, zeroed by mark-read action |
| Unique index on `[tenant, customer]` | | prevents duplicate rooms |

Access:
- read: `users` → `ownTenantOnlyQuery`; `customers` → `{ customer: { equals: user.id } }`; guests `false`
- create: `users` only (customers never create rooms directly — the
  post-message action auto-creates the room)
- update/delete: `ownTenantOnly`

### `chat-messages` (new collection, slug `chat-messages`, id prefix `chmsg_`)

| Field | Type | Notes |
|---|---|---|
| `tenant` | relationship → studios | tenant isolation + `ownTenantOnly` mutations |
| `room` | relationship → chat-rooms | required, index |
| `commentId` | text | stable client-sync id (UUID, as today) |
| `parent` | relationship → chat-messages | replies stay, thread scope = room |
| `author` | group (id/name/avatar/role) | unchanged shape |
| `content` | richText (Lexical) | unchanged pipeline |
| `commission` | relationship → commissions | OPTIONAL context chip only |

Access:
- read: `users` → `ownTenantOnlyQuery`; `customers` →
  `{ 'room.customer': { equals: user.id } }` (same dot-path pattern as today's
  `queueItem.commission.customer`)
- create: `users` → `true` (action enforces tenant via room lookup, like
  `postCommentAction` does today); `customers` → verify `room.customer ===
  user.id` by loading the room (mirrors today's `canCreateComment`)
- update: `false`; delete: `ownTenantOnly`

Hooks:
- `beforeValidate`: `associateTenant`
- `afterChange` (context-guarded, like `notifyCustomerOnCommissionFinished`):
  - `redis.incr('chat:version:<roomId>')` + 24h expire
  - set `room.lastMessageAt` and increment the OTHER party's unread counter
  - revalidatePath for room / dashboard chat list
- `afterDelete`: bump version stamp

## 4. Action layer (`src/features/chat/`, new feature)

- `get-or-create-room.action.ts` — `getOrCreateRoom({ tenantId, customerId })`:
  finds by `[tenant, customer]`, creates with `overrideAccess: true` on miss
  (catch unique-race like `getOrCreateQueueItemForCommission`).
- `post-message.action.ts` — evolved `postCommentAction(roomId, content,
  parentId?, commissionId?)`: rate limit → auth → load room → artist:
  `userHasTenantAccess(user, room.tenant)`; customer: `room.customer ===
  user.id` → parent-must-belong-to-same-room check (today's cross-thread
  guard) → DB create (transaction) → Redis lpush/ltrim/expire + version incr.
- `get-room-messages.action.ts` — `getRoomMessages(roomId)` (DB, replaces
  `getQueueComments`) and `getCachedRoomMessages(roomId)` (auth gate +
  `chat:cache:<roomId>` lrange, replaces `getCachedComments`).
- `mark-room-read.action.ts` — zeroes `unreadCustomer`/`unreadArtist` for the
  caller's side.
- `notify-customer.action.ts` — `notifyCustomerAction` becomes room-scoped
  (tenant check stays; email/Discord copy references the room, optionally the
  commission chip).
- `get-rooms-for-user.action.ts` — chat list for dashboard (artist: rooms by
  tenant sorted by `lastMessageAt`; customer: rooms by `customer` id).

Redis key migration (simple, old keys expire in 24h):
`comments:cache:<queueItemId>` → `chat:cache:<roomId>`,
`comments:version:*` → `chat:version:<roomId>`,
`comments:ticket:*` → `chat:ticket:<uuid>` (ticket now stores `{ roomId,
userId }`; poll route verifies room match).

## 5. Data migration + backfill

Two phases:

1. **DDL** — register both collections in `payload.config.ts`; let Payload
   push generate the create-table migration (`chat_rooms`, `chat_messages`,
   single-relationship columns `tenant_id`, `customer_id`, `room_id`,
   `commission_id`, `parent_id`, plus indexes and the unique
   `[tenant_id, customer_id]` index on rooms). Add id prefixes
   `chatRoom: 'chrm_'`, `chatMessage: 'chmsg_'` to `src/lib/utils/id.ts`.
2. **Backfill** — one-off Node script (pattern: existing `scripts/sync-db.ts`,
   `src/seedGlobals.ts`) using the Local API with `overrideAccess: true`:
   - For each distinct `(tenant, commission.customer)` pair present in
     `queue-comments` (via queue → commission join): create the room.
   - For each `queue-comment`: create `chat-message` copying `commentId`,
     `author`, `content`, `createdAt`, `updatedAt`; set `commission` from the
     old `queueItem.commission`; remap `parent` via `commentId` (old parent
     `qcm_…` → new message whose `commentId` matches).
   - Idempotent (skip already-migrated `commentId`s), logged counts, runnable
     as `npx tsx scripts/backfill-chat-rooms.ts`.

Rollout order (single deploy, two releases):
- **Release A**: ship collections + actions + backfill script; RUN backfill;
  switch UI to rooms; keep `queue-comments` readable (legacy) — remove the
  Queue `beforeDelete` cascade (rooms must outlive queue items); old
  `/queue/[id]/discussion` and `/api/comments/cache/[id]` redirect to the new
  room routes so stale links keep working.
- **Release B** (one release later): drop/archive `queue-comments` and
  `deleteQueueCommentsByQueueItem`.

## 6. UI changes

- `DiscussionDock` → accepts `roomId` + optional `commission` (context chip) +
  `tenantId`; phone link becomes `/chat/<roomId>?commission=<id>`. Server
  resolves `roomId` via `getOrCreateRoom` before rendering.
- `/queue/[id]/discussion` → 308-redirect: resolve room from queue item, then
  `/chat/<roomId>`.
- New `/chat/[roomId]` full-page (phone) reusing `CommentsSection`.
- `CommentsWrapper`/`CommentsSection`: swap `queueItemId` for `roomId`;
  poll endpoint `/api/chat/cache/[roomId]` (ticket + version short-circuit
  logic unchanged).
- `NotifyButton` → room-scoped notify action.
- `CommissionDetailContent` + `queue/[id]/page.tsx`: pass room props.
- Dashboard: new chat list view (`/dashboard/messages`) with unread badges —
  can be a follow-up UI phase; room unread counters are ready in schema.

## 7. File-by-file impact

New:
- `src/collections/ChatRooms.ts`, `src/collections/ChatMessages.ts`
- `src/features/chat/{index.ts, actions/*}` (5 actions above)
- `src/app/api/chat/cache/[roomId]/route.ts` (poll route v2)
- `src/app/(frontend)/(studio)/chat/[roomId]/page.tsx`
- `scripts/backfill-chat-rooms.ts`
- migration file (Payload-generated DDL)

Modified:
- `src/payload.config.ts` (register collections)
- `src/lib/utils/id.ts` (prefixes)
- `src/features/queue/actions/comment-actions.ts` → superseded by chat actions
  (keep legacy re-export during Release A)
- `src/features/queue/actions/queue-comment.action.ts` (legacy only)
- `src/features/queue/components/{CommentsWrapper,CommentsSection}.tsx`
  (roomId swap; keep poller logic)
- `src/features/queue/components/DiscussionDock.tsx`, `NotifyButton.tsx`
- `src/app/(frontend)/(studio)/queue/[id]/page.tsx`,
  `.../discussion/page.tsx`, `.../actions.ts`
- `src/features/commissions/components/review/CommissionDetailContent.tsx`
- `src/collections/Queue.ts` (remove comment cascade in Release A)
- `docs/caching-strategy-and-scaling.md` (update hot-path note)

Unchanged (reused): `src/lib/utils/commentContent.ts` (validation/sanitization),
`commentRateLimiter`, Lexical editor, review system, poller mechanics
(version + ticket), tenant access helpers.

## 8. Risks / decisions to confirm

- **Multi-artist studio**: room is tenant-wide; per-message `author` group
  keeps attribution correct. Confirm "2 parties" (not literally 2 accounts)
  matches intent.
- **Customer with multiple studios**: separate room per studio — correct by
  tenant isolation.
- **Old links**: keep redirects in Release A so emailed discussion links and
  bookmarks land in the room.
- **Unread semantics**: who "unreads" what — decided: message author's side
  never increments; other side increments; mark-read on opening the room.
- **Commission chips**: purely cosmetic reference; deleting a commission must
  NOT cascade to messages (set null on delete or keep orphan chip text).
- **Queue item deletion**: stops deleting chat history (the whole point).
  Legacy `queue-comments` cleanup only.

## 9. Suggested implementation order

1. Collections + id prefixes + config (Release A code).
2. Chat actions + Redis key swap + poll route v2.
3. Backfill script; run on staging, verify counts.
4. UI swap (wrapper/section/dock/pages) + legacy redirects.
5. Notifications + unread + mark-read.
6. Release B cleanup of `queue-comments`.
