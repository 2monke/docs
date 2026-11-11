---
status: shipped
collections: [digital-assets, media, raw-media, themes]
features: []
---

# Multi-Tenant S3 Storage, Folder Partitioning & Streamed Deliverables

How ArtPack splits public storefront media from purchased/private assets: public media is cached on CloudFront, while digital assets and commission deliverables live in a private bucket and are delivered only through the access-checked streaming gateways.

## Architecture

```
  NEXT_PUBLIC_S3_BUCKET                      S3_PRIVATE_BUCKET
  ┌───────────────────────┐                 ┌───────────────────────────────┐
  │ media/                │                 │ digital-assets/  raw-media/   │
  │ (public)              │                 │ (private)                     │
  └──────────┬────────────┘                 └───────────────┬───────────────┘
             │                                              │
             ▼                                              ▼
      CloudFront CDN                         /api/files/[id] · /api/deliverables/[id]
      (public assets, fonts)                  auth check → same-origin stream
                                              (no presigned GET URL is issued)

  Tenant partitioning stays inside each bucket: tenants/{studioId}/{collection}/…
```

## Key Components

### 1. S3 Bucket Resolution & Upload Signing (`src/lib/infra/s3.ts`)

- **`getS3Client()`**: Singleton `@aws-sdk/client-s3` instance with socket reuse to avoid TLS handshake overhead.
- **`getPublicS3Bucket()` / `getPrivateS3Bucket()`**: the two bucket names. Dev/test fall back to the public bucket while the split is being rolled out; **production requires `S3_PRIVATE_BUCKET` and fails closed (empty bucket) when it is missing.**
- **`getS3BucketForCollection(collection)`**: `media` → public; `digital-assets` / `raw-media` → private.
- **`isPrivateS3Configured()`**: credentials plus a usable private bucket (dev/test may use the public fallback; production requires the real bucket).
- **`generatePresignedUploadUrl(options)`**: expiring PUT URLs for client-to-S3 uploads. There is deliberately **no presigned-GET helper** — a download URL that leaves the server is a reusable bearer capability.
- **`resolveS3Key(filename, prefix?, collection?)`**: Converts relative filenames to canonical S3 keys (e.g. `digital-assets/tenants/1/model.zip`).

### 2. Idempotent S3 Folder Prefixing Hook (`src/collections/hooks/prefixMediaFilename.ts`)

- Attached as `beforeChange` hook across `Media`, `RawMedia`, and `DigitalAssets`.
- Automatically resolves the document's `tenantId` (from doc or request context).
- Formats filename as `tenants/${tenantId}/${collectionSlug}/${filename}`.
- **Idempotent Guard**: Skips re-prefixing if `data.filename.startsWith('tenants/')` on subsequent updates.

### 3. Dual-Mode Secure Streaming & Download Gateway (`src/app/api/files/[id]/route.ts`)

Protects digital assets and course files through a dual-delivery model:

#### Access & Authentication Check:
1. Validates `payload-token` cookie from session.
2. If customer, validates product purchase ownership via `getCachedAssetAccess(user.id, assetId)`.

#### Mode A: Course Viewing & PDF Streaming (`SecurePDFViewerClient` / Video Player)
* **Triggered When**: Request is for inline viewing (`?inline=true`, `?preview=true`, `x-preview: true`, `x-mobile-optimized: true`) or media format is PDF/Video/Audio.
* **Delivery**: Direct server-side streaming from S3 via `transformToWebStream()`.
* **Benefits**:
  - Zero CORS redirect errors for `pdf.js` with `withCredentials: true`.
  - Supports HTTP `206 Partial Content` and `Range: bytes=...` for smooth video seeking.

#### Mode B: Downloads (ZIP, PSD, Deliverables)
* **Triggered When**: Buyer clicks "Download Resource" or requests binary archives (`.zip`, `.psd`, `.rar`, `.7z`, `?download=true`).
* **Delivery**: Same server-side stream as Mode A, with `Content-Disposition: attachment`.
* **No presigned URL is issued**, so there is no reusable S3 link to leak or replay. The trade-off is server egress/bandwidth for large files; app-streaming is the price of not handing the browser a bearer capability.

### 4. CDN URL Generator (`src/lib/utils/cdn.ts`)

* Computes public asset URLs for `Media` and `Themes`.
* If `CDN_URL` is configured, serves through CloudFront (`https://cdn.vronyakat.com/tenants/1/media/banner.webp`).
* Appends cache-busting version queries (`?v=timestamp`) on file updates.
* Falls back to local storage endpoints `/api/media/file/...` in local development without AWS credentials.

## Summary of Files

| File | Purpose |
| :--- | :--- |
| [`src/lib/infra/s3.ts`](file:///Users/eugenepucko/Projects/artpack/src/lib/infra/s3.ts) | S3 client singleton, bucket resolution & upload signing |
| [`src/app/api/files/[id]/route.ts`](file:///Users/eugenepucko/Projects/artpack/src/app/api/files/[id]/route.ts) | Access-checked streaming gateway for digital assets (no presigned downloads) |
| [`src/app/api/deliverables/[id]/route.ts`](file:///Users/eugenepucko/Projects/artpack/src/app/api/deliverables/[id]/route.ts) | Access-checked streaming gateway for raw-media deliverables |
| [`src/collections/hooks/prefixMediaFilename.ts`](file:///Users/eugenepucko/Projects/artpack/src/collections/hooks/prefixMediaFilename.ts) | Auto-prefixes uploads with `tenants/{id}/{collection}/` |
| [`src/lib/utils/cdn.ts`](file:///Users/eugenepucko/Projects/artpack/src/lib/utils/cdn.ts) | CloudFront CDN URL generator with versioning |
| [`src/collections/DigitalAssets.ts`](file:///Users/eugenepucko/Projects/artpack/src/collections/DigitalAssets.ts) | Upload collection for private course & store assets |
| [`src/collections/Media.ts`](file:///Users/eugenepucko/Projects/artpack/src/collections/Media.ts) | Upload collection for public artist assets |
| [`src/collections/RawMedia.ts`](file:///Users/eugenepucko/Projects/artpack/src/collections/RawMedia.ts) | Upload collection for commission references |

## Bucket split & migration

- `NEXT_PUBLIC_S3_BUCKET` — public media (`media/`), CloudFront origin.
- `S3_PRIVATE_BUCKET` — private digital assets + raw media. Required in
  production (private storage fails closed when unset); dev/test may fall back
  to the public bucket while migrating.

AWS setup for the private bucket:

1. Block all public access, `BucketOwnerEnforced` ownership, default encryption.
2. Grant the app's IAM role `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`,
   `s3:AbortMultipartUpload` on the objects and `ListBucket` on the bucket.
   Do not add a public-read policy.
3. CORS for the private bucket: `AllowedOrigins: ["*"]` (artists upload
   deliverables from arbitrary studio custom domains), `AllowedMethods:
   ["PUT", "POST", "HEAD"]` — deliberately **no `GET`** — `AllowedHeaders:
   ["*"]`, `ExposeHeaders: ["ETag"]`, short max-age. No private presigned GET
   URL is issued, so browser JS cannot read private content cross-origin.
4. Keep the private bucket off the public CloudFront distribution. If a CDN is
   added later, use a private distribution with OAC + signed URLs/cookies.

Copy existing private-prefix objects with the **same keys** before deploying:

```bash
aws s3 sync "s3://$NEXT_PUBLIC_S3_BUCKET/digital-assets/" "s3://$S3_PRIVATE_BUCKET/digital-assets/"
aws s3 sync "s3://$NEXT_PUBLIC_S3_BUCKET/raw-media/" "s3://$S3_PRIVATE_BUCKET/raw-media/"
# dev/test deployments use the same prefixes under dev/ or test/
aws s3 sync "s3://$NEXT_PUBLIC_S3_BUCKET/dev/digital-assets/" "s3://$S3_PRIVATE_BUCKET/dev/digital-assets/"
aws s3 sync "s3://$NEXT_PUBLIC_S3_BUCKET/dev/raw-media/" "s3://$S3_PRIVATE_BUCKET/dev/raw-media/"
# test deployments use the same prefixes under test/
aws s3 sync "s3://$NEXT_PUBLIC_S3_BUCKET/test/digital-assets/" "s3://$S3_PRIVATE_BUCKET/test/digital-assets/"
aws s3 sync "s3://$NEXT_PUBLIC_S3_BUCKET/test/raw-media/" "s3://$S3_PRIVATE_BUCKET/test/raw-media/"
```

Set `S3_PRIVATE_BUCKET`, deploy, and verify both `/api/files/<id>` and
`/api/deliverables/<id>` (inline + download) before deleting the old prefixes.
Also scope the public CloudFront distribution (or its bucket policy) to
`media/*`: the old private objects remain in the public bucket until the copy
is verified, and the public CDN must not be able to serve them.

## Debug & Verification Reference

1. **Verify the JSON output is an app URL, not an S3 URL**:
   ```bash
   curl -H "Cookie: payload-token=..." \
        -H "Accept: application/json" \
        http://localhost:3000/api/files/123
   ```
   **Expected Response**:
   ```json
   {
     "url": "http://localhost:3000/api/files/123?download=1",
     "filename": "file.zip",
     "filesize": 104857600
   }
   ```
   There must be no `X-Amz-*` query string: private downloads never hand out a reusable S3 capability.

2. **Verify Course PDF Streaming (HTTP 200)**:
   ```bash
   curl -I -H "Cookie: payload-token=..." \
        -H "x-preview: true" \
        http://localhost:3000/api/files/123
   ```
   **Expected Headers**:
   ```http
   HTTP/1.1 200 OK
   Content-Type: application/pdf
   Content-Disposition: inline; filename="course.pdf"
   Accept-Ranges: bytes
   ```
