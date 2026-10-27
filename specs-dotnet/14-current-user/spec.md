# 14 — Current User Session Endpoint

## 1. Overview

- **Purpose:** Let a client discover who is logged in, which role they hold, and what they are
  allowed to do.
- **Why it is needed:** auth tokens live in `HttpOnly` cookies
  ([04](../04-authorization-session/spec.md)), so browser code cannot read the JWT payload, and the
  login endpoints return only a message ([03](../03-authentication/spec.md)). Without this
  endpoint the frontend cannot learn its own `userId` — required by every `/users/{userId}` call —
  or decide which admin screens to show.
- **Target roles:** any authenticated user.
- **Dependencies:** [04](../04-authorization-session/spec.md) (authentication),
  [02](../02-rbac/spec.md) (permission expansion), [13](../13-avatar-upload/spec.md) (avatar URL
  resolution).

## 2. Endpoint

| Endpoint | Method | Auth | Rate limit policy |
| --- | --- | --- | --- |
| `/api/v1/auth/me` | GET | `[Authorize]`, **no RBAC permission required** | `profile-read` |

### 2.1 Response

```csharp
public sealed record CurrentUserResponseDto(
    Guid UserId,
    string Email,
    UserStatus Status,
    string? Photo,
    RoleDto? Role,
    IReadOnlyList<string> Permissions);

public sealed record RoleDto(Guid Id, string Name);
```

```json
{
  "userId": "0c6300a2-46c4-4264-af41-7c493d242253",
  "email": "user@example.com",
  "status": "active",
  "photo": "https://<project>.supabase.co/storage/v1/object/public/avatars/<userId>/<uuid>.webp",
  "role": { "id": "a1b2c3d4-...", "name": "ADMIN" },
  "permissions": ["users:read", "users:update", "files:create", "files:read", "history:read"]
}
```

- `role` is a **single object**, because users have exactly one role
  ([17](../17-user-role-assignment/spec.md)). It is `null` only when data is inconsistent, and
  clients must treat `null` as "no permissions".
- `permissions` is sorted and contains no duplicates.
- `photo` is the resolved public URL or `null`, never the stored path.

## 3. Execution logic

1. Authenticate; the user must exist and be `active`.
2. Load the user **from the database**. Do **not** trust the role in the JWT, which may be up to
   15 minutes old after a role change — escaping that staleness is the purpose of this endpoint.
3. Resolve effective permissions from the RBAC cache: the union of every grant of the user's role,
   expanded to `permission:action` strings. **A grant with no actions, or an empty list, expands to
   every action of that permission**, matching the authorization handler's behaviour.
4. Sort, deduplicate and return.

Step 3 calls the same `RbacSnapshot` expansion the authorization handler uses
([02](../02-rbac/spec.md) §2). Sharing the code is the only way to guarantee that the advertised
permissions match the enforced ones.

The response must reflect role and grant changes immediately, because it reads the database and the
RBAC cache rather than the token.

## 4. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `401` | `not_authenticated` | Missing, invalid or expired access token |
| `401` | `not_authenticated` | The user is no longer `active` — blocked, pending or deleted |

The client is expected to try `POST /auth/refresh` once and retry.

## 5. Audit and logging

**Do not log on every call** — the frontend calls this on each app load. Log only failures, at
`warn` level, with the user id. The endpoint is excluded from request logging
([27](../27-observability-and-audit-logging/spec.md) §2).

## 6. Non-functional and security requirements

- Never return `password`, verification data or internal storage paths.
- The `permissions` list is a **UI hint only**. Every endpoint continues to enforce permissions
  server-side; nothing reads this response back as an authorization decision.
- Keep the endpoint cheap: one user lookup plus an in-memory cache lookup, no joins.
- `Cache-Control: no-store` — the response must never be served stale.

## 7. OpenAPI

Documented under the `Authentication` tag with the cookie security scheme and the
`CurrentUserResponseDto` schema, so the generated client receives a typed response
([28](../28-openapi-and-versioning/spec.md)).

## 8. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 14.1 | Read the role from the database, not the JWT | This is the whole point of the endpoint: it reflects role changes immediately | Reading claims, which would reproduce the staleness it exists to escape |
| 14.2 | Shared permission expansion with [02](../02-rbac/spec.md) | The advertised list must match enforcement, including the all-actions rule | A second expansion implementation |
| 14.3 | Permissions as sorted, deduplicated `permission:action` strings | Stable, diffable, and trivially consumable as a client-side set | Nested objects — verbose, harder to test membership |
| 14.4 | `401` for non-active users | A blocked user has no session; `403` would imply a valid one with insufficient rights | `403` |
| 14.5 | No success logging | The endpoint is called on every app load, and would otherwise dominate the log volume | Uniform request logging |
| 14.6 | `no-store`, no ETag | Immediacy is the feature | A short cache, which reintroduces the staleness |

## 9. Risks and open points

- **This is the most-hit authenticated endpoint**, and the database lookup is per call by design.
  If it becomes hot, cache the *user row* briefly — never the permission expansion, which must
  stay immediate.
- **`permissions` is a UI hint**, and is exactly the kind of field a frontend eventually starts
  trusting for authorization. Restate that in the OpenAPI description.
- **`role: null` signals inconsistent data** — a user with no role row, which the schema permits
  (at most one, not exactly one). Log it at `warn` when it occurs.

## 10. Acceptance criteria

1. An authenticated active user receives their `userId`, `email`, `status`, `photo`, `role` and
   expanded `permissions`.
2. After an admin changes a user's role or a role's grants, the next call returns the new role and
   permissions without logging in again.
3. A request without a valid access token returns `401`.
4. A blocked or pending user receives `401`.
5. The response never contains `password` or a raw storage path.
6. The OpenAPI document shows the endpoint with a typed response schema.
7. Unit tests cover permission expansion, including all-actions grants; integration tests cover
   success, `401`, and the role-change case.
