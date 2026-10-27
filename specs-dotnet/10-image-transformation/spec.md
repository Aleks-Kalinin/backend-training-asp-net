# 10 — Image Transformation

## 1. Overview

- **Purpose:** Convert images between PNG, JPEG and SVG, including SVG rasterisation.
- **Target roles:** Authenticated users.
- **Dependencies:** SkiaSharp and Svg.Skia, admin-configurable per-format size limits
  ([26](../26-configuration-and-settings/spec.md) §3), the shared concurrency limit and timeout
  from [09](../09-data-file-transformation/spec.md) §6,
  [11](../11-transformation-history/spec.md) (history row),
  [12](../12-transformation-storage/spec.md) (optional result storage).

## 2. Supported conversion matrix

| Source | Targets |
| --- | --- |
| PNG | JPEG |
| JPEG | PNG |
| SVG | PNG, JPEG (rasterisation) |

**Vectorisation — PNG or JPEG to SVG — is not supported** and returns `400`
`unsupported_conversion`.

## 3. Endpoints

| Endpoint | Method | Auth | Rate limit policy |
| --- | --- | --- | --- |
| `/api/v1/images/convert` | POST, `multipart/form-data` | authenticated | `conversion` |
| `/api/v1/images/convert/formats` | GET | authenticated | `conversion` |

### 3.1 Conversion request

| Field | Type | Required | Range / default |
| --- | --- | --- | --- |
| `file` | binary | yes | Non-empty PNG, JPEG or SVG |
| `targetFormat` | string | yes | `png`, `jpeg` or `svg` |
| `quality` | integer | no | 1–100, default 80 — JPEG compression |
| `width` | integer | no | 1–1920 — SVG rasterisation target width |
| `height` | integer | no | 1–1080 — SVG rasterisation target height |
| `background` | string | no | Hex colour, default `#ffffff` |
| `save` | boolean | no | Default `false` |

All options are **typed multipart fields**, each validated independently. Values outside the stated
ranges return `400` with a field-specific message.

**Validation rules:**

1. The file must be present and non-empty.
2. The source must not exceed the admin-configured limit for PNG, JPEG or SVG respectively.
3. The conversion direction must be supported — reject PNG or JPEG to SVG.
4. SVG content must be safe: no scripts, external references or embedded entities (§5).
5. Target dimensions must not exceed the configured system maximum.
6. `quality` must be an integer between 1 and 100.

**Response:** `200`, a streamed binary body, `Content-Type` of `image/png`, `image/jpeg` or
`image/svg+xml`, and `Content-Disposition: attachment; filename="converted.<ext>"`.

### 3.2 Formats discovery

`GET /api/v1/images/convert/formats` returns `ConversionFormatDto[]`:

```json
[
  { "source": "png",  "target": ["jpeg"] },
  { "source": "jpeg", "target": ["png"] },
  { "source": "svg",  "target": ["png", "jpeg"] }
]
```

## 4. Execution

```csharp
public interface IImageConverter
{
    Task ConvertAsync(Stream source, ImageFormat from, ImageFormat to,
                      ImageOptions options, Stream destination, CancellationToken ct);
}
```

| Path | Implementation |
| --- | --- |
| PNG to JPEG | `SKBitmap.Decode`, flatten onto `background`, `SKImage.Encode(Jpeg, quality)` |
| JPEG to PNG | Decode, `Encode(Png)` |
| SVG to PNG or JPEG | `SKSvg.Load`, render to an `SKSurface` at the target size, encode |

**Flow:**

1. Authenticate the caller.
2. Validate the payload, source format and size limit, streaming as in
   [09](../09-data-file-transformation/spec.md) §4.1.
3. For raster sources, decode pixel data and encode to the target.
4. For SVG sources, sanitise (§5), then render to a canvas with the given `background`, `width` and
   `height`, then encode.
5. Stream the result and write the history row.

**Alpha handling:** converting a transparent PNG or SVG to JPEG fills with `background` first.
JPEG has no alpha channel, and the default would otherwise be black.

**SVG sizing:** with neither `width` nor `height` given, use the SVG's intrinsic size; with one
given, derive the other from the aspect ratio. The result is clamped to the configured maximum
dimension, and exceeding the configured bound is `400`, never a silent resize.

## 5. Security and performance

| Requirement | Implementation |
| --- | --- |
| **Decompression bombs** | Read the image header first and reject before full decode when `width * height` exceeds a configured pixel cap, default 50 MP. Header inspection is the only check that runs *before* the allocation it is meant to prevent |
| **SVG sanitisation** | Parse with `XDocument` and DTD disabled, then apply an **allowlist**: permitted elements and attributes only. `<script>`, `<foreignObject>`, `on*` handlers, `href`/`xlink:href` to anything but `data:image/*`, and external `<image>` references are stripped. Anything unrecognised is removed rather than patched |
| **Format verification** | Decided from decoded content, never from the filename or declared MIME type |
| **Size limits** | Per-format admin setting, enforced mid-stream |
| **Timeout and concurrency** | Shares the 30-second limit and the `SemaphoreSlim` from [09](../09-data-file-transformation/spec.md) §6 |
| **Logging** | `transformation.execute` with user id, source and target formats, file size, status and duration. **Never log image bytes** |

There is no mainstream .NET SVG sanitiser, so the allowlist pass is first-party security code. It
is the highest-risk component in this document and needs its own focused test set.

## 6. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `400` | `invalid_image` | Corrupt or undecodable image data |
| `400` | `unsupported_conversion` | Unsupported direction, including PNG or JPEG to SVG |
| `400` | `validation_failed` | Dimensions or quality outside the permitted range |
| `401` | `not_authenticated` | Missing or invalid JWT |
| `413` | `file_too_large` | Source exceeds its format's size limit |
| `415` | `unsupported_media_type` | Unsupported media type uploaded |
| `504` | `conversion_timeout` | Hard per-task timeout reached |

## 7. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 10.1 | SkiaSharp plus Svg.Skia | MIT-licensed, maintained, and covers raster work and SVG rasterisation in one engine | `SixLabors.ImageSharp` (split/commercial licence); `Magick.NET` (Apache-2.0 and very capable, but a large native surface and heavier images — the fallback if Skia's SVG support proves inadequate) |
| 10.2 | Reject vectorisation with an explicit `400` | A clear error beats surprising the caller with a bitmap wrapped in an SVG element | Wrapping a raster in `<image>` |
| 10.3 | Pixel cap checked from the header | A post-decode check runs after the allocation it was meant to prevent | Checking dimensions after decode |
| 10.4 | Allowlist SVG sanitiser | Denylists lose to encoding tricks and newly added attributes | Regex stripping of `<script>`, which is trivially bypassed |
| 10.5 | Typed multipart fields only | One parsing path, one schema, no precedence rule between competing input shapes | Also accepting an `options` JSON blob |
| 10.6 | `background` applied on any alpha-to-JPEG conversion | Without it, transparency becomes black | Attempting to preserve alpha, which JPEG cannot |
| 10.7 | Dimension overflow is `400`, not a clamp | Bounds are validation, and silent resizing surprises callers | Clamping |

## 8. Risks and open points

- **SkiaSharp needs native assets** that minimal container images lack — see
  [30](../30-build-ci-and-docker/spec.md) §3. This is the most likely works-locally,
  fails-in-Docker failure in the project, and the smoke test there is not optional.
- **Svg.Skia does not implement every SVG feature** (filters, some text layout). Complex files may
  rasterise imperfectly rather than failing. Decide whether a degraded render is acceptable or
  should be detected and rejected.
- **The sanitiser is first-party security code.** Treat changes to it as security-sensitive and
  require a second reviewer.
- **`quality` is ignored for PNG targets.** Consider returning `400` instead, for consistency with
  the strict-validation stance elsewhere.

## 9. Acceptance criteria

1. PNG to JPEG, JPEG to PNG, SVG to PNG and SVG to JPEG all succeed.
2. PNG or JPEG to SVG returns `400`.
3. A corrupt image returns `400`; a file whose real format differs from its extension is handled by
   content, not by name.
4. An oversized file returns `413`.
5. An SVG containing `<script>` or an external reference is rejected or rendered inert, and no
   outbound request occurs.
6. An image whose declared dimensions exceed the pixel cap is rejected before decoding.
7. A transparent source converted to JPEG shows the requested background, not black.
8. `quality=0`, `quality=101`, `width=0` and `height=2000` each return `400`.
