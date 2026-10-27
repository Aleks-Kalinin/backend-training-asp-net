# Feature: Image Transformation

## 1. Overview

- **Purpose:** Convert image assets between PNG, JPEG, and SVG formats, including SVG rasterization.
- **Target Roles:** Authenticated Users (`access_token` JWT cookie required).
- **Dependencies:** Image processing engine (e.g., Sharp), per-format admin file size limits, SVG rasterization caps.

## 2. Supported Conversion Matrix

- **PNG → JPEG**
- **JPEG → PNG**
- **SVG → PNG** (Rasterization)
- **SVG → JPEG** (Rasterization)
- **STRICT LIMITATION:** Vectorization (**PNG/JPEG → SVG**) is **NOT supported**.

## 3. Technical Contract

### 3.1 Image Conversion Endpoint

- **Endpoint:** `POST /api/images/convert`
- **Content-Type:** `multipart/form-data`
- **Input Parameters:**
  - `file` (binary, required): Source image file (PNG, JPEG, or SVG).
  - `targetFormat` (string, required): Allowed values: `"png"`, `"jpeg"`, `"svg"`.
  - `options` (JSON string or form fields, optional):
    - `quality` (number, 1–100): JPEG compression quality (default: 80).
    - `width` (number, optional): Target width for SVG rasterization.
    - `height` (number, optional): Target height for SVG rasterization.
    - `background` (string, optional, default: `"#ffffff"`): Hex background color for SVG rasterization or PNG transparency removal.

- **Validation Rules:**
  1. File must be non-empty and present.
  2. Source format size must not exceed the admin-configured limit for PNG, JPEG, or SVG.
  3. Conversion direction must be supported (reject PNG/JPEG → SVG).
  4. SVG content must be safe (no scripts, external links, or embedded entities).
  5. Target dimensions (`width`/`height`) must not exceed system maximum bounds (admin-configured/hardcoded).
  6. `quality` parameter must be an integer between 1 and 100.

- **Execution Logic:**
  1. Authenticate user.
  2. Validate payload, source format, and size limits.
  3. For raster formats (PNG/JPEG): decode pixel data and encode to target format.
  4. For SVG rasterization: render vector paths to canvas with specified `background`, `width`, and `height`, then encode to PNG/JPEG.
  5. Return image stream with appropriate headers.

- **Headers & Response:**
  - `200 OK`: Streamed binary file response.
  - `Content-Type`: `image/png`, `image/jpeg`, or `image/svg+xml`.
  - `Content-Disposition`: `attachment; filename="converted.<ext>"`

### 3.2 Formats Discovery Endpoint

- **Endpoint:** `GET /api/images/convert/formats`
- **Access:** Authenticated users only.
- **Response Payload (`200 OK`):**
  ```json
  [
    { "source": "png", "target": ["jpeg"] },
    { "source": "jpeg", "target": ["png"] },
    { "source": "svg", "target": ["png", "jpeg"] }
  ]
  ```

## 4. Error Handling & HTTP Statuses

- `400 Bad Request`: Corrupted image file, unsupported conversion direction (e.g., PNG → SVG), or requested dimensions exceed maximum bounds.
- `401 Unauthorized`: Missing or invalid JWT.
- `413 Payload Too Large`: Source file exceeds size limit.
- `415 Unsupported Media Type`: Unsupported media type uploaded.

## 5. Security & Performance

- **SVG Sanitization:** Strip `<script>` tags, inline JS handlers, and external link references prior to processing.
- **Decompression Bomb Protection:** Limit maximum allowable pixel count during image decoding to prevent memory exhaustion attacks.
- **Alpha Channel Management:** Fill transparent background with specified background color when converting PNG/SVG with transparency to JPEG.
- **Audit Logging:** Log `userId`, `sourceFormat`, `targetFormat`, `fileSize`, HTTP status, and processing duration. Do not log binary content.
