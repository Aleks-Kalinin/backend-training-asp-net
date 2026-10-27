# Feature: User Write Hardening

## 1. Overview

- **Purpose:** Fix privilege and data-handling problems in the user create and update endpoints, and add role selection when an administrator creates a user.
- **Problems in the current implementation:**
  1. **Self-escalation:** `PATCH /users/{userId}` blocks only `email` for non-admin self updates (`users.service.ts`). A user can set their own `status`, `isVerified` and `password`, for example marking an unverified account as verified.
  2. **Plain-text passwords:** Only `POST /auth/signup` hashes passwords. `POST /users` and `PATCH /users/{userId}` store the submitted password unchanged, so those users cannot log in and their passwords are stored in clear text.
  3. **Response leak:** `POST /users` returns the user entity directly instead of passing it through `UserMapper`, which can expose the password hash and internal fields.
  4. **No role on creation:** Administrators cannot choose a role when creating a user; the default `USER` role is always applied.
- **Target Roles:**
  - **Self:** May update only self-writable fields (currently none through PATCH; email uses the dedicated flow, avatar uses the photo endpoints).
  - **Admin:** Callers with `users:create` / `users:update`.
- **Dependencies:** Users module, Auth module (`PasswordHasherPort`), RBAC module, [spec 06](../06-user-profile-update/spec.md), [spec 15](../15-admin-user-access/spec.md), [spec 17](../17-user-role-assignment/spec.md).

## 2. Technical Contract

### 2.1 Writable Field Policy (Default-Deny)

Define writable field lists in the users domain, next to the profile field config:

| Field | Self (PATCH) | Admin (PATCH) | Admin (POST) |
|---|---|---|---|
| `email` | ❌ (use email-change flow) | ✅ | ✅ (required) |
| `status` | ❌ | ✅ | ✅ (optional, default `pending`) |
| `isVerified` | ❌ | ✅ | ✅ (required) |
| `password` | ❌ (future: [spec 20](../20-password-management/spec.md)) | ❌ (future: reset email, spec 20) | ✅ (required) |
| `roleId` | ❌ | ❌ (use spec 17 endpoint) | ✅ (optional, default `USER`) |

- A request containing any field not writable for the caller is rejected with `403 Forbidden` (not silently dropped), and the forbidden field names are logged (not their values).
- "Admin" is decided by permission (`users:update` / `users:create`), not by the `ADMIN` role name.

### 2.2 Update DTO

- `UpdateUserDto` must no longer extend `CreateUserDto`. Define it explicitly with `email`, `status`, `isVerified` (all optional). Remove `password` from the update contract.

### 2.3 Create User

- **Endpoint:** `POST /api/v1/users` (requires `users:create`).
- **Body:** `email`, `password` (min 8), `isVerified`, optional `status`, optional `roleId` (UUID).
- **Execution Logic:**
  1. Normalize the email and return `409 Conflict` if it is taken.
  2. Hash the password with `PasswordHasherPort` before saving.
  3. If `roleId` is given, it must reference an existing role (`400 Bad Request` otherwise). If omitted, assign the default `USER` role.
  4. Save and return the user through `UserMapper` (admin field set, [spec 15](../15-admin-user-access/spec.md)) with status `201`.

### 2.4 Password Hashing Rule

- Hashing happens in the users application layer (or a shared service), so every code path that sets a password uses it. Sign-up must not hash twice.

## 3. Error Handling & Edge Cases

- `400 Bad Request`: Invalid payload, unknown `roleId`.
- `401 Unauthorized`: Not authenticated.
- `403 Forbidden`: Caller tries to write a field not allowed for them.
- `404 Not Found`: Target user does not exist (currently `updateUser` throws a plain `Error`, producing `500`; it must throw `NotFoundException`).
- `409 Conflict`: Email already in use.

## 4. Audit & Logging

- Keep the existing update audit log (actor, target, field names, status). Add the same log for create, including the assigned role name.
- Never log passwords or password hashes.

## 5. Non-Functional & Security Requirements

- Existing users created through `POST /users` before this change may have plain-text passwords. Add a migration or one-off script that detects non-bcrypt values in `users.password` and forces those accounts into a "must reset password" state, or document that they must be recreated. Do not attempt to hash existing values in place without confirming they are not already hashes.

## 6. Acceptance Criteria

1. A non-admin user cannot change their own `status`, `isVerified` or `password` through `PATCH /users/{userId}` (`403`).
2. An admin can change `email`, `status` and `isVerified` of another user.
3. `PATCH` no longer accepts `password` from anyone.
4. A user created through `POST /users` can log in with the submitted password, and the stored value is a bcrypt hash.
5. `POST /users` with `roleId` assigns that role; without it assigns `USER`; with an unknown ID returns `400`.
6. `POST /users` responses never contain `password`.
7. Updating a non-existent user returns `404`, not `500`.
8. Unit and e2e tests cover each row of the writable field table.
