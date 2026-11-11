---
status: shipped
collections: [artists, commissions, customers, digital-assets, media, products, raw-media, studios, themes]
features: []
---


# Media Uploads, File Size Limits, Formats & Storage Tiers

> **Status**: Shipped

Reference for ArtPack's three upload collections (`media`, `raw-media`, `digital-assets`), maximum file size limits, allowed image and font MIME types, crop guards, and private delivery endpoints.

## Three Upload Collections

1. **Public Media (`src/collections/Media.ts`, slug `media`)**:
   - Used for public storefront images (galleries, product previews, logos, reference images) and custom theme fonts (`Themes.customFontFile` in `src/collections/Themes.ts`).
   - Publicly readable and served directly from the CloudFront CDN (`getCdnUrl` in `src/lib/utils/cdn.ts`).
2. **Original Files (`src/collections/RawMedia.ts`, slug `raw-media`)**:
   - Used for uncompressed original-quality artwork (hero backgrounds, mascots, and private commission deliverables).
   - Generates a `600×600` centre-cropped `thumbnail` image size.
   - When `isPrivate: true` (or when attached to a commission in `src/collections/Commissions.ts`), `url` and `thumbnailUrl` resolve to the authenticated proxy `/api/deliverables/[id]` (and `/api/deliverables/[id]?size=thumbnail`). Only artists/admins can create `raw-media` records (`isArtistOrAdmin`).
3. **Shop Downloads (`src/collections/DigitalAssets.ts`, slug `digital-assets`)**:
   - Private downloadable files attached to digital shop products, with an optional `compressedFile` (**Compressed Mobile Version**) upload field.
   - Restricted to studio owners, admins, and verified buyers (`customers` via `getCachedAssetAccess` in `src/lib/cache.ts`), served through `/api/files/[id]`.

## File Size Limits (`src/lib/utils/media-mime.ts`)

- **30 MB Platform Upload Ceiling (`MAX_UPLOAD_FILE_BYTES`)**:
  - The maximum file size for uploads is **30 MB** (`30 * 1024 * 1024` bytes), enforced across multipart REST/admin uploads, S3 client uploads, and Local API writes via `createRejectOversizedUploadHook` (`src/collections/hooks/rejectOversizedUpload.ts`): `"This file is too large (X MB). The maximum is 30 MB."`
- **10 MB Onboarding Mascot Ceiling (`MAX_MASCOT_IMAGE_BYTES`)**:
  - Unauthenticated onboarding mascot uploads are capped at **10 MB** server-side (5 MB client-side) and verified by magic bytes (`sniffImageMime`).

## Allowed Formats & SVG Security Ban

- **Images (`IMAGE_MIME_TYPES`, allowed in `media` and `raw-media`)**: `image/png`, `image/jpeg`, `image/webp`, and `image/gif`.
- **Custom Fonts (`FONT_MIME_TYPES`, allowed in `media`)**: `.woff2`, `.woff`, `.ttf`, and `.otf` (`font/woff2`, `font/woff`, `font/ttf`, `font/otf`, `application/font-woff`, `application/x-font-ttf`, `application/font-opentype`).
- **Why SVG is Blocked**: `image/svg+xml` is deliberately excluded from `MEDIA_MIME_TYPES` and `IMAGE_MIME_TYPES` because SVG files can embed executable scripts.

## Crop Enforcement & SSRF Protection

- **Crop Guard**: `applyCropRectToUploadEdits` (`src/collections/hooks/applyCropRectToUploadEdits.ts`) and `rejectIneffectiveCrop` (`src/collections/hooks/rejectIneffectiveCrop.ts`) ensure that any crop rectangle requested in the admin uploader is applied and never silently ignored.
- **Safe Fetch Allowlist**: When re-reading S3 files for cropping/focal-point edits, `skipSafeFetch` uses `getTrustedMediaFetchAllowlist()` (`src/lib/utils/cdn.ts`) rather than `true` so arbitrary external URLs cannot be fetched server-side.
