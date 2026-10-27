# Feature: User Role Assignment & Admin Self-Protection

## 1. Overview

- **Purpose:**
  1. Let administrators change a user's role. Grants connect roles to permissions ([spec 02](../02-rbac/spec.md)), but nothing assigns a role to a user, so nobody can be promoted to admin.
  2. Prevent administrators from locking themselves or the system out (self-deletion, self-blocking, self-demotion, removing the last admin).
  3. Seed the system roles and permissions, and provide a way to create the first administrator.
- **Role model:** Each user has **exactly one** role. The `user_roles` join table stays many-to-many in the schema, but the application enforces one row per user.
- **Target Roles:**
  - **Admin:** Callers with `users:update` (role change) and `users:delete` (deletion).
- **Dependencies:** Users module, RBAC module, [spec 07](../07-user-deletion/spec.md), [spec 14](../14-current-user/spec.md), [spec 16](../16-user-write-hardening/spec.md).

## 2. Technical Contract

### 2.1 Change User Role

- **Endpoint:** `PUT /api/v1/admin/users/{userId}/role`
- **Access:** `@RequirePermission('users', 'update')`. No `@AllowSelf`.
- **Body:** `{ "roleId": "<uuid>" }`
- **Execution Logic:**
  1. Verify the target user exists (`404`) and the role exists (`400`).
  2. Apply the protection rules in 2.2.
  3. Replace the user's role in one transaction.
  4. Return the updated user through `UserMapper` (admin field set, [spec 15](../15-admin-user-access/spec.md)).
- Existing JWTs keep the old role list for up to 15 minutes. Because `RbacGuard` reads roles from the JWT, the new role takes effect on the user's next token refresh. Document this in Swagger. `/auth/me` reflects it immediately ([spec 14](../14-current-user/spec.md)).

### 2.2 Protection Rules

Return `409 Conflict` with a clear message when any rule is broken:

| Action | Rule |
|---|---|
| Change role | An admin cannot change **their own** role. |
| Change role | The role of the **last remaining active user with the `ADMIN` role** cannot be changed. |
| Delete user (`DELETE /users/{userId}`) | A caller with the `ADMIN` role cannot delete **their own** account, through either the admin path or the self-deletion OTP flow. |
| Delete user | The last remaining active `ADMIN` cannot be deleted. |
| Update user (`PATCH /users/{userId}`) | An admin cannot set **their own** `status` to anything other than `active`. |
| Update user | The last remaining active `ADMIN` cannot be blocked. |

- Rules are enforced on the server even though the UI disables these actions.
- "Last admin" counts users whose role is `ADMIN` and whose status is `active`.

### 2.3 Roles Service Guards

- Deleting or renaming the `ADMIN` and `USER` system roles returns `409 Conflict` (they are referenced by code: `SystemRole` enum and the default role lookup).
- Deleting a role that is still assigned to users returns `409 Conflict` (in addition to the existing "has grants" check).

### 2.4 Seed Data

Add a migration that inserts, if missing:

- Roles: `ADMIN`, `USER`.
- Permissions and actions:

| Permission | Actions |
|---|---|
| `users` | `create`, `read`, `update`, `delete`, `list` |
| `files` | `create`, `read` |
| `history` | `read` |
| `settings` | `read`, `update` |
| `roles` | `create`, `read`, `update`, `delete` |
| `permissions` | `create`, `read`, `update`, `delete` |
| `grants` | `create`, `read`, `update`, `delete` |

- Grants: `ADMIN` gets all actions of every permission. `USER` gets `files:create` and `files:read`.
- The migration must be idempotent and must not overwrite grants an administrator has changed.

### 2.5 First Administrator Bootstrap

- Add an npm script `npm run admin:bootstrap -- --email <email>` that promotes an existing **active** user to `ADMIN`.
- The script refuses to run if an active `ADMIN` already exists, unless `--force` is passed.
- No HTTP endpoint and no environment-variable credentials are used for bootstrapping.

## 3. Error Handling & Edge Cases

- `400 Bad Request`: Invalid or unknown `roleId`.
- `401 Unauthorized` / `403 Forbidden`: Standard auth and permission failures.
- `404 Not Found`: Target user does not exist.
- `409 Conflict`: Any protection rule in 2.2 or 2.3 is broken.

## 4. Audit & Logging

- Log role changes with `actorUserId`, `targetUserId`, previous role name, new role name and status.
- Log rejected protection-rule attempts at `warn` level with the rule name.
- Log bootstrap script runs (email is masked in logs).

## 5. Non-Functional & Security Requirements

- Role changes run in a single transaction. The "last admin" check and the change happen inside the same transaction (use a row lock or `SERIALIZABLE` isolation) so two concurrent requests cannot remove both of the last two admins.
- Emit the existing RBAC change event after role changes where relevant, so caches stay consistent.

## 6. Acceptance Criteria

1. An admin can change another user's role; the response and `/auth/me` show the new role.
2. An admin cannot change their own role, delete themselves, or block themselves (`409`).
3. The last active admin cannot be demoted, deleted or blocked (`409`).
4. `ADMIN` and `USER` roles cannot be deleted or renamed; a role assigned to users cannot be deleted.
5. A fresh database after migrations contains the seed roles, permissions and grants; running migrations again changes nothing.
6. `admin:bootstrap` promotes a user only when no active admin exists (or with `--force`).
7. Unit tests cover each protection rule; e2e tests cover role change, self-protection and last-admin cases, including two concurrent demotions.
