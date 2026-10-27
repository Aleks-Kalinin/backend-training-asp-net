# 09 — Data File Transformation (CSV, JSON, XML, YAML)

## 1. Overview

- **Purpose:** Convert structured data files between CSV, JSON, XML and YAML.
- **Target roles:** Authenticated users.
- **Dependencies:** Parsing and serialisation engines, admin-configurable per-format size limits
  ([26](../26-configuration-and-settings/spec.md) §3), streaming file handling,
  [11](../11-transformation-history/spec.md) (history row),
  [12](../12-transformation-storage/spec.md) (optional result storage).

## 2. Supported conversion matrix

Twelve bidirectional flows:

| Source | Targets |
| --- | --- |
| CSV | JSON, XML, YAML |
| JSON | CSV, XML, YAML |
| XML | JSON, CSV, YAML |
| YAML | JSON, CSV, XML |

## 3. Endpoints

| Endpoint | Method | Auth | Rate limit policy |
| --- | --- | --- | --- |
| `/api/v1/convert` | POST, `multipart/form-data` | authenticated | `conversion` |
| `/api/v1/convert/formats` | GET | authenticated | `conversion` |

### 3.1 Conversion request

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `file` | binary | yes | Source data file, non-empty |
| `targetFormat` | string | yes | `csv`, `json`, `xml` or `yaml` |
| `save` | boolean | no | Default `false`; see [12](../12-transformation-storage/spec.md) |

**Validation rules:**

1. `file` must be present and non-empty.
2. The source format is auto-detected — see §4.2.
3. The source file must not exceed the admin-configured limit **for its own source format**.
4. `targetFormat` must be one of the four supported values.
5. The source-to-target pair must be in the matrix in §2.
6. The source must be syntactically valid, UTF-8, with any byte-order mark handled.

**Response:** `200`, a streamed binary body, `Content-Type` matching the target
(`text/csv`, `application/json`, `application/xml`, `application/x-yaml`), and
`Content-Disposition: attachment; filename="converted.<ext>"`.

### 3.2 Formats discovery

`GET /api/v1/convert/formats` returns `ConversionFormatDto[]`:

```json
[
  { "source": "csv",  "target": ["json", "xml", "yaml"] },
  { "source": "json", "target": ["csv", "xml", "yaml"] },
  { "source": "xml",  "target": ["json", "csv", "yaml"] },
  { "source": "yaml", "target": ["json", "csv", "xml"] }
]
```

## 4. Execution

```csharp
public interface IFormatCodec
{
    DataFormat Format { get; }
    Task<JsonNode> ReadAsync(Stream source, CancellationToken ct);
    Task WriteAsync(JsonNode document, Stream destination, CancellationToken ct);
}
```

Four codecs — `CsvCodec`, `JsonCodec`, `XmlCodec`, `YamlCodec` — resolved from a keyed registry.
A conversion is `source.ReadAsync` followed by `target.WriteAsync`, so the twelve pairs are eight
implementations, not twelve converters.

The intermediate representation is `System.Text.Json.Nodes.JsonNode`: in-box, models the
object / array / scalar shape all four formats share, and streams on both ends.

**Flow:**

1. Authenticate the caller.
2. Validate inputs and the source-format size limit while streaming (§4.1).
3. Parse the input stream into the `JsonNode` model.
4. Apply the target mapping rules (§4.3).
5. Serialise to the target format.
6. Stream the result, and write the history row
   ([11](../11-transformation-history/spec.md) §4).

### 4.1 Upload handling

Uses `MultipartReader`, not `IFormFile`: the per-format size limit must be enforced **while**
reading, so an oversized file is rejected before it is buffered. Exceeding the limit aborts the
read and returns `413`.

### 4.2 Source-format detection

In order of trust: content sniffing of the first non-whitespace bytes, then the file extension,
then the declared MIME type. Client-supplied metadata is the weakest signal and never the only
one. No match returns `415`.

### 4.3 Mapping and ambiguity rules

| Rule | Behaviour |
| --- | --- |
| CSV to JSON | Rows become an array of objects, the header row supplying the keys |
| JSON to CSV | Flattened to a union of columns, nested paths joined with `.`. A document that cannot be flattened returns `400` `csv_shape_unsupported` |
| XML to JSON | Attributes map to `@attributeName` properties; repeated tags map to arrays |
| JSON to XML | Output is wrapped in a `<root>` element; array items become `<item>` elements |
| Encoding | UTF-8 enforced on parse; byte-order marks stripped on read and never written |

## 5. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `400` | `invalid_source_syntax` | Syntax or structural parsing failure in the source file |
| `400` | `validation_failed` | Invalid parameters |
| `400` | `unsupported_conversion` | Source-to-target pair not in the matrix |
| `400` | `csv_shape_unsupported` | Document cannot be represented as CSV |
| `401` | `not_authenticated` | Missing or invalid auth cookie |
| `413` | `file_too_large` | Source exceeds the configured limit for its format |
| `415` | `unsupported_source_format` | Unrecognised or unsupported source format |
| `504` | `conversion_timeout` | Hard per-task timeout reached |

## 6. Security and performance

| Requirement | Implementation |
| --- | --- |
| **XXE and DTD defence** | `XmlReaderSettings { DtdProcessing = DtdProcessing.Prohibit, XmlResolver = null, MaxCharactersFromEntities = 0 }`. Non-negotiable, and covered by a test feeding a classic XXE payload |
| **YAML gadget defence** | YamlDotNet deserialises to the node model only, never to arbitrary CLR types, so no type-resolution gadget exists |
| **JSON depth** | `JsonSerializerOptions.MaxDepth` bounded at 64 |
| **Execution timeout** | 30 seconds, enforced with a linked `CancellationTokenSource`; expiry returns `504` |
| **Bounded concurrency** | A `SemaphoreSlim` sized by `Conversion:MaxConcurrency`; queue wait is itself bounded, and a full queue returns `429` |
| **Logging** | `transformation.execute` with user id, source and target formats, file size, status code and duration. **Never log raw file contents** |

### 6.1 On CPU-bound work

A synchronous CPU-bound call on .NET blocks a thread-pool thread, not a server-wide event loop, so
no worker-thread offloading is required. The real risks — unbounded parallelism and unbounded
duration — are handled by the concurrency limit and the timeout above. Do not add `Task.Run`
wrappers or a separate worker process to imitate an event-loop design; they would add latency and
no safety.

## 7. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 09.1 | `JsonNode` as the intermediate model | In-box, streams, and models every shape the four formats share | A bespoke AST (more code, same semantics); `ExpandoObject` (untyped, slow) |
| 09.2 | `IFormatCodec` registry, four read plus four write | Turns twelve pairs into eight implementations and makes a fifth format purely additive | A converter class per pair |
| 09.3 | `MultipartReader` streaming | A size limit only protects memory if enforced mid-stream | `IFormFile` with a length check after buffering |
| 09.4 | CsvHelper | Correctly handles quoting, embedded newlines and delimiter edge cases; a hand-written CSV parser is a classic underestimate | Splitting on commas |
| 09.5 | YamlDotNet restricted to the node model | Eliminates the deserialisation-gadget vulnerability class | Deserialising to CLR types |
| 09.6 | `SemaphoreSlim` plus timeout | Expresses the real constraint on a thread-pool runtime | Worker threads; out-of-process workers |
| 09.7 | Sniff, then extension, then MIME | Client-supplied metadata is the least trustworthy input | Trusting `Content-Type` |
| 09.8 | `504` for a timeout | Distinguishes "we gave up" from "your file is bad" | `400`; `408`, which means a client-side read timeout |

## 8. Risks and open points

- **JSON to CSV is lossy, and sometimes impossible.** Deeply nested or heterogeneous documents have
  no faithful tabular form. The flatten-or-`400` rule is a judgement call — confirm it with the
  frontend before shipping.
- **XML and YAML need the whole document in memory** for the node model, so the per-format size
  limits are the real memory bound. Set them conservatively.
- **Sibling ordering is not preserved** when interleaved heterogeneous XML elements become arrays.
  Acceptable for the supported matrix, but document it.

## 9. Acceptance criteria

1. All twelve conversion pairs succeed on representative inputs.
2. A malformed source file returns `400` without leaking parser detail.
3. A file exceeding its format's limit returns `413`, and memory does not grow to the file's size.
4. An unrecognised format returns `415`.
5. An XML file containing an external entity declaration is rejected, and no outbound network or
   file access occurs.
6. A file with a UTF-8 byte-order mark converts correctly, and no BOM appears in the output.
7. A conversion exceeding 30 seconds returns `504`.
8. Concurrent requests beyond the configured limit queue, then return `429` rather than exhausting
   memory.
9. A history row is written for both success and failure.
