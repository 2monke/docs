---
status: spec
collections: [commission-settings, queue]
features: [auth, studio-settings]
---


# Anonymous Live Preview & Admin Nested Border Refactor

Architectural specification for Automatic Anonymous Live Preview mode and the nested border design cleanup across Admin commission settings and Stripe payouts.

---

## 1. Architecture Overview

### Automatic Anonymous Live Preview

Previously, opening Payload CMS Live Preview or testing a theme draft reused the artist/admin's active authenticated session (`payload-token` cookie). This caused preview frames to render internal administrative UI (such as the `ArtistAdminTopBar`, edit shortcuts, or studio owner navigation) instead of reflecting the true end-user / public visitor experience.

Automatic Anonymous Live Preview decouples the preview frame from the active session by default, while providing an opt-in bypass (`?auth=true`) when authenticated preview is specifically required.

```
Admin Live Preview / Draft Request
(e.g., ?preview=true, ?draft=..., or sec-fetch-dest: iframe with referer /admin)
                         │
                         ▼
             Next.js Proxy (`src/proxy.ts`)
                         │
        ┌────────────────┴────────────────┐
        │                                 │
  has ?auth=true?                  no ?auth=true (Default)
        │                                 │
        ▼                                 ▼
Preserve credentials             Enable Anonymous Mode:
- Forward payload-token           - Strip `authorization` header
- Forward authorization           - Strip `payload-token` from `cookie`
- Set `x-is-live-preview: true`   - Set `x-is-live-preview: true`
                                  - Set `x-anonymous-preview: true`
                                          │
                         ┌────────────────┴────────────────┐
                         ▼                                 ▼
             Server Auth Actions / Cache           Studio Layout & Page Data
         - `getAuthenticatedUser()` returns null   - `StudioHeader`: renders public
         - `getCachedUser()` returns null            visitor nav, omits AdminTopBar
         - Server components render as guest       - `HomeContent`: `token = undefined`,
                                                     blocks render public visitor data
```

---

## 2. Component & Layer Breakdown

### A0. Shared Contract Module (`src/lib/theme/live-preview-headers.ts` + `src/lib/theme/live-preview.ts`)

All preview detection rules live in one place now. Before the extraction the
header names, the query-param list and the detection rules were copy-pasted
across five call sites (the proxy, `StudioHeader`, `auth.cache` ×2,
`getAuthenticatedUser`, and the client-side `ArtistAdminTopBar`), and two of
them had drifted: the proxy required an iframe **and** an admin referer, while
`StudioHeader` accepted either.

- **`src/lib/theme/live-preview-headers.ts`** — pure (no `next/*` imports) so the edge
  proxy and unit tests can both import it:
  - `LIVE_PREVIEW_HEADER` / `ANONYMOUS_PREVIEW_HEADER` — the only place the
    header names are spelled out.
  - `PREVIEW_QUERY_PARAMS` + `hasPreviewQueryParam()` — one param list, shared
    with the client top bar.
  - `readPreviewState()` / `isAnonymousPreview()` — read the **trusted** headers
    only (auth-safe).
  - `isAdminIframeEmbed()` / `detectPreviewRequest()` — the proxy's rule.
  - `resolvePreviewPresentation()` — the component rule: trusted verdict first,
    then the broader embed fallback (iframe *or* admin referer), which is
    cosmetic and must never influence authentication.
- **`src/lib/theme/live-preview.ts`** — request-scoped `getPreviewState()`
  (`cache()`-memoized, out-of-request-store safe), mirroring the existing
  `subdomain.ts` / `subdomain-host.ts` pairing.

Call sites now consume the module: `proxy.ts` mints the headers from
`detectPreviewRequest`, `StudioHeader` reads `getPreviewState()`, the three auth
paths ask `isAnonymousPreview()`, and `ArtistAdminTopBar` reuses the shared
param list.

`StudioHeader` no longer pre-empts anonymous mode with its own header check:
`getCachedUser()` already returns `null` for anonymous preview, so the extra
branch was a second copy of the same rule (removed).

### A. Edge Proxy (`src/proxy.ts`)

- **Untrusted Header Scrubbing**: Strips client-supplied `x-is-live-preview` and `x-anonymous-preview` headers so external callers cannot forge preview state.
- **Preview Detection**: delegates to `detectPreviewRequest()` — query parameters (`preview`, `draft`, `livePreview`, `live-preview`) or `sec-fetch-dest: iframe` originating from a `/admin` referer.
- **Anonymous Scrubbing**:
  - Checks if `searchParams.get('auth') === 'true'`.
  - When false, injects `x-anonymous-preview: true` into forwarded request headers.
  - Strips the `authorization` header and filters `payload-token=<jwt>` out of the forwarded `cookie` string.

### B. Auth Actions & Request-Scoped Cache

- **`getPayloadToken` (`src/features/auth/cache/auth.cache.ts`)**:
  - Centralized, request-memoized (`React.cache()`) token retriever.
  - Automatically checks the trusted anonymous-preview header via
    `isAnonymousPreview()` (shared contract module) and returns `undefined` in
    anonymous live preview.
  - Returns `cookieStore.get('payload-token')?.value` for standard authenticated requests.
- **`getCachedUser` (`src/features/auth/cache/auth.cache.ts`)**:
  - Inspects incoming request headers via `headers()`.
  - Automatically returns `null` when the shared `isAnonymousPreview()` check is true.
  - Made the `cookieToken` argument optional (`getCachedUser(cookieToken?: string)`); defaults to calling `await getPayloadToken()`.
  - Components and actions can simply call `await getCachedUser()` without needing to manually inspect or pass `cookies().get('payload-token')`.
- **`getAuthenticatedUser` (`src/features/auth/actions/get-authenticated-user.action.ts`)**:
  - Detects anonymous preview via `isAnonymousPreview(resolvedHeaders)` — the
    trusted proxy header only; `sec-fetch-dest`/`referer` are client-supplied
    and never reach an auth decision.
  - Early-returns `{ user: null, payload }` without querying database or verifying tokens.

### C. Studio Chrome & Content Rendering

- **`StudioHeader` (`src/components/StudioHeader.tsx`)**:
  - Reads preview state through `getPreviewState()` (shared contract module)
    instead of re-deriving it from headers.
  - Calls `getCachedUser(token)`, which resolves to `null` during anonymous
    preview — rendering guest actions (Sign In / Register) and omitting
    `ArtistAdminTopBar`.
- **`HomeContent` & `QueueContent`**:
  - Reads `const token = await getPayloadToken()` and passes it to `loadPageData`, ensuring live preview renders guest content cleanly.
- **Shop & Chat Components (`UserActionsWrapper`, `ProductReviewsWrapper`, `chat/[roomId]`, `queue/[id]`)**:
  - Call `await getCachedUser()` directly with zero cookie-reading boilerplate.

---

## 3. UI & Styling Refactor: Elimination of Nested Borders

### A. Pricing Rules (`src/components/admin/commission-settings/CommissionTypesTab.tsx`)

- **Problem**: The AI prompt instructions for commission categories had a container with `border: '1px solid var(--theme-elevation-150)'` wrapping a native `<textarea>` that also carried an input border, creating an awkward double-card visual effect.
- **Solution**: Removed the redundant outer card border and surrounding padding. The `<textarea>` renders directly as an elevated form control with standard input border and proportional `minHeight: '68px'`.

### B. Stripe Payouts Section (`src/features/studio-settings/components/IntegrationsTab.tsx`)

- **Problem**: Payouts rendered nested `.payouts-card` containers inside `.payouts-section.cs-card`, compounded by nested alert boxes and fee cards.
- **Solution**:
  - Unified outer wrapper into `.payouts-section.cs-card` with an anchored section header and clean border divider.
  - Unconnected state: Eliminated inner card border; action buttons and guidance align cleanly inside the parent card.
  - Connected state: Converted inner panels (Payment Processing, Bank Payouts, Account ID, Express Dashboard) into subtle stat tiles (`var(--theme-elevation-50)` background with `var(--theme-elevation-100)` borders).
  - Platform fee info: Refactored into a modern left-accented callout banner (`borderLeft: 3px solid hsl(241, 67%, 67%)`) rather than a full nested card.

---

## 4. Summary of Files Changed

| File | Nature of Change |
| --- | --- |
| `src/lib/theme/live-preview-headers.ts` | **New** — pure contract module: header names, preview query params, `detectPreviewRequest` (proxy rule), `resolvePreviewPresentation` (component rule), auth-safe `isAnonymousPreview`. |
| `src/lib/theme/live-preview.ts` | **New** — request-scoped `getPreviewState()` accessor (memoized, out-of-request-store safe). |
| `src/proxy.ts` | Added live preview detection, anonymous mode header injection, and cookie/auth stripping with `?auth=true` bypass; detection now delegates to the shared contract module. |
| `src/features/auth/actions/get-authenticated-user.action.ts` | Early return `{ user: null, payload }` on anonymous preview, via `isAnonymousPreview()`. |
| `src/features/auth/cache/auth.cache.ts` | `getCachedUser` / `getPayloadToken` return null/undefined on anonymous preview, via `isAnonymousPreview()`. |
| `src/components/StudioHeader.tsx` | Preview state now read from `getPreviewState()`; the duplicated anonymous-header branch removed (`getCachedUser` already enforces it). |
| `src/components/ArtistAdminTopBar.tsx` | Reuses the shared preview query-param list instead of its own copy. |
| `src/app/(frontend)/(studio)/HomeContent.tsx` | Strips token in `loadPageData` when in anonymous preview. |
| `src/components/admin/commission-settings/CommissionTypesTab.tsx` | Removed redundant outer border around category pricing rules. |
| `src/features/studio-settings/components/IntegrationsTab.tsx` | Refactored Payouts card hierarchy, stat tiles, and platform fee callout. |
| `tests/int/anonymous-live-preview.int.spec.ts` | Integration suite for the edge proxy and server auth behavior, plus the shared contract module's detection matrix and auth-safety rules. |
| `scripts/smoke-preview-http.sh` | **New** — HTTP-level smoke check of the end-to-end preview path (authenticated vs `?preview=true` vs `?preview=true&auth=true`). |

---

## 5. Verification & Testing

### Vitest Integration Tests

Run the test suite:
```bash
npx vitest run tests/int/anonymous-live-preview.int.spec.ts
```

Invariants tested:
1. Standard requests do not inject `x-is-live-preview` or `x-anonymous-preview`.
2. `?preview=true` activates live preview and strips `payload-token` from forwarded cookies.
3. `?draft=...` activates live preview and anonymous mode.
4. Admin iframe embeds (`sec-fetch-dest: iframe` + `/admin` referer) activate live preview and anonymous mode.
5. Explicit `?auth=true` preserves authenticated session and prevents anonymous mode.
6. Client-spoofed preview headers are stripped when the request is not an actual preview.
7. `getAuthenticatedUser` returns `{ user: null }` when `x-anonymous-preview` is set.

### Type Check

```bash
npx tsc --noEmit
```
Must pass with 0 errors.

---

## 6. Debug Reference

To inspect anonymous live preview behavior in dev:
1. Open any studio page with `?preview=true` or edit a theme in Payload Admin.
2. In the Network tab or server console, verify:
   - Header `x-anonymous-preview: true` is present in server component logs.
   - Header `x-is-live-preview: true` is present.
   - No `ArtistAdminTopBar` appears in the preview frame.
   - Navigation displays guest "Sign In" / "Register" CTAs.
3. To test the authenticated bypass, append `&auth=true` to the URL. The page will render with the active artist session intact.
