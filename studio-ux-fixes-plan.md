---
status: shipped
collections: [artists, commission-settings, commissions, media, pages, product-types, products, raw-media, studios, themes]
features: [admin-dashboard, admin-navigation, commissions, onboarding]
---

# Studio UX Fixes — Plan (11 issues from artist testing)

Date: 2026-10-05 · Branch: `px-or-rem` (clean) · Source: user's test notes

Ordering: **Wave 1** = small, self-contained fixes shipped first · **Wave 2** = focused UX builds · **Wave 3** = bigger changes with decisions/migrations. Each item names its real owner files (verified by reading the code, not guessed).

---

## Wave 1 — quick wins (small diffs, no decisions needed)

### 1. Enter submits studio creation
- [ ] Where: `src/features/onboarding/components/OnboardingView.tsx` (form has `onSubmit={(e) => e.preventDefault()}` — this is why Enter does nothing), `OnboardingFooter.tsx` (CTA holds `onSubmit={handleSubmit}`), `src/features/onboarding/hooks/useOnboardingForm.ts` (`handleSubmit`).
- Approach: wire the form's `onSubmit` to the step's primary action (Enter on the final step → `handleSubmit`; Enter on earlier steps → advance if the step is valid). Guard `<textarea>` (Enter = newline, not submit).
- Verify: `tests/e2e/artist-onboarding.e2e.spec.ts` — add a case: fill last step, press Enter, studio is created.

### 2. "Open Studio Admin" must not deep-link to the theme page for already-registered studios
- [ ] Where: `src/app/(frontend)/(platform)/studios/page.tsx` `getStudioAdminUrl()` — `redirectPath = isDraft && themeId ? /admin/collections/themes/${themeId} : '/admin'` is the bug: any draft studio deep-links to the theme doc on every visit.
- Approach: always land on `/admin` from the platform link. The one-time theme landing stays where it already happens correctly:
  - right after creation: `useOnboardingForm` does `window.location.replace('/admin/collections/themes/<themeId>')` (keep);
  - first-ever revisit: the Setup Guide auto-opens for a draft studio and its **step 1 is `theme-and-colors`** (`src/lib/tour/curriculum.ts`), and the runner's auto-navigation already steers to the step's screen (theme doc) — that is the "only once" moment, governed by the tour's persisted `hasAutoOpened` (`src/lib/tour/state.ts`), not by a URL condition.
  - opening the guide manually while it sits on step 1 → same runner auto-navigation (already shipped; add a regression check).
- Verify: e2e — (a) click Open Studio Admin as a registered draft studio → lands on `/admin`; (b) fresh studio after creation → theme doc; (c) open guide at step 1 from `/admin` → runner steers to theme doc.

### 3. Restore the burger menu on mobile admin
- [ ] Where: `src/app/(payload)/styles/_admin-theme.scss` — two hide blocks: ~line 1728 (`< 769px`: hides `.nav-toggler*`, `.nav`, `.nav__header` "in favor of Card Dashboard + '← Admin Hub'") and ~line 5039 (canvas views hide nav/toggler).
- Approach: re-enable `.nav-toggler*` in the mobile block (keep the '← Admin Hub' button as an extra), keep the canvas-view hiding desktop-only. Check `src/features/admin-navigation/components/ResponsiveNavPortal.tsx` opens the portalled nav as an overlay at mobile widths.
- Verify: manual probe at 390px (burger visible, opens nav overlay, closes) + stylesheet guard in the existing admin theme specs.

### 4. List-view search row: icon centering + control heights
- [ ] Where: `src/app/(payload)/styles/_admin-theme.scss` search block (~line 261: `.search-bar`, `.search-filter`, `.list-controls__search`, `.search-filter__input-wrapper`) vs the columns/filters buttons in the same row.
- Approach: one height/line-height token for the row (input, icon, buttons), center the search icon via flex instead of absolute offsets.
- Verify: Playwright probe measuring computed heights at 1440px on Products list (input height == buttons height; icon centered).

---

## Wave 2 — focused UX builds

### 5. Root admin page: task cards instead of a collections list
- [ ] Where: `src/components/admin/DashboardViewServer.tsx` (renders `DashboardHero` + `CollectionGrid`), `src/features/admin-dashboard/components/CollectionGrid.tsx`, `DirectStudioLinks.tsx` (existing direct-link pattern), `src/lib/utils/admin-ia.ts` (labels/taxonomy).
- Approach: for artists, replace the nav-groups grid with 4–6 verb cards reusing the existing card styles: **Create a product** (`/admin/collections/products/create`), **Update commission prices** (`/admin/commission-settings`), **Create a page** (`/admin/collections/pages/create`), **Customize theme** (theme doc), + Shop/Pages quick links. Keep the technical grid for superadmins.
- Verify: render spec for the artist dashboard (cards + hrefs) + e2e click-through.

### 6. Product types admin: view components, icon field, slug
- [ ] Where: `src/collections/ProductTypes.ts` (icon = plain text field with a name list in the description; slug hook only fills `slug` on create when empty), `src/components/admin/ProductTypeSelectField.tsx` + `ProductTypeCell.tsx` (what the product edit view renders for a product type).
- Approach:
  - **Product view**: restyle the product-type selector/cell to the kit grammar (cards/segmented, consistent with ChoiceControl) — needs a quick screenshot from the user to pin the "terrible" part.
  - **Icon field**: build a visual icon picker from the existing `ICON_MAP` (10 lucide icons already used by the storefront) plus a free-text "write the name" fallback — the *":D"* reading; confirm with the user whether they want the picker or genuinely the plain name input (see decisions).
  - **Slug**: derive live from the name in the UI (readOnly-with-override), and extend the existing `beforeValidate` hook to keep it in sync when the name changes and the artist has not manually edited the slug.
- Verify: int specs — icon picker renders + writes the name; slug follows name; product-type field render spec.

### 7. Info hints in the commission form (and commission types)
- [ ] Where: `src/features/commissions/components/CommissionForm.tsx` (no tooltips today) writes to the customer-facing application form; commission types are edited in the admin (`CommissionSettingsAdminView` — admin descriptions already render as "?" tooltips through `FieldHelpTooltipEnhancer`).
- Approach: add a lightweight storefront `InfoHint` (accessible "?" with tooltip) to the customer-facing fields that need explaining (drawing tier, background, extras, reference notes), copy in plain language; confirm whether "for commission types too" means the public type selector or the admin type editor.
- Verify: render spec (hint present + aria-describedby) + no layout shift on mobile.

---

## Wave 3 — bigger changes (need decisions first)

### 8. No default commission tiers at studio creation + guidance
- [ ] Where: `src/features/onboarding/actions/artist-onboarding.action.ts` (`STARTER_PRICING_TIERS`, ~line 44 and 266 — the comment explains they exist because a studio with zero tiers **cannot take a commission**: `CommissionForm` returns no blocks, `ProjectTierSelector` renders nothing and submit hard-blocks).
- This is why removing defaults is not a one-liner: the empty state must become a *guided* empty state.
- Approach: (a) make the commission flow degrade gracefully with zero tiers (prices page + form show "Prices not set up yet — owner guidance" instead of a dead end); (b) stop seeding tiers at creation; (c) add a "Set up your prices" action card + point the Setup Guide's pricing step at real guidance (the tour already has the tiers step — `docs/setup-tour-content-plan.md` has ready copy).
- Verify: new-studio walkthrough without tiers (form renders guidance, no hard-block confusion), prices page empty state, tour step reachable; onboarding e2e updated.

### 9. Brand logo & iconography → theme page
- [x] **Implemented 2026-10-05/06 — option A (crop tool preserved).**
  - `Themes` Header group gains `logo` + `logoMobile` upload fields, edited by the new `ThemeBrandImageField`: upload → Payload's adjust drawer (crop + focal point, server-side rewrite) → stored as the media id. Vectors upload as-is, as before.
  - The Studio & Artist profile screen drops its two logo tiles and now points to the Theme editor (`data-guide-target="studio-logo-moved"`); its save no longer writes the deprecated studio fields, and the dirty-check ignores them.
  - Consumers read theme-first with a studio fallback: `studios.cache.ts` (`toStorefrontStudioSettings` + `fetchSocialSettings` — this is what `NavHeader`/`SiteFooter`/`Logo` consume), `deliver-push.action.ts` (push icon), `useTourFacts.ts` (branding satisfaction). The theme save hook now revalidates both `meta` and `layout` tags so a saved logo cannot sit behind a stale layout cache.
  - Tour: the theme step gains a `theme_logo` sub-step; the branding step becomes socials-only ("Your Social Links"), with a tip pointing at where the logo went. `tour-anchors` spec updated to the new tile set.
  - Migration `20261005_170145_move_brand_logo_to_theme`: adds the two columns (+FKs, indexes) and copies studio logos into empty theme slots (idempotent, never overwrites). Registered in `src/migrations/index.ts`; the data move is a no-op on logos-less databases.
  - **Pending**: live verification once the dev server runs; and the dev DB needs the two columns — starting the dev server push-applies them (code now selects `logo_id`, so specs against the un-pushed DB fail with `column "logo_id" does not exist` until then).

### 10. "How My Studio Pages Work" panel (DECISION)
- [ ] Where: `src/components/admin/PagesListView.tsx` (~line 213) + `src/lib/system-pages-guide.ts`.
- The user's tone says it is noise. Options: remove the panel entirely / collapse it to a one-line link / fold the content into the tour's Pages & Legal topic tour (which already covers it).
- Recommendation: remove the panel from the list header and keep the guidance in the tour only.

---

## Suggested order & rough sizes
1. (1) Enter submit — XS · 2. (2) redirect — S · 3. (3) burger — S · 4. (4) search row — S
5. (6) product types (slug first, then icon, then view) — M · 6. (5) dashboard cards — M · 7. (7) info hints — S–M
8. (8) tiers + guidance — M–L · 9. (9) logo move — L (decision) · 10. (10) panel removal — XS (decision)

### Crop / logo apply 500 — diagnosed and fixed 2026-10-07
The theme logo upload itself worked; **applying crop/focal edits** (and the same flow on the profile screens) failed with `500 FileRetrievalError: Blocked unsafe attempt to kotik.localhost`. Chain:
1. S3 configured (`NEXT_PUBLIC_S3_BUCKET` + `S3_ACCESS_KEY_ID`) → `disableLocalStorage` → Payload re-reads the file by URL instead of disk.
2. The stored URL is relative, so it resolves against the request origin (`kotik.localhost:3000`) and Payload's SSRF guard rejects private hosts — with an exact-match allowlist that cannot express tenant subdomains.
   → **Fix: `skipSafeFetch: true`** on the `media` + `raw-media` upload configs (the fetched URL always comes from the storage adapter, never user input; Payload's documented escape hatch for trusted destinations). Verified: the 500 became a fetch attempt (`SELF_SIGNED_CERT_IN_CHAIN`).
3. Dev servers run `--experimental-https` with a self-signed cert; Node's server-side self-fetch rejects it.
   → **Fix: `NODE_TLS_REJECT_UNAUTHORIZED=0` in the `dev` script** (dev-only; production uses valid certs). Verified standalone: with the env var the self-fetch returns 200.

Post-restart verification pending: restart `yarn dev` (new env), then re-run the crop PATCH (probe kept at `tests/.probe-crop-fix.mjs`): expect **200**.

Also noted while diagnosing: admin-on-subdomain logs a CORS/CSRF warning ("origin https://kotik.localhost:3000 is not in the allowlist … set the serverURL") — harmless here, but a candidate follow-up config improvement.

### Progress 2026-10-05/07 (session)
- **Shipped in code (verified):** all ten items — 1 (Enter submit + e2e case), 2 (redirect), 3 (mobile burger + overlay), 4 (search row), 5 (dashboard task cards), 6 (product types: icon picker, auto-slug, product-view rewrite to the admin kit), 7 (commission-form info hints), 8 (no seeded tiers + guided empty states + tour copy), 9 (brand logo moved to the theme editor, crop flow preserved, migration included), 10 (panel removed).
- **Verified live (dev server, browser probes + e2e):** dashboard task cards + collections grid hidden for artists; mobile burger opens/closes the overlay; search input & toggle pills both 44.2px with the icon centered (±1px — the filter's line-box descender offset was the 10px bug; fixed with a flex wrapper); product view renders the type chooser as 4 icon cards / 0 raw selects; product-type view shows the 10-icon picker + auto-slug field; theme editor shows Brand logo + Mobile mark with upload controls; onboarding e2e (Enter advances, textarea newline guard, submit-path validation) and tour-anchors e2e **20/20** (incl. the new `theme_logo` sub-step; `ThemeBrandImageField` now renders `id="field-logo"` on its root so guide anchors and the scroll bridge resolve).
- **Rem token calibration saga (settled 2026-10-07):** the px→rem sweep first used px/16, but Payload's admin root is 13px ($baseline-body-size) — every converted value rendered ~19% small (44px→35.75px, verified live). A `calc(N * var(--ap-px))` intermediate was rejected as over-engineered; the settled solution is **clean rem values snapped to a 0.05rem grid at the 13px root** (3.4rem ≈ 44.2px, 5.4rem ≈ 70.2px, 0.6rem, 1.25rem…), with the named geometry living in the `--ap-*` tokens. Source reads as ordinary rem; rendering is within ~0.3px of the original pixel design.
- **Known pre-existing failures (not from this work):** status-token offender in `ProfileNotificationsCard.tsx`; 2× S3 media-prefix specs; Stripe Accounts v1 deprecation (external). `chat_rooms.customer_id` is NOT NULL under an `ON DELETE SET NULL` FK — deleting a customer with rooms fails at the DB (test hygiene fixed); the FK itself deserves a decision.

### Product type create ends on the table — 2026-10-08
After item 6 shipped, creating a type still felt unfinished: `/admin/collections/product-types/create` is Payload's document form, and on save Payload pushes the browser onto the **new document's edit view** (`/admin/collections/product-types/<id>`), which for a two-field type is the form the artist just filled in, one URL later. Fix: `src/components/admin/ProductTypeCreateRedirect.tsx`, mounted through `admin.components.edit.beforeDocumentControls` — the only slot *inside* the document form, where Payload's form state is reachable (the form is a child of the edit view, so a `views.edit.default` wrapper sits outside it and cannot see a save).

- **Detection**: on the create screen, a finished save is `processing` true→false with `submitted` **false** (a failed save leaves `submitted` set — Payload's own Form state; `isSuccessfulSaveTransition` in the component, unit-pinned).
- **Carrier**: a one-shot `sessionStorage` marker, not component state — verified live that Payload **remounts the document view** on the create → new-document navigation (an instance-id probe reset), so a ref-based "armed" flag was gone exactly when needed (first attempt: view wrapper + ref → no redirect fired).
- **Destination**: on any document screen the marker is consumed and the view `router.replace`d to `/admin/collections/product-types`; `replace` because the transient edit view was never a destination the artist chose. Drawers (`editDepth >= 2`) and updates (the marker is only written while `isEditing` is false) never fire.
- **Verified live** (dev server, Chromium probe as seeded artist `cyberneon@artpack.dev`): create → new doc → list with the new row visible; an existing type's edit view does not bounce; probe types cleaned up. Pinned by `tests/int/product-type-create-redirect.int.spec.ts` (9/9: pure rule, marker consume-once, wiring incl. the remount case, drawer/failure/edit-screen negatives, collection registration) and `tests/e2e/product-type-create-redirect.e2e.spec.ts` (needs the suite seed; not run in this session).
