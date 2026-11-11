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

## Studio Banner Announcements (`src/collections/Announcements.ts`)

Managed under **Media & Content → Announcements** (`src/features/studios/actions/announcements.action.ts`):
- Rich-text banner (`RichTextField`) with an optional custom `backgroundColor` and `type`:
  - `'1'` — **Top Stripe** (displayed at the top of the storefront).
  - `'0'` — **Bottom Stripe** (displayed at the bottom of the storefront).
