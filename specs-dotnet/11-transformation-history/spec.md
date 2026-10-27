# 11 — Transformation History

## 1. Overview

- **Purpose:** Give users and administrators access to the execution history of data-file and image
  transformations.
- **Target roles:**
  - **Self** — can view only their own history.
  - **Admin** — a caller with the `history:read` permission can view any user's history.
- **Dependencies:** [04](../04-authorization-session/spec.md), [02](../02-rbac/spec.md),
  [22](../22-persistence-and-migrations/spec.md) §4 (keyset pagination),
  [09](../09-data-file-transformation/spec.md) and [10](../10-image-transformation/spec.md)
  (producers), [12](../12-transformation-storage/spec.md) (stored files),
  [25](../25-background-jobs/spec.md) (retention job).

## 2. Endpoints

| Endpoint | Method | Auth | Rate limit policy |
| --- | --- | --- | --- |
| `/api/v1/transformations/history` | GET | authenticated, self-scoped | `admin-search` |
| `/api/v1/admin/users/{userId}/transformations/history` | GET | `[RequirePermission("history","read")]` | `admin-search` |

Both accept the same query parameters:

| Parameter | Type | Notes |
| --- | --- | --- |
| `cursor` | string | Opaque, signed |
| `limit` | number | Default 20, max 100 |
| `type` | enum | `file` or `image` |
| `sourceFormat` | string | `csv`, `json`, `xml`, `yaml`, `png`, `jpeg`, `svg` |
| `targetFormat` | string | As above |
| `status` | enum | `success` or `error` |
| `createdAtFrom` | ISO-8601 | Inclusive |
| `createdAtTo` | ISO-8601 | Exclusive |

### 2.1 Response

```json
{
  "items": [
    {
      "id": "string",
      "type": "file | image",
      "sourceFormat": "string",
      "targetFormat": "string",
      "status": "success | error",
      "fileSize": 1024,
      "durationMs": 145,
      "errorCode": "invalid_source_syntax",
      "createdAt": "2026-09-01T16:00:00.000Z",
      "downloadable": true
    }
  ],
  "nextCursor": "string | null"
}
```

```csharp
public sealed record HistoryItemDto(
    Guid Id, TransformationType Type, string SourceFormat, string TargetFormat,
    TransformationStatus Status, long FileSize, int DurationMs,
    string? ErrorCode, DateTimeOffset CreatedAt, bool Downloadable);
```

`Downloadable` is `true` when a non-expired stored file exists for the item
([12](../12-transformation-storage/spec.md)). The internal file relation and any storage path are
never serialised.

Results are ordered by `createdAt` descending.

## 3. Execution logic

**Self endpoint:**

1. Authenticate the caller.
2. Query the history store filtered by the caller's own id — applied **inside the repository**, not
   taken from a parameter. There is no code path on this endpoint that can read another user's
   rows, so the IDOR requirement in §6 is structural rather than a check.
3. Apply the optional filters.
4. Return a paginated, descending-ordered page.

**Admin endpoint:**

1. Authenticate and evaluate the `history:read` permission.
2. Verify the target user exists — `404` if not.
3. Query rows for that user id.
4. Return the same shape.

Pagination is keyset-based on `(created_at desc, id desc)`, sharing the signed-cursor helper with
[08](../08-admin-user-list/spec.md). A cursor is bound to the filter and ordering it was issued
for; a mismatch is `400`.

Date filters parse as ISO-8601 with offset; an unparseable value is `400`.

## 4. Writing history

Every conversion writes exactly one row, on success and on failure, from the conversion pipeline.

- `DurationMs` and `FileSize` come from the same stopwatch and byte counter that feed the metrics
  in [27](../27-observability-and-audit-logging/spec.md) §5, so the API and any dashboard agree.
- `ErrorCode` is the stable error code from [24](../24-error-handling/spec.md) §4, never a
  message.
- **A history-write failure must not fail the request.** It is logged at `error` and swallowed:
  losing an audit row is better than discarding a conversion the user has already paid for in CPU.
- **Never store file contents.** Storing *output* files is a separate opt-in feature
  ([12](../12-transformation-storage/spec.md)).

## 5. Retention

Rows are retained for `transformation_retention_days`, an admin-configurable setting defaulting to
90 ([26](../26-configuration-and-settings/spec.md) §3).

`ExpiresAt` is computed and stamped at write time rather than derived from age at query time, so
the pruning job needs one indexed predicate and a settings change does not retroactively delete
history.

`TransformationRetentionJob` ([25](../25-background-jobs/spec.md) §3.1) runs hourly, deleting the
stored object first and the row second, in batches.

## 6. Non-functional and security requirements

- **IDOR protection:** query scoping guarantees a non-admin cannot reach another user's records.
- **Database performance:** composite index on `(user_id, created_at desc, id desc)`, plus indexes
  on `type`, `status` and `expires_at`.
- **Data retention:** admin-configurable TTL with automated cleanup.

## 7. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `400` | `invalid_cursor` | Malformed, mis-signed or mismatched cursor |
| `400` | `validation_failed` | `limit` out of range, invalid filter or date value |
| `401` | `not_authenticated` | Missing or invalid JWT |
| `403` | `permission_denied` | Non-admin calling the admin endpoint |
| `404` | `user_not_found` | Admin endpoint, unknown target user |
| `429` | `rate_limited` | History-retrieval limit exceeded |

## 8. Audit and logging

- `transformation.history.query` with actor id, target id for admin queries, applied filter
  **keys**, status code and returned item count.
- **Never store or log raw file contents** in history records or application logs.

## 9. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 11.1 | Permission expressed as `history` + `read` | Matches the permission-plus-action model and the seed data in [17](../17-user-role-assignment/spec.md) §5 | A dotted name such as `transformations.history.admin` |
| 11.2 | Self scoping inside the repository | Makes IDOR impossible rather than merely guarded | A `userId` parameter checked in the service |
| 11.3 | Shared keyset helper with [08](../08-admin-user-list/spec.md) | Same semantics, same signing, one set of edge-case tests | A second pagination implementation |
| 11.4 | History-write failures are swallowed and logged | A failed audit insert must not discard a completed conversion | Failing the request |
| 11.5 | `ExpiresAt` stamped at write time | One indexed predicate for pruning; settings changes are not retroactive | Computing age at query or prune time |
| 11.6 | `Downloadable` computed, not stored | It depends on both file existence and expiry; a stored flag would drift | A persisted boolean |

## 10. Risks and open points

- **This is the fastest-growing table in the system**, written on every conversion. The composite
  index plus hourly pruning is the plan; if volume outgrows it, partition by month on `created_at`.
- **Retention changes are not retroactive** (decision 11.5). An administrator lowering the setting
  will reasonably expect old rows to expire sooner. Either document it or add a backfill.
- **`errorCode` is visible to admins** for other users' failed conversions. Harmless as long as
  error codes never embed user content — keep them from doing so.

## 11. Acceptance criteria

1. A user sees only their own history, whatever parameters they send.
2. An admin with `history:read` can read any user's history; without it, `403`.
3. The admin endpoint returns `404` for an unknown user.
4. Each filter narrows results correctly, and combinations compose.
5. Paging with `nextCursor` returns every row exactly once.
6. Both successful and failed conversions produce a row, with `errorCode` set only on failures.
7. A row whose stored file has expired reports `downloadable: false`.
8. No response or log contains a storage path or file contents.
