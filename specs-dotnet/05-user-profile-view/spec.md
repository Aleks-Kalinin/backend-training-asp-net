# 05 — User Profile View

## 1. Overview

- **Purpose:** Retrieve user profile data under role-based access rules and field-level
  permissions.
- **Target roles:**
  - **Self** — an authenticated user requesting their own profile.
  - **Admin / support** — a caller holding the `users:read` permission, requesting another user's
    profile.
- **Dependencies:** [04](../04-authorization-session/spec.md) (access token),
  [02](../02-rbac/spec.md) (permission check), [13](../13-avatar-upload/spec.md) (avatar URL
  resolution).

## 2. Endpoint

| Endpoint | Method | Auth | Rate limit policy |
| --- | --- | --- | --- |
| `/api/v1/users/{userId}` | GET | `[AllowSelf("userId")] [RequirePermission("users","read")]` | `profile-read` |

**Access control:**

1. **Self access** — granted when the path `userId` equals the caller's `sub`.
2. **Admin access** — granted when the caller holds `users:read`.
3. Neither — `403 Forbidden`.

## 3. Response field masking (default-deny)

Field filtering happens server-side. The field set is chosen in this order:

| # | Condition | Field set |
| --- | --- | --- |
| 1 | `callerId == userId` | **Self** |
| 2 | Caller holds `users:read`, checked via `IRbacCache` and **never** by role name | **Admin** |
| 3 | Otherwise | **Public** — unreachable on this endpoint, since §2 already returned `403` |

| Field set | Fields |
| --- | --- |
| Self | `userId`, `email`, `status`, `isVerified`, `createdAt`, `updatedAt`, `photo`, `role` |
| Admin | `userId`, `email`, `status`, `isVerified`, `createdAt`, `updatedAt`, `photo`, `role` |
| Public | `userId`, `email`, `photo` |

`role` is a single object `{ id, name }` — users have exactly one role
([17](../17-user-role-assignment/spec.md)).

Fields not in the selected set are **omitted** from the response object, not nulled.

### 3.1 Implementation

The sets are domain data, and each has a concrete response DTO:

```csharp
// Domain/Users/UserProfileFields.cs — the policy, and the test oracle
public static class UserProfileFields
{
    public static readonly FrozenSet<UserField> Self;
    public static readonly FrozenSet<UserField> Admin;
    public static readonly FrozenSet<UserField> Public;
}
```

| DTO | Used for |
| --- | --- |
| `SelfUserResponseDto` | self-scoped responses |
| `AdminUserResponseDto` | admin detail, and the item type of `PaginatedUsersResponseDto` |
| `PublicUserResponseDto` | everything else |

A unit test asserts that each DTO's property names equal its field set exactly. That is what keeps
default-deny enforced while OpenAPI still receives named, typed schemas
([28](../28-openapi-and-versioning/spec.md) §3).

**Default-deny in practice:** adding a column to `users` changes no response until the field is
added to a set *and* to a DTO. The equality test fails loudly if only one of the two is done.

The same mapper serves every endpoint that returns a profile: this one,
`PATCH /users/{userId}`, `POST /users`, `PUT`/`DELETE /users/{userId}/photo`, and the admin list
in [08](../08-admin-user-list/spec.md).

## 4. Avatar URL resolution

`photo` in a response is the **public URL** resolved from the stored object key, or `null` when no
avatar is set. The raw storage path is never serialised.

```csharp
public interface IAvatarUrlResolver { string? Resolve(string? storageKey); }
```

Two adapters matching the configured storage driver: Supabase public URL, or the local static
route used in end-to-end runs ([13](../13-avatar-upload/spec.md) §5). Resolution happens inside
the mapper, so every profile-returning endpoint behaves identically.

## 5. Execution logic

1. Validate the access token and extract the caller context.
2. Resolve authorization (self versus `users:read`); record which matched.
3. Load the target user. **Not found returns `404` only after authorization has passed**, so a
   caller with neither self access nor `users:read` receives `403` for a real and a fake id alike.
4. Select the field set, map to the matching DTO, resolve `photo`.
5. Return `200`.

## 6. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `401` | `not_authenticated` | Missing, invalid or expired access token |
| `403` | `permission_denied` | Caller requesting another user's profile without `users:read` |
| `404` | `user_not_found` | Target user does not exist, and the caller was authorized |
| `429` | `rate_limited` | Profile-read limit exceeded |

## 7. Audit and logging

- `user.profile.view` with viewer id, target id and status code.
- **Never log profile field values.**

## 8. Non-functional and security requirements

- **IDOR prevention:** the caller's id is compared against the path parameter before any database
  query runs.
- **Default-deny exposure:** new profile columns are masked until explicitly added to a field set
  and a DTO.
- **Never expose** the password hash, verification records or the raw avatar storage path.

## 9. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 05.1 | Concrete DTO per field set | Named schemas are required for typed generated clients; dynamic dictionaries produce `unknown` | `Dictionary<string,object?>` projection (untyped, unschematised) |
| 05.2 | Field sets as domain data plus a DTO-equality test | Keeps the default-deny policy explicit and testable while allowing concrete DTOs; the test prevents the two representations drifting | DTOs alone (policy becomes implicit in class shape); projection alone (loses typing) |
| 05.3 | Field set chosen by permission, never by role name | A custom role granted `users:read` must behave as an admin; a role named `ADMIN` without the permission must not | `if (role == "ADMIN")` |
| 05.4 | `403` evaluated before `404` | Avoids an existence oracle for unprivileged callers | Checking existence first |
| 05.5 | `IAvatarUrlResolver` applied inside the mapper | One place where a storage key becomes a URL, shared by every profile-returning endpoint | Resolving in each controller |

## 10. Risks and open points

- **Self and Admin field sets are currently identical.** Keeping them separate is deliberate: they
  will diverge, and merging them now would make that divergence a refactor rather than an edit.
- **Admin profile views are audited on every call.** If an admin UI polls this endpoint, the audit
  stream becomes noisy ([27](../27-observability-and-audit-logging/spec.md) §9).

## 11. Acceptance criteria

1. A user fetching their own profile receives the self field set.
2. A caller with `users:read` fetching another profile receives the admin field set.
3. A caller with neither receives `403`, for both existing and non-existing target ids.
4. An unauthenticated caller receives `401`.
5. `photo` is a resolved public URL or `null`, never a storage path.
6. No response contains `password` or verification data.
7. Unit tests cover all three mapper branches and the DTO/field-set equality.
