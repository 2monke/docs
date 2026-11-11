---
status: shipped
collections: [pages, products]
features: []
---

# Admin navigation & the leave guard

## The reported bug

Create a product, type something, click **Products** in the nav, confirm
**Leave**, then open **Create New** again: the create screen renders for a
moment and is **pushed back to the products list** (~0.6 s later). Leaving via
the browser Back button instead of the prompt does not do it.

It has nothing to do with the setup guide (the guide being closed, or open, does
not matter — the setup tour's steps can only ever target theme / pages /
studio-profile / payouts / `/admin`, never a collection list), and nothing to do
with any of the repo's own redirects.

## What actually happens

Payload's `LeaveWithoutSaving` (`usePreventLeave`) does not navigate when you
click a link while a form has unsaved changes. It:

1. intercepts the click on a **`document` capture listener**, stores the anchor's
   href in a ref, and calls `preventDefault()`;
2. when the artist confirms in the modal, pushes that stored URL **inside
   `startRouteTransition`** — i.e. at *transition priority*, which React may
   apply long after the call.

On a dev server (slow RSC fetches) that deferred push can land **after the
artist's next navigation**: the create page commits, then ~0.6 s later the queued
push to the list wins and the artist is dragged back.

Reproduced and measured (Chromium, real session):

```
click "Leave"  → [BLOCK] click preventDefault href=/admin/collections/products   ← Payload's guard
               → [NAV] pushState /admin/collections/products                     ← accepted leave
on the list    → [ROUTER:push] /admin/collections/products/create                 ← the artist's click
               → [ROUTER:push] https://…/admin/collections/products               ← Payload's deferred push
               → [ROUTER:push] https://…/admin/collections/products               ← …twice
```

Both stray pushes come from the same line in Payload's `usePreventLeave`
(@payloadcms/ui 3.90.1); leaving with Back (no interception, no stored href)
never produces them. Payload 3.90.1 has no config flag to switch its guard off
(`disableLeaveWithoutSaving` is a document-view prop, not a collection option).

## The rule

**The admin owns the leave prompt for its document forms.** Payload's guard must
never see a leaving click, because anything it stores can be pushed later.

## The implementation

`src/components/admin/AdminLeaveGuard.tsx`, mounted into every collection's
document form by `withAdminLeaveGuard()` in `payload.config.ts` (so a new
collection cannot be born without it; existing `beforeDocumentControls` entries
are preserved, e.g. ProductTypes' create redirect).

- It listens for clicks on **`window` in the capture phase** — which runs before
  Payload's `document` listener — and calls `preventDefault()` +
  `stopPropagation()`. That cancels the navigation *and* keeps Payload's guard
  from ever storing an href, so there is nothing it can push later.
- It arms under Payload's own condition (`modified || !isValid`), so no prompt is
  gained or lost relative to the old behaviour, and it ignores same-URL,
  hash-only, new-tab and download clicks.
- On **Leave** it navigates **eagerly, once** (`router.push` outside any
  transition) and releases the document lock, matching what Payload's guard did.
  On **Stay** the pending href is dropped.
- The modal is Payload's own `ConfirmationModal` with Payload's translations
  ("Leave without saving?" / "Stay on this page" / "Leave anyway"), so the copy
  and styling are unchanged.

## Verification

- `tests/int/admin-leave-guard.int.spec.ts` — the guard takes the click over and
  shields Payload's document listener; a confirmed leave pushes **exactly once**
  and never again (the regression); clean forms, same-URL/hash/new-tab/download
  clicks are untouched; the collection wiring appends without duplicating or
  mutating. Mutation-checked (dropping `stopPropagation` fails the shielding
  case).
- `tests/e2e/admin-leave-guard.e2e.spec.ts` — the reported flow in a real
  browser: leave via the prompt, reopen Create New, still on Create New after
  2.5 s. Passes against the dev server.
- Live confirmation that the interception now comes from
  `AdminLeaveGuard.useEffect.handleClick` (window capture) instead of Payload's
  document listener, and that the prompt still appears.
