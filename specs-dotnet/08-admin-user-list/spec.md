# 08 — Admin User List & Search

## 1. Overview

- **Purpose:** Provide administrators with a paginated, filterable, searchable list of registered
  users.
- **Target roles:**
  - **Admin** — a caller holding the `users:list` permission.
  - **Everyone else** — access prohibited, `403 Forbidden`.
- **Dependencies:** [04](../04-authorization-session/spec.md), [02](../02-rbac/spec.md),
  [22](../22-persistence-and-migrations/spec.md) §4 (keyset pagination) and §3.1 (indexes),
  [13](../13-avatar-upload/spec.md) (avatar URL resolution).

## 2. Endpoint

| Endpoint | Method | Auth | Rate limit policy |
| --- | --- | --- | --- |
| `/api/v1/admin/users` | GET | `[RequirePermission("users","list")]` | `admin-search` |

There is no `AllowSelf` on this endpoint — it is admin-only. A caller without the permission
receives `403`.

```csharp
public sealed record ListUsersQuery(
    string? Cursor,
    int Limit = 20,
    string? Q = null,
    UserStatus? Status = null,
    UserSort Sort = UserSort.CreatedAt,
    SortOrder Order = SortOrder.Desc);
```

| Parameter | Rules |
| --- | --- |
| `cursor` | Opaque, signed; bound to the active `sort` and `order` |
| `limit` | Default 20, **max 100**; out of range is `400`, not a silent clamp |
| `q` | Matches `email`, `id` or name fields |
| `status` | `active`, `blocked` or `deleted` |
| `sort` | `created_at` (default), `last_login` or `email` |
| `order` | `asc` or `desc` (default) |

### 2.1 Response

```json
{
  "items": [
    {
      "userId": "string",
      "email": "string",
      "photo": "string | null",
      "status": "string",
      "isVerified": true,
      "createdAt": "ISO-8601",
      "updatedAt": "ISO-8601",
      "role": { "id": "string", "name": "string" }
    }
  ],
  "nextCursor": "string | null"
}
```

The item type is `AdminUserResponseDto` — the admin field set from
[05](../05-user-profile-view/spec.md) §3, applied through the same mapper. The wrapper is
`PaginatedUsersResponseDto`, and both are named schemas in the OpenAPI document
([28](../28-openapi-and-versioning/spec.md) §3).

`photo` is the public URL resolved from the stored avatar key, or `null`. It is never an externally
supplied URL, and the raw storage path is not exposed.

## 3. Query execution

Keyset pagination, per [22](../22-persistence-and-migrations/spec.md) §4:

```sql
SELECT ... FROM users
WHERE (@status IS NULL OR status = @status)
  AND (@q IS NULL OR email ILIKE @pattern OR id::text = @q)
  AND (sort_col, id) < (@cursorKey, @cursorId)
ORDER BY sort_col DESC, id DESC
LIMIT @limit + 1;
```

- `sort` and `order` map to a **whitelist** of column expressions. They are never
  string-concatenated into SQL.
- A cursor issued for one `sort`/`order` combination is invalid for another — `400`.
- `limit + 1` rows are fetched so `nextCursor` is computed without a second query. **No
  `COUNT(*)`.**
- `q` uses a GIN trigram index on `email`; an exact Guid parse covers `id`.

Required indexes: unique `email`; `(status, created_at desc)`; `(last_login_at desc)`; GIN trigram
on `email`.

## 4. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `400` | `invalid_cursor` | Malformed, mis-signed, or wrong-sort cursor |
| `400` | `validation_failed` | `limit` out of range, unknown `sort`, `order` or `status` |
| `401` | `not_authenticated` | Missing or invalid JWT |
| `403` | `permission_denied` | Caller lacks `users:list` |
| `429` | `rate_limited` | Admin-search limit exceeded |

## 5. Audit and logging

- `admin.users.list` with actor id, the **names** of applied filter parameters, status code and
  returned record count.
- **Do not log the raw `q` value** — it may contain PII.
- Log `403` attempts with the actor id.

## 6. Non-functional and security requirements

- **Sensitive data:** responses omit password hashes, secrets, internal flags and security tokens
  by construction — no response DTO has those properties.
- **Performance:** cursor pagination is mandatory, and the sort and search columns must be indexed.

## 7. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 08.1 | Permission expressed as `users` + `list` | Matches the permission-plus-action model in [02](../02-rbac/spec.md) and the seed data in [17](../17-user-role-assignment/spec.md) §5 | A dotted permission name such as `users.list.admin`, which the model does not support |
| 08.2 | `400` on an out-of-range `limit` | Silent clamping hides client bugs | Clamping to 100 |
| 08.3 | Whitelisted sort columns | The only safe way to expose a user-selectable ordering | Dynamic LINQ or string interpolation |
| 08.4 | `AdminUserResponseDto` as the item type | Generated clients need a named item schema, not an inline anonymous one | Inline schema |
| 08.5 | GIN trigram index for `q` | `ILIKE '%term%'` is unindexable otherwise, and this endpoint is explicitly performance-sensitive | Full-text search (wrong tool for substring matching on emails); unindexed scan |
| 08.6 | `q` excluded from logs | It may carry PII | Logging the raw query |

## 8. Risks and open points

- **The permission attribute is the whole security of this endpoint.** Its absence is invisible in
  review — nothing looks wrong. Add an architecture test asserting that every action under an
  `/admin/` route carries a `RequirePermission` attribute, so the class of mistake is caught
  rather than this one instance.
- **Trigram index maintenance** costs write throughput on `users`. Acceptable at this scale.
- **Deleted users appear in listings** with scrubbed PII when `status=deleted` is requested. That
  is intended; the admin UI should make it visually obvious.

## 9. Acceptance criteria

1. A caller without `users:list` receives `403`; an admin receives the list.
2. An unauthenticated caller receives `401`.
3. Items carry `status`, `isVerified`, `createdAt`, `updatedAt`, `photo` and `role`.
4. Paging through the full set with `nextCursor` returns every user exactly once, with no
   duplicates or gaps across concurrent inserts.
5. `limit=101` and `limit=0` both return `400`.
6. A cursor issued under one sort order is rejected with `400` under another.
7. A tampered cursor returns `400`.
8. No response contains a password hash or a raw storage path.
