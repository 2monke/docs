---
status: shipped
collections: [themes]
features: []
---

# Theme Live Preview

How the ArtPack theme editor shows unsaved (draft) theme changes in real-time across any page.

## Architecture

```
Field change in Admin (localhost:3000)
        │
        ├──► AIThemeGenerator useEffect → document.cookie
        │    payload-theme-preview=<JSON>; domain=localhost
        │
        ├──► Themes livePreview.url callback → URL with
        │    ?preview=theme&draft=<base64>&primary=...&bg=...
        │
        ▼
  New tab / iframe (vronyakat.localhost:3000)
        │
        ├──► proxy.ts reads cookie (or draft param)
        │    → injects x-draft-theme-data header
        │
        ├──► DynamicTheme (server component) reads header
        │    → merges draft with saved DB theme
        │    → renders <style> tag in <head>
        │
        └──► LivePreviewThemeOverride (client)
             ├─ Iframe: postMessage from Payload admin
             └─ New tab: polls document.cookie every 500ms
                → injects/updates <style id="lp-theme-override">
```

## Key components

### 1. AIThemeGeneratorAdminField (`src/components/admin/AIThemeGeneratorAdminField.tsx`)

- Uses `useField` hooks for all theme fields — colors, `fontFamily`/`fontSource`, `borderRadius`, `borderWidth`, `cardStyle`, `headerStyle`, `customFontName`, `customFontUrl`, `useLiquidGlass`, plus the customization levers (`fontSizeScale`, heading font split/scale/tracking/transform, `buttonShape`, `cardAspectRatio`, `containerMaxWidth`, `sectionSpacing` — see `docs/theme-customization-levers.md`)
- `useEffect` watches field values and writes to `document.cookie` with `domain=localhost` on every change
- Cookie format: `payload-theme-preview=<JSON>` (URL-encoded, NOT base64)
- Covers both manual form edits and AI-generated themes

### 2. Proxy (`src/proxy.ts`)

- Runs on all non-API/page requests
- Reads `payload-theme-preview` cookie (primary source — always current)
- Falls back to `draft` URL query param (secondary — may be stale)
- Injects `x-draft-theme-data` request header for server components
- Bridge between URL/cookie and server-readable headers

### 3. DynamicTheme (`src/app/(frontend)/layout.tsx:79`)

- Server component inside root layout
- Reads `x-draft-theme-data` header via `headers()`
- Parses: tries raw JSON first (cookie format), then base64 decode (URL format)
- Merges draft theme over saved DB theme: `{ ...savedTheme, ...draftTheme }`
- Renders `<style>` tag with CSS custom properties:
  ```css
  :root { --primary: 231 73% 55%; --background: 222 47% 11%; ... }
  body { background-color: hsl(222 47% 11%); ... }
  ```

### 4. LivePreviewThemeOverride (`src/components/LivePreviewThemeOverride.tsx`)

Client component for live updates after page load:

- **Iframe (Path A):** Listens for `payload-live-preview` postMessage events from Payload admin. Extracts theme fields from the document payload and injects `<style id="lp-theme-override">` with `!important` overrides. The `--…` declaration list comes from the shared `generateThemeVarDeclarations()` builder (also used by the SSR stylesheet), so SSR and preview can never drift.

- **New tab (Path B):** Polls `document.cookie` every 500ms for `payload-theme-preview`. When changes detected, injects the same `<style id="lp-theme-override">` override tag. Uses `useRef` to track last-serialized theme and avoid unnecessary DOM updates.

- Also keeps `data-card-ratio` on `<html>` in sync — the `natural` card framing mode needs a structural rule that CSS variables cannot express (see `docs/theme-customization-levers.md`).

### 5. BlocksRenderer gate (`src/components/BlocksRenderer.tsx:99`)

- Checks for `?preview=theme` in URL + `isInsideIframe`
- Skips `useLivePreview` when both are true — prevents cross-origin REST requests to the admin API that would 403

## CORS configuration

### Payload config (`src/payload.config.ts`)

```ts
cors: (req) => {
  const origin = req?.headers?.get?.('origin') || ''
  if (/^https?:\/\/([a-z0-9-]+\.)?localhost:\d+$/.test(origin)) return origin
  if (/^https:\/\/([a-z0-9-]+\.)?vronyakat\.com$/.test(origin)) return origin
  return origin || '*'
}
```
Returns exact origin (not `*`) so credentialed cross-subdomain requests work.

### API route handler (`src/app/(payload)/api/[...slug]/route.ts`)

- `withCors()` wrapper injects `Access-Control-Allow-Origin: <specific-origin>` + `Access-Control-Allow-Credentials: true` into ALL REST responses (GET/POST/PUT/PATCH/DELETE), not just OPTIONS.

## Why localStorage/BroadcastChannel failed

- Admin origin: `localhost:3000` — sets localStorage here
- Preview origin: `vronyakat.localhost:3000` — reads localStorage here
- localStorage is **per-origin** — these are completely separate storage buckets
- BroadcastChannel is also **per-origin** — can't cross between admin and preview
- Cookies with `domain=localhost` bridge this gap, accessible from both origins

## Why searchParams failed

- `searchParams` is a **page prop**, not available in Next.js layouts
- Layout components (where DynamicTheme runs) can't receive searchParams directly
- Workaround: middleware reads query/cookie → injects request header → server component reads via `headers()`

## Debug log reference

To verify the pipeline is working, check terminal/server logs for:

```
[DYNAMIC-THEME] x-draft-theme-data header: true
[DYNAMIC-THEME] decoded draft theme: { primaryColor: '#000000', ... }
[DYNAMIC-THEME] final background CSS var: 0 0% 0%
```

The `final background CSS var` should match the edited background color (e.g., `#000000` → `0 0% 0%`).
