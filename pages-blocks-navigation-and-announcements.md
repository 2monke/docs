---
status: shipped
collections: [announcements, artists, media, pages, queue, studios]
features: [admin-pages, studios]
---


# Pages, Storefront Blocks, Header Navigation & Announcements

> **Status**: Shipped

How studio pages (`pages`), system singleton routes (`home`, `prices`, `shop`, `queue`), header navigation ordering (`admin-pages`), page builder blocks, and banner announcements (`announcements`) work.

## System Pages vs. Custom Pages (`src/collections/Pages.ts`)

Every studio has system-provisioned pages (`isSystem: true`) alongside artist-created custom pages (`pageType: 'standard'`), enforced by `enforcePageTypeRules` (`src/collections/hooks/enforcePageTypeRules.ts`) and `preventSystemPageDeletion` (`src/collections/hooks/preventSystemPageDeletion.ts`):

- **Home Page (`slug: 'home'`)**: Cannot be deleted, cannot be renamed, and **cannot be hidden** (`isHidden` is forced to `false` so the studio root route always renders).
- **Singleton System Pages (`pageType: 'prices' | 'shop' | 'queue'`)**:
  - Each studio can have at most one `prices`, `shop`, and `queue` page (`"This studio already has a Prices/Shop/Queue page. Only one is allowed."`).
  - Their slugs are locked to `/prices`, `/shop`, and `/queue`, and `isSystem: true` prevents deletion.
  - Unlike `home`, non-home pages (including `prices`, `shop`, `queue`, and custom pages) can set `isHidden: true` (**Hidden from visitors**) to return a `404` to anonymous visitors.
- **Custom Pages (`pageType: 'standard'`)**:
  - Artists can create custom pages with any non-reserved slug. Built-in studio route names (`RESERVED_STUDIO_ROUTE_SLUGS` in `src/lib/blocks/systemPages.ts`) are rejected with `"is a built-in studio route, not a page — please choose a different slug."`
  - Duplicate manually entered slugs within the same studio are rejected, while title-derived slugs auto-append a 6-character suffix.

## Header Navigation & Reordering (`src/features/admin-pages`)

Each page has a **Header Navigation** sidebar group:
- `showInHeader` (**Show in Header**): Displays the page in the studio's top navigation bar.
- `navTitle` (**Navigation Title**): Optional custom menu label (defaults to `title` if blank).
- `headerOrder` (**Header Order**): Numeric sort order (lowest numbers appear first). Bulk reordering in the Pages admin view is capped at `MAX_HEADER_PAGE_UPDATES = 100` (`src/features/admin-pages/constants.ts`) via `src/features/admin-pages/actions/reorder-header-pages.action.ts`.

## Available Page Blocks

Pages compose layouts from 10 block types in `src/collections/Pages.ts`:
- `hero` (`HeroBlock`), `bentoGrid` (`BentoGridBlock`), `commissionForm` (`CommissionFormBlock`), `richText` (`RichTextBlock`), `mediaGallery` (`MediaGalleryBlock`), `faq` (`FAQBlock`), `socialLinks` (`SocialLinksBlock`), `sectionedContent` (`SectionedContentBlock`), `latestProducts` (`LatestProductsBlock`), and `featuredProducts` (`FeaturedProductsBlock`).

## Hero Buttons (`src/blocks/Hero.ts`)

The hero's two buttons are artist-controlled — the block hard-coded `/apply` and `/prices` before, which left no way to advertise one campaign or point somewhere else. Both buttons ask the same question (**Where should it go?**) in the artist's vocabulary:

- **Commission form** — `/apply#commission`, every type that is open.
- **One commission round** — `/apply?round=<id>`; optionally a `commissionTypeKey`, so the form opens on that round and type (`/apply?round=…&commissionType=…#commission`). The type list is read from Commission Settings and narrowed to the round's binding, so the artist picks a label, not a key.
- **A page on my site** — a picker of the studio's real pages by title, stored as the path (`/about`). Storing the path (not the page id) is deliberate: the same value renders in cached storefront data and in live-preview form state, where relationships may arrive unpopulated.
- **A custom web address** — validated by `sanitizeLinkHref` (`src/components/blocks/hero-links.ts`): paths, anchors, `http(s)`, `mailto:` and `tel:` are accepted; `javascript:`/`data:` are refused and the button is not rendered.

The main button stays *smart* only while untouched: with no custom text and the commission-form destination it still becomes **View Live Queue** while a studio is closed (`ApplySlotButton`). Any custom text or destination means the artist's choice wins. Existing blocks need no edit — `resolveHeroButton` falls back to the old defaults (`/apply`, `View Prices`), and the migration backfills those defaults onto stored rows.

## Hero Background Artwork Crop (`src/components/admin/HeroArtworkField.tsx`)

The hero's **Background artwork** slot is the one image an artist must frame *before* it is stored. It sits full-bleed behind the headline, and the block only offers `object-fit: cover` — so an artwork uploaded whole is cropped by the browser at an arbitrary centre, with no way to choose which part survives.

Payload's stock upload picker has no crop step, and its crop drawer is not a dependable substitute: the output size comes from the drawer's **Width/Height (px)** inputs rather than the drawn rectangle, so a narrowed box can save byte-identical bytes with no error (`collections/hooks/rejectIneffectiveCrop.ts` refuses exactly that for stored files, and `applyCropRectToUploadEdits` rewrites the request — both only on `operation === 'update'`).

So the slot uses its own `UploadFieldClientComponent` instead, configured through `styledImageField({ mediaFieldComponent })`:

- **New upload — crop happens client-side, first.** The chosen file opens `ImageCropModal` (the avatar's `react-easy-crop` editor: drag to place, slider to zoom) with a shape row — Wide 16:9 (default), Classic 3:2, Banner 2:1, Square, Portrait 4:5. The reason there is no single ratio: the frame the artwork covers changes with the viewport, and `object-fit: cover` trims the remainder around the media's focal point. `renderCroppedFile` writes the rectangle at native resolution, capped at `CROP_MAX_SIDE` (3200 px) and never upscaled; the file uploaded to `/api/media` is already the artist's frame, so no server crop step runs at all.
- **Vector (SVG) uploads as-is.** There are no pixels to crop; `isAdjustableImage` is the gate.
- **Stored artwork — `MediaAdjustDrawer`.** The file is on the CDN, so the *server* rewrites it in place with sharp (crop rectangle + focal point), same document, same key, fresh `updatedAt` so the re-versioned `<Image>` in the storefront follows. This is the same drawer the theme brand images use.
- The live readout shows the output pixels and flags a crop below 1400 px on its longest edge as small for a large screen — advice, not a block: an artist focusing on a detail is making a choice.

`resolveCropOutputPixels` is the single calculation behind both the readout and the canvas, so what the editor promises and what the file contains cannot drift.

## Hero Size & Alignment (`src/components/blocks/hero-layout.ts`)

Every hero part — background artwork, headline, intro, badge row and each badge, selling-point row and each point, button row and each button — has an artist-set **width** (20–100% of its row), a **height** where it has a box (the artwork, 20–100% of the section), an optional **offset** (`x`/`y`, how far it has been moved), and **alignment** (left / centre / right / full). Text elements also have a **text size** (70–150%, in 10% steps) and each CTA button has a **button size** (small / medium / large). All are edited in Live Preview, on the element itself:

- **Editing is on by default.** The preview is an editing surface; a corner pill switches to plain browsing when the artist wants to click a link. Hovering an element draws a dashed outline; clicking selects it, posts `PAYLOAD_HERO_ELEMENT_FOCUS`, and reveals its field in the admin.
- **Clicking always lands somewhere.** The field lookup tries the element's own id, then walks up its parents, then the block (`heroFieldIdCandidates`): a button's `buttons.primary` group may have no DOM element of its own, and the old exact-id lookup made a click look like a no-op. The background artwork is tagged too, and gets `pointer-events-auto` in preview so it can be clicked at all (visitors always click through it).
- **Hit-testing looks behind transparent layout.** The content column sits above the artwork with no background of its own, so `event.target.closest('[data-art-el]')` finds nothing across most of the hero and hovering “the background” showed no outline. Hover, click and press therefore resolve the tagged element under the pointer, falling back to a box hit-test (smallest match wins, so a badge beats its row). The element actually under the pointer still wins when there is one.
- **Empty sections are not drawn and not tagged.** A selling-points row with no items used to render an empty padded grid, which the preview measured as a real box — an invisible draggable area. The row is only rendered when it has content, and the overlay ignores anything under 16px tall.
- **The selected element carries the controls.** A right-edge handle resizes width (pointer drag, or ←/→ ±1%, Shift ±5%, Home/End). The artwork also gets a bottom edge and a corner grip for height (↑/↓ on the keyboard), because it is the one element whose box can grow downward. A compact toolbar holds left/centre/right/full, a **text size** stepper (text elements), **S/M/L button sizes** (CTAs), and a reset. Every control posts `PAYLOAD_HERO_LAYOUT_COMMIT` with `width`, `height`, `align`, `fontSize`, `buttonSize`, or a combination. The admin validates the sender (`event.source` must be the preview iframe, and its `src` origin must match), clamps each value, and writes `{ elementKey: { width, height, align, fontSize, buttonSize } }` into the block's **hidden** `layout` field through `dispatchFields`.
- **Text size composes with the theme, it does not replace it.** `fontSize` is stored as a percentage and reaches the type classes as `--art-font-scale`, multiplied alongside `--heading-scale`. A 120% headline is still 120% of whatever the studio's Heading Font Scale lever and each breakpoint already chose, so it cannot pin one screen size. Button sizes are inline padding/type/height overrides (`heroButtonSizeStyle`) shared by the renderer and the preview's local apply; `md` is identity and leaves the component's own classes untouched, `sm` keeps the 44px touch target, and `lg` is a 56px surface.
- **The toolbar sits inside the selected box, and only its buttons take clicks.** It used to float 44px above the element, which put it directly over whatever sat above — the badges row is directly above the headline — so clicking a badge hit the toolbar's buttons and the headline stayed selected. Inside the box it can only cover the element being edited; the container is `pointer-events-none` with `[&>button]:pointer-events-auto`, so any gap lets the click through to the page.
- **A click belongs to the element that was pressed.** Selection happens at `pointerdown` and is remembered for the following `click` (`PRESS_MEMORY_MS`). This matters because `click` fires on mouseup, and the preview can move in between: revealing the pressed element's field scrolls the admin pane, which slides the iframe on screen, so the pointer is over a *different* element at release. Re-hit-testing at click time selected whatever slid under the cursor — press a badge, release over the headline, and the headline became active. The press now wins, and the focus message is sent once rather than again on release.
- **Items are moved, not resized by a box.** A badge, a selling point and a button are sized by their own label, so `heroElementSupportsWidth` and `heroElementSupportsAlign` are false for them: `sanitizeHeroLayout` drops a stored width or alignment, the editor shows no width handle, no alignment and no readout, and the rendered element carries no `--hero-el-w` at all. They still drag, and their toolbar carries the move grip plus **Reset position**. A CTA is the one item with an explicit size lever (`buttonSize`); badges and selling points have none. Text blocks and the artwork keep full width/height/alignment controls, and text blocks additionally own `fontSize`.
- **The badge, selling-point and button rows are not elements.** There is no `row` kind: those rows are plain layout containers the block composes, so they carry no `data-art-el` and never hover, get selected, or compete for a click with the items inside them. Each badge and each button is an independent element — it is what the artist picks, moves and resets. (An `auto` margin on an item would absorb the flex row's free space and shove its siblings, so item placement is the drag offset alone.)
- **A chosen width beats a design cap.** The intro paragraph ships with `max-w-2xl` (a readable measure). In CSS `max-width` wins over `width`, so a capped paragraph stopped following the handle past ~672px, and the next drag started from the capped box and overwrote the chosen value. The cap is now conditional — it applies only while no width is set (`lg:max-w-none` once one is), lives on the sized wrapper, and a width drag always seeds itself from the **stored** `--hero-el-w`, never from a rendered box that may be constrained.
- **A sized element is never the animated one.** The headline always had a plain `<div>` carrying `data-art-el` and the width, with the `<h1>` inside it. The intro applied its width to the `Entrance` `<p>` directly — the same element a transform-based enter animation (`slide-in-from-bottom-6`, `duration-700`, `fill-mode: both`, 150ms delay) is holding, so every width step re-rasterised an animating layer and read as lag. The intro now uses the headline's shape: plain wrapper for the box and the width, `Entrance` inside for the text.
- **Any part can be dragged.** Press the element itself and drag — no move button, because a transparent button over the selected element (the size of the whole block for the artwork) swallowed every other click. While something is selected the hero is outlined as the **drag area**. A move stores `x`/`y` and applies them as `lg:left-[var(--hero-el-x)] lg:top-[var(--hero-el-y)]` on a **relatively** positioned element: it keeps its slot in the column, so its siblings never collapse or reflow — only the element appears moved. Alignment classes stay (a relative offset is measured from the element's own box), and `x`/`y` apply at desktop width only.
- **Moving never changes the size.** Because the element stays in the flow, its width class is the same as before the drag; a moved headline keeps its row width instead of snapping to its `max-content` line. A size is only ever changed by the width handle. An unset half of an offset is `0` — only one axis was dragged — rather than the safety clamp.
- **Offsets are stored in pixels, and that is load-bearing.** `x`/`y` are `lg:left-[var(--hero-el-x)]` / `lg:top-[var(--hero-el-y)]` on a **relatively** positioned element, emitted as `px`. A percentage there resolves against the nearest *block container*, which is not the thing the drag measures against: for a badge inside its flex row that container is the row (~30px tall), so “100%” moved the badge 30px while the same value moved the headline (whose block container is the tall column) hundreds. That is exactly why a badge could never be dragged to the bottom of the hero and a button could never reach the top. Pixels are the one unit where the drag and the CSS agree, whatever ancestor the browser picks.
- **Offsets are bounded by the block.** The editor measures the hero frame against the element's natural box and clamps every drop there, keeping `HERO_FRAME_INSET_PX` from the edges (the block is `overflow-hidden`, so flush reads as clipped). The artwork is the exception to relative positioning: it is already an absolute layer, so there it becomes `left`/`top` inside that layer. The fixed safety clamp (±2000px) is applied **after** the measured bounds, never before.
- **Grid and snapping.** Layout mode draws a 12-column grid (matching `lg:grid-cols-12`) plus horizontal guides at the 8px snap step, toggled from the **Grid** pill. A move snaps to that step so two elements can share an edge; hold **Alt** while dragging for free placement. The guide, the readout and the committed value all use the snapped number, so what the artist sees is what is saved.
- **Nothing leaves the block, and the readout never goes negative.** Two different numbers are in play. The *stored* offset is a percentage of the element's `offsetParent` — a column narrower than the hero — so it is legitimately negative when an element is moved toward the block's left or top edge; that is what makes “drag a badge to the block's bottom” possible at all. The *readout* the artist sees is `framePositionPercent`, measured against the hero block, so it is always 0–100 and means “how far from the edge”. On top of that, `boundsFor` keeps `HERO_FRAME_INSET_PX` (8px) between every element and the hero's edges: the block has `overflow-hidden`, so a flush element would read as cut off even though its coordinate was in range.
- **The artwork is a full-bleed background by design.** It defaults to 100% × 100% of the block (that is what a background *is*), and it can now be resized and moved like anything else — its box is `left-0 top-0` plus `w-full h-full`, never `inset-0`, because fixing both `left` and `right` makes CSS ignore `width` (and the same for `top`/`bottom`/`height`).
- **Reset unpins.** The toolbar's reset sends `{ reset: true }` rather than a set of default values, because “no position” and “positioned at 0,0” are different states; the admin clears the whole entry, returning the element to the flow at its default width and alignment. “Full width” is the same reset, since it is the element's default state rather than a fifth alignment.
- **The `layout` field is hidden** (`admin.hidden`): the hero already exposes headline, intro, badges, selling points and both buttons, and a table-shaped layout panel on top of that was configuration nobody asked for. The value still persists and still reaches the preview through Payload's own form-state broadcast.
- **Widths are desktop only; type scale is not.** Widths travel as `--hero-el-w` / `--hero-el-h` into `lg:w-[…]` / `lg:h-[…]` classes, so phones always show the full-width stack; an untouched block has no inline value and renders exactly as before. Text size and button size apply at every breakpoint. The width/height/offset properties are registered with `@property { inherits: false }` so a size change does not invalidate every descendant's style; `--art-font-scale` is left inheriting on purpose, because the text class that consumes it is a child of the wrapper.
- **Drag cost.** A drag never resizes the page. While the pointer is down, only the editor's guide box and the % readout move (two style writes per frame, coalesced with `requestAnimationFrame`, no React render, hover bookkeeping suspended); the element's `--hero-el-w` (or `left`/`top`, for a move) is written once on release and the commit follows. Resizing the real element per frame reflowed the headline, shifted the vertically-centred column and repainted the hero's `blur(140px)` glows and full-bleed artwork — the difference between “click and release feels fine” and “dragging stutters”. The properties are registered with `@property { inherits: false }`, so that single write invalidates only the element that owns it.
- **Trust boundary.** The preview is a different origin (the studio's subdomain or custom domain), so the admin-side check is window identity plus `src` origin — not the platform-root allowlist, which custom domains are not part of.

## Click-to-Field in the Preview (`src/components/blocks/field-focus.ts`)

Every block's text can point at the field that edits it, without a per-block editor:

- A block tags its text with `data-art-field="<field path>"`, block-relative — `title`, `description`, `questions.0.question`, `links.0.label`, `cards.2.title`. `PageHeader` takes optional `titleField`/`descriptionField` props for the shared header, and leaves them off everywhere else.
- `BlockFieldFocus` (`src/components/live-preview/BlockFieldFocus.tsx`, a lazy preview-only chunk mounted once in `BlocksRenderer`) resolves the block row from the nearest `data-block-index` wrapper, outlines the hovered element, and on click posts `PAYLOAD_BLOCK_FIELD_FOCUS` with `{ blockIndex, field }`.
- `PagesBlocksField` validates that payload (`parseBlockFieldFocusMessage`: a non-negative integer row and a path made of Payload-safe segments) and calls `revealBlockField`, the same scroll-and-flash used by the hero. Nothing is invented: the field path travels from the DOM, so a block without tags stays inert.
- Read-only by design for a click. It outlines and reports; it never edits, resizes or moves. Anything that reshapes a block stays in that block's own editor.
- **Hovering a field on the left points at the element on the right.** `BlockFocusEditor` reads the hovered Payload field wrapper id (`resolveBlockFieldFromId`, scoped to this block's prefix) and broadcasts `action: 'field-hover'` through the same preview transports as isolate/scroll. `BlockFieldFocus` finds the element tagged with that field (exact path first, then the longest tagged prefix, then a group/array's first child), outlines it, scrolls the preview only when it is off-screen, and announces the nearest `data-art-el` key. That block's `HeroPreviewEditor` selects the element, so its alignment/move/text-size/button-size toolbar is already open by the time the artist looks at the preview. A `null` field clears the outline but keeps the selection, so the toolbar survives the trip from the field to the preview.

Tagged so far: hero (artwork, headline, intro, each badge and selling point, both buttons), FAQ (title, description, each question and answer), media gallery (title and each image), rich text (the content area), sectioned content (each section), open rounds (title, subtitle), latest products and featured products (title), and Bento grid (header title/subtitle and each card's title, description, badge and link text).

## Block Links (`src/components/blocks/link-target.ts`)

Any block that sends people somewhere stores the same shape, because the artist only ever has two answers:

- **A page on my site** — picked from the studio's pages by title (`PageLinkField`), stored as the page's **path** (`/about`). A path renders identically from cached page data and from live-preview form state, where a relationship may arrive as a bare id.
- **A custom web address** — a full URL, a path with a query, `mailto:`, `tel:`. `sanitizeLinkHref` refuses `javascript:`/`data:` and bare words, so a stored value is always followable.

`linkTargetFields()` (`src/fields/linkTarget.ts`) builds the Payload group (`destination`, `pagePath`, `url`, `openInNewTab`), and `resolveLinkTarget()` turns it into `{ href, external, newTab } | null` for the renderer: `next/link` for internal destinations, a plain anchor with `target`/`rel` for external ones, nothing at all when the value cannot be followed.

Consumers today: the hero's buttons (through `hero-links.ts`) and the **Bento grid card link** (`src/blocks/BentoGrid.ts` + `HomeBentoGrid.tsx`), which replaced its free-text `linkUrl` box.

Legacy note: the Bento card's old flat `linkUrl` column is *reused* as `link.url`, and `resolveLinkTarget` falls back to `url` when no page was picked. Cards saved before the field change therefore keep their link without a data migration — and the seeds write `{ destination: 'custom', url: '/shop?filter=course' }` for their query-string destinations, which are not page paths.

## Studio Banner Announcements (`src/collections/Announcements.ts`)

Managed under **Media & Content → Announcements** (`src/features/studios/actions/announcements.action.ts`):
- Rich-text banner (`RichTextField`) with an optional custom `backgroundColor` and `type`:
  - `'1'` — **Top Stripe** (displayed at the top of the storefront).
  - `'0'` — **Bottom Stripe** (displayed at the bottom of the storefront).
