# 12 — Transformation Result Storage & File Download

## 1. Overview

- **Purpose:** Let users opt in to saving conversion output to backend storage. Saved files remain
  downloadable from transformation history until the retention period expires.
- **Target roles:**
  - **Self** — saves and downloads their own transformed files.
  - **Admin** — a caller with `history:read` can download any user's saved files.
- **Dependencies:** a storage abstraction (local filesystem, S3-compatible),
  [11](../11-transformation-history/spec.md) (history rows and retention),
  [25](../25-background-jobs/spec.md) (asynchronous persistence and pruning),
  [09](../09-data-file-transformation/spec.md) and [10](../10-image-transformation/spec.md)
  (producers).

## 2. Conversion request extension

Both `POST /api/v1/convert` and `POST /api/v1/images/convert` accept an additional multipart
field:

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `save` | boolean | `false` | Request server-side storage of the output file |

**Execution:**

1. Run the normal conversion pipeline.
2. Stream the converted file back to the client **immediately**. The storage operation must not
   block or delay the response.
3. If `save` is true and the conversion succeeded, persist the output, generate a unique `fileId`,
   attach it to the corresponding history item, and set the file's expiry to match the history
   retention TTL.
4. If `save` is false, write the history row with no attached file.

Persistence runs in `PersistTransformationResultJob` ([25](../25-background-jobs/spec.md) §3),
which uploads the buffered output and attaches the resulting `StoredFile` row.

**Consequence:** a download attempted in the seconds between the response and job completion
returns `404`. `Downloadable` on the history item is false until the file exists, so a
well-behaved client never hits this. Document it in the OpenAPI description.

## 3. Storage abstraction

```csharp
public interface IResultStorage
{
    Task<string> SaveAsync(Stream content, string contentType, TimeSpan ttl, CancellationToken ct);
    Task<Stream?> OpenAsync(string storageKey, CancellationToken ct);
    Task DeleteAsync(string storageKey, CancellationToken ct);
}
```

Adapters live in `Infrastructure/Storage/`:

| Adapter | Use |
| --- | --- |
| `LocalResultStorage` | Filesystem — development and end-to-end runs |
| `S3ResultStorage` | `AWSSDK.S3`, which also covers MinIO via a custom `ServiceURL` with `ForcePathStyle` |

Selected by `Storage:Results:Driver`. File operations are encapsulated behind this interface so a
backend can be swapped without altering core domain logic.

The storage key is server-generated — `{userId}/{yyyy}/{MM}/{guid}.{ext}` — and **never leaves the
server**: not in responses, not in logs. Clients address files by the history item id only.

## 4. Download endpoints

| Endpoint | Method | Auth | Rate limit policy |
| --- | --- | --- | --- |
| `/api/v1/transformations/history/{itemId}/download` | GET | authenticated, self-scoped | `download` |
| `/api/v1/admin/users/{userId}/transformations/history/{itemId}/download` | GET | `[RequirePermission("history","read")]` | `download` |

**Self:**

1. Authenticate the caller.
2. Fetch the history item **scoped to the caller's id**, so another user's item is simply not
   found.
3. Verify a `fileId` is present and the file has not expired.
4. Open the stream from `IResultStorage` and proxy it.

**Admin:**

1. Authenticate and verify `history:read`.
2. Verify the target user exists.
3. Fetch the item matching both `itemId` and that user id.
4. Verify a non-expired `fileId` exists.
5. Stream the payload.

**Response headers:** `200`, `Content-Type` matching the file's format, and
`Content-Disposition: attachment; filename="<original_name>"`. `Content-Disposition` must be in
the CORS exposed-headers list or the browser cannot read the filename
([31](../31-security-rate-limiting-and-cors/spec.md) §2).

## 5. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `401` | `not_authenticated` | Missing or invalid JWT |
| `403` | `permission_denied` | Non-admin calling the admin endpoint |
| `404` | `history_item_not_found` | Item does not exist, or belongs to another user |
| `404` | `file_not_found` | No file was ever saved for this item |
| `410` | `file_expired` | The record shows a file existed, but retention has purged it |
| `429` | `rate_limited` | Download limit exceeded |
| `500` | `storage_unavailable` | Storage provider read or write failure |

A `500` body stays generic: no bucket names, no provider messages, no keys.

## 6. Lifecycle and retention

- A stored file's expiry is set from the same `transformation_retention_days` setting that governs
  its history row, so the two expire together.
- `TransformationRetentionJob` deletes the physical object **first** and the database row
  **second**. If the object delete fails, the row is left for the next run — an orphaned row is
  recoverable, an orphaned object is not discoverable.

## 7. Audit and logging

- `storage.operation` with user id, transformation id, file id, action (`save` or `download`),
  result status, file size and duration.
- **Never expose or log** raw storage paths, bucket keys, signed URLs or file contents.

## 8. Non-functional and security requirements

- **IDOR and direct-link prevention:** internal storage paths and bucket keys are never exposed.
  All access goes through the authorised streaming endpoints above.
- **Asynchronous persistence:** storage writes are decoupled from the conversion response so they
  add no latency.
- **Lifecycle synchronisation:** background pruning deletes storage assets at the same time as the
  corresponding history records expire.
- **Provider abstraction:** all file operations sit behind `IResultStorage`.

## 9. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 12.1 | `IResultStorage` port with local and S3 adapters | Swappable backends are a stated requirement, and S3 covers MinIO and most clouds with one adapter | A provider-specific client in the service layer |
| 12.2 | Proxy the stream, never redirect | Storage paths must not be exposed, and a pre-signed URL is a bearer credential that outlives the request and bypasses the audit log | Pre-signed URLs |
| 12.3 | Asynchronous persistence via a job | Keeps storage latency off the response path, with retries for free | Inline upload; fire-and-forget `Task` (lost on restart) |
| 12.4 | `404` rather than `403` for another user's item | `403` confirms the item exists, which is an existence oracle | `403` on an ownership mismatch |
| 12.5 | `410` only when a file is known to have expired | Distinguishing "expired" from "never saved" is genuinely useful to a client | Always `404`; always `410` |
| 12.6 | Expiry shared with the history row | One setting, one pruning job, no drift | Independent TTLs |
| 12.7 | Buffer the output before responding | Required in order to both stream the response and hand the bytes to the job; bounded by the per-format size limits | Re-running the conversion inside the job, which doubles CPU |

## 10. Risks and open points

- **Buffering doubles peak memory** for large outputs. The per-format limits bound it; for larger
  files, spool to a temporary file and have the job read from there.
- **The save-then-download race** in §2 is client-visible. `Downloadable` mitigates it; a client
  assuming immediate availability will see `404`.
- **Orphaned objects** are possible if the job uploads and then fails to attach the row. The
  retention job only reaches objects via a row, so add a periodic sweep for unreferenced keys, or
  accept slow leakage.
- **`500` during a provider outage** will be user-visible. Keep the problem detail generic.

## 11. Acceptance criteria

1. `save=true` on a successful conversion eventually produces a downloadable file; `save=false`
   produces a history row with no file.
2. The conversion response is not delayed by storage: response time is comparable with `save` true
   and false.
3. A user can download their own saved file; requesting another user's item returns `404`.
4. An admin with `history:read` can download any user's file; without it, `403`.
5. A purged file returns `410`, and an item that never had a file returns `404`.
6. No response header, body or log line contains a storage key or bucket name.
7. When retention expires an item, both the row and the object are gone.
