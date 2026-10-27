# 16 — User Write Policy & User Creation

## 1. Overview

- **Purpose:** Define which fields each caller may write on a user record, and specify
  administrative user creation. This document owns the **writable-field policy** and the
  `POST /users` contract; the endpoints that apply the policy are in
  [06](../06-user-profile-update/spec.md).
- **Target roles:**
  - **Self** — may write only self-writable fields.
  - **Admin** — callers with `users:create` or `users:update`.
- **Dependencies:** [02](../02-rbac/spec.md), [23](../23-validation-and-partial-updates/spec.md)
  (presence detection), [06](../06-user-profile-update/spec.md),
  [15](../15-admin-user-access/spec.md) (response field sets),
  [17](../17-user-role-assignment/spec.md) (roles and protection rules).

## 2. Writable-field policy (default-deny)

```csharp
public enum UserWriteScope { SelfPatch, AdminPatch, AdminCreate }

public static class UserWritePolicy
{
    public static IReadOnlySet<string> Allowed(UserWriteScope scope);
}
```

| Field | Self PATCH | Admin PATCH | Admin POST |
| --- | --- | --- | --- |
| `email` | no — use the email-change flow ([06](../06-user-profile-update/spec.md) §4) | yes | yes, required |
| `status` | no | yes | optional, default `pending` |
| `isVerified` | no | yes | yes, required |
| `password` | no | **no** — use the reset flow ([20](../20-password-management/spec.md)) | yes, required |
| `roleId` | no | no — use [17](../17-user-role-assignment/spec.md) §2 | optional, default `USER` |

**Enforcement:**

1. Compute the set of fields **present** in the request, using `Optional<T>`
   ([23](../23-validation-and-partial-updates/spec.md) §3).
2. Compute the allowed set from the caller's scope. "Admin" means holding `users:update` or
   `users:create`, **not** being named `ADMIN`.
3. Any present field outside the allowed set causes `403` `field_not_writable`. It is
   **rejected, not silently dropped**, and the offending field **names** are logged at `warn`.
   Values are never logged.
4. Only present, allowed fields are applied.

These sets are the test oracle: one test per row of the table, asserting both directions
([29](../29-testing-strategy/spec.md) §2.1).

### 2.1 Update request shape

`UpdateUserRequest` is defined explicitly with `email`, `status` and `isVerified`, all optional.
It **does not derive** from the creation request, and `password` is not part of the update
contract at all. DTO inheritance is how a password field leaks into an update endpoint.

## 3. Create user

| Endpoint | Method | Auth | Rate limit policy |
| --- | --- | --- | --- |
| `/api/v1/users` | POST | `[RequirePermission("users","create")]` | `profile-write` |

**Body:** `email`, `password` (minimum 8 characters), `isVerified`, optional `status`, optional
`roleId` (UUID).

**Execution:**

1. Normalise the email; return `409` `email_already_registered` if it is taken.
2. **Hash the password** through `IPasswordHasher` before saving.
3. If `roleId` is supplied, it must reference an existing role — `400` `unknown_role` otherwise.
   If omitted, assign the default `USER` role.
4. Save, and return the user through the mapper using the admin field set
   ([15](../15-admin-user-access/spec.md) §3), with status `201`.
5. Audit `user.create` with the assigned role name.

The response is produced by the mapper, never by serialising the entity, so no response can carry a
password hash or an internal field.

## 4. Single hashing path

Every code path that sets a password must hash it, exactly once. This is enforced structurally
rather than by convention:

- `User.PasswordHash` is settable only through `User.SetPassword(PasswordHash hash)`.
- `PasswordHash` is a value type that only `IPasswordHasher.Hash(Password raw)` can produce.
- A caller holding a plain string therefore cannot construct one, and cannot reach the setter.
- `Password` and `PasswordHash` are also the unloggable wrapper types from
  [27](../27-observability-and-audit-logging/spec.md) §4 — their `ToString()` returns `***`.

This makes storing a plain-text password, or double-hashing one, unrepresentable rather than
merely forbidden.

## 5. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `400` | `validation_failed` | Invalid payload |
| `400` | `unknown_role` | `roleId` does not exist |
| `400` | `unknown_field` | Unrecognised property in the body |
| `401` | `not_authenticated` | Not authenticated |
| `403` | `field_not_writable` | A present field the caller may not write |
| `404` | `user_not_found` | Target user does not exist on an update |
| `409` | `email_already_registered` | Email already in use |

## 6. Audit and logging

- `user.profile.update` with actor, target, changed field **names** and status.
- `user.create` with actor, target, assigned role name and status.
- **Never log passwords or password hashes.**

## 7. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 16.1 | `403` for a non-writable field, not a silent drop | Silence hides both client bugs and privilege-escalation attempts; the caller learns the request was refused | Ignoring fields the caller may not write |
| 16.2 | `Password` and `PasswordHash` value types with a single producer | Makes "stored in plain text" and "hashed twice" impossible to express, rather than relying on every author remembering | A convention that services must hash before saving |
| 16.3 | `UpdateUserRequest` does not inherit the create request | Inheritance silently widens the update contract as the create contract grows | DTO inheritance |
| 16.4 | `password` not writable by admins through PATCH | An admin setting a password for someone means that password travels through a channel neither party controls; the reset flow is the correct path | Allowing admin password writes |
| 16.5 | `roleId` accepted on create only | Role *changes* carry the last-admin and self-protection rules in [17](../17-user-role-assignment/spec.md); allowing `roleId` on PATCH would bypass them | Accepting `roleId` on PATCH |
| 16.6 | Admin-created users default to `pending` | A created account should not be live until someone decides it is | Defaulting to `active` |

## 8. Risks and open points

- **Self PATCH has no writable fields at all** under this policy. The endpoint exists for admins
  and for future self-writable profile fields; confirm the frontend does not expect otherwise.
- **Admin-created users with `isVerified: true` skip every verification flow**, which makes
  `POST /users` a trust boundary worth auditing closely.
- **The `pending` default** means an admin creating a user and expecting an immediate login must
  pass `status: active` explicitly. Surface that in the admin UI.

## 9. Acceptance criteria

1. A non-admin cannot change their own `status`, `isVerified` or `password` through PATCH — each
   returns `403` naming the field.
2. An admin can change another user's `email`, `status` and `isVerified`.
3. PATCH accepts `password` from nobody — it is not in the contract, so it returns `400`.
4. A user created through `POST /users` can log in with the submitted password, and the stored
   value is a bcrypt hash.
5. `POST /users` with a `roleId` assigns that role; without one assigns `USER`; with an unknown id
   returns `400`.
6. `POST /users` responses never contain `password`.
7. Updating a non-existent user returns `404`, not `500`.
8. Unit and integration tests cover every row of the table in §2.
