# Feature: Data File Transformation (CSV, JSON, XML, YAML)

## 1. Overview

- **Purpose:** Convert structured data files between CSV, JSON, XML, and YAML formats.
- **Target Roles:** Authenticated Users (`access_token` JWT cookie required).
- **Dependencies:** Parsing and serialization engines, admin-configurable per-format file size limits, streaming file processing engine.

## 2. Supported Conversion Matrix

The system supports 12 total bidirectional conversion flows:

- **CSV:** CSV ↔ JSON, CSV ↔ XML, CSV ↔ YAML
- **JSON:** JSON ↔ CSV, JSON ↔ XML, JSON ↔ YAML
- **XML:** XML ↔ JSON, XML ↔ CSV, XML ↔ YAML
- **YAML:** YAML ↔ JSON, YAML ↔ CSV, YAML ↔ XML

## 3. Technical Contract

### 3.1 File Conversion Endpoint

- **Endpoint:** `POST /api/convert`
- **Content-Type:** `multipart/form-data`
- **Input Parameters:**
  - `file` (binary, required): Source data file.
  - `targetFormat` (string, required): Allowed values: `"csv"`, `"json"`, `"xml"`, `"yaml"`.

- **Validation Rules:**
  1. `file` must be non-empty and present in payload.
  2. Source format is auto-detected via MIME type, file extension, and content sniffing.
  3. Source file size must not exceed the admin-defined limit configured specifically for that source format.
  4. `targetFormat` must be one of the 4 supported enum values.
  5. The requested source → target conversion pair must be valid.
  6. Source file must be syntactically valid UTF-8/BOM-handled structured data.

- **Execution Logic:**
  1. Authenticate caller via JWT.
  2. Validate input constraints and source format size limit.
  3. Parse input stream into intermediate internal representation (AST/Object model).
  4. Transform intermediate representation according to target format mapping rules.
  5. Serialize target payload.
  6. Return output as a streamed file response.

- **Headers & Response:**
  - `200 OK`: Streamed binary file output.
  - `Content-Type`: Matching target format (`text/csv`, `application/json`, `application/xml`, `application/x-yaml`).
  - `Content-Disposition`: `attachment; filename="converted.<ext>"`

### 3.2 Formats Discovery Endpoint

- **Endpoint:** `GET /api/convert/formats`
- **Access:** Authenticated users only.
- **Response Payload (`200 OK`):**
  ```json
  [
    { "source": "csv", "target": ["json", "xml", "yaml"] },
    { "source": "json", "target": ["csv", "xml", "yaml"] },
    { "source": "xml", "target": ["json", "csv", "yaml"] },
    { "source": "yaml", "target": ["json", "csv", "xml"] }
  ]
  ```

## 4. Ambiguity & Mapping Handling Rules

- **CSV → JSON:** Map rows as an array of JSON objects (headers as keys).
- **XML → JSON:** Attributes mapped to `@attributeKey` properties; repeated tags mapped to JSON arrays.
- **JSON → XML:** Wrap output in a default root element (`<root>`); map array items to `<item>` elements.
- **Encoding:** Enforce UTF-8 parsing; strip and handle Unicode Byte Order Marks (BOM) gracefully.

## 5. Error Handling & HTTP Statuses

- `400 Bad Request`: Syntax error in source file, invalid parameters, or structural parsing failure.
- `401 Unauthorized`: Missing or invalid auth token cookie.
- `413 Payload Too Large`: Source file exceeds the configured size limit for its input format.
- `415 Unsupported Media Type`: Unrecognized or unsupported source file format.

## 6. Audit & Security Requirements

- **Logging:** Log `userId`, `sourceFormat`, `targetFormat`, `fileSize`, status code, and processing duration. Never log raw file contents.
- **XXE Defense:** Explicitly disable external entity resolution (XXE) and DTD processing in XML parsers.
- **Thread Safety:** Execute CPU-bound conversion jobs in worker threads/processes to prevent blocking the main NestJS event loop.
- **Execution Timeout:** Enforce a hard timeout (e.g., 30 seconds) per conversion task.
