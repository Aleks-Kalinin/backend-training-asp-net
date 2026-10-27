# 17 — User Role Assignment & Admin Self-Protection

## 1. Overview

**Purpose:**

1. Let administrators change a user's role. Grants connect roles to permissions
   ([02](../02-rbac/spec.md)), but without role assignment nobody can be promoted to admin.
2. Prevent administrators from locking themselves or the system out — self-deletion,
   self-blocking, self-demotion, or removing the last admin.
3. Seed the system roles, permissions and grants, and provide a way to create the first
   administrator.

**Role model:** each user has **exactly one** role. The `user_roles` table is many-to-many in
shape, with a unique index on `user_id` enforcing one row per user
([22](../22-persistence-and-migrations/spec.md) §3.1).

**Target roles:** callers with `users:update` (role change) and `users:delete` (deletion).

**Dependencies:** [02](../02-rbac/spec.md), [06](../06-user-profile-update/spec.md),
[07](../07-user-deletion/spec.md), [14](../14-current-user/spec.md),
[16](../16-user-write-hardening/spec.md).

## 2. Change user role

| Endpoint | Method | Auth | Rate limit policy |
| --- | --- | --- | --- |
| `/api/v1/admin/users/{userId}/role` | PUT | `[RequirePermission("users","update")]`, **no** `AllowSelf` | `profile-write` |

**Body:** `{ "roleId": "<uuid>" }`.

**Execution:**

1. Verify the target user exists (`404`) and the role exists (`400`).
2. Apply the protection rules in §3.
3. Replace the user's role in **one transaction**.
4. Return the updated user through the mapper, admin field set
   ([15](../15-admin-user-access/spec.md) §3).

**Response:** `200` with `AdminUserResponseDto`.

## 3. Protection rules

Each returns `409 Conflict` with a clear message and a distinct code:

| Action | Rule | Code |
| --- | --- | --- |
| Change role | An admin cannot change **their own** role | `admin_self_role_change_blocked` |
| Change role | The role of the **last remaining active user with the `ADMIN` role** cannot be changed | `last_admin_protected` |
| Delete user | A caller with the `ADMIN` role cannot delete **their own** account, through either the admin path or the self-deletion flow | `admin_self_delete_blocked` |
| Delete user | The last remaining active `ADMIN` cannot be deleted | `last_admin_protected` |
| Update user | An admin cannot set **their own** `status` to anything other than `active` | `admin_self_block_blocked` |
| Update user | The last remaining active `ADMIN` cannot be blocked | `last_admin_protected` |

"Last admin" counts users whose role is `ADMIN` **and** whose status is `active`.

Rules are enforced on the server even though the UI disables these actions.

They live in one domain service, `AdminProtectionPolicy`, called from three places: role change
(§2), deletion ([07](../07-user-deletion/spec.md) §4) and profile update
([06](../06-user-profile-update/spec.md) §5).

### 3.1 Concurrency

The last-admin check and the write must happen in the **same transaction**, or two concurrent
requests can each observe two admins and each remove one.

Implementation: an `IsolationLevel.Serializable` transaction wrapping both the count and the write,
with a retry on PostgreSQL error `40001`
([22](../22-persistence-and-migrations/spec.md) §6). One request succeeds; the other either
retries and then fails the rule with `409`, or succeeds harmlessly. The invariant asserted by tests
is "exactly one succeeds and an active admin still exists", not which one wins.

## 4. Role-change timing

Existing JWTs keep the old role for up to 15 minutes. Because the authorization handler reads the
role from the JWT ([02](../02-rbac/spec.md) §3), the new role takes effect on the user's next token
refresh. **Document this in the OpenAPI description.** `GET /auth/me` reflects it immediately
([14](../14-current-user/spec.md)).

After a role change, publish the RBAC change notification so caches stay consistent
([02](../02-rbac/spec.md) §4.1).

## 5. Seed data

An idempotent migration ([22](../22-persistence-and-migrations/spec.md) §5.1) inserts, if missing:

**Roles:** `ADMIN`, `USER`.

**Permissions and actions:**

| Permission | Actions |
| --- | --- |
| `users` | `create`, `read`, `update`, `delete`, `list` |
| `files` | `create`, `read` |
| `history` | `read` |
| `settings` | `read`, `update` |
| `roles` | `create`, `read`, `update`, `delete` |
| `permissions` | `create`, `read`, `update`, `delete` |
| `grants` | `create`, `read`, `update`, `delete` |

**Grants:** `ADMIN` receives all actions of every permission. `USER` receives `files:create` and
`files:read`.

The migration must be idempotent and **must not overwrite grants an administrator has changed** —
`ON CONFLICT DO NOTHING`, never `DO UPDATE`. Re-running migrations on a seeded database changes
zero rows.

### 5.1 System role guards

| Rule | Status | Code |
| --- | --- | --- |
| Deleting or renaming the `ADMIN` or `USER` system roles | `409` | `system_role_protected` |
| Deleting a role still assigned to users | `409` | `role_in_use` |
| Deleting a role that still has grants | `409` | `rbac_entity_in_use` |

The system roles are referenced by the `SystemRole` enum and the default-role lookup, so renaming
them would break code.

## 6. First administrator bootstrap

```
dotnet run --project src/FileConverter.Cli -- admin bootstrap --email <email> [--force]
```

- Promotes an existing **active** user to `ADMIN`.
- Refuses to run if an active `ADMIN` already exists, unless `--force` is passed.
- **No HTTP endpoint and no environment-variable credentials** are used for bootstrapping.
- Shares the application's DI container, so it goes through the same role lookup, transaction and
  protection policy as the API ([30](../30-build-ci-and-docker/spec.md) §2).
- Logs the run with the email **masked**.

## 7. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `400` | `unknown_role` | Invalid or unknown `roleId` |
| `401` | `not_authenticated` | Not authenticated |
| `403` | `permission_denied` | Caller lacks `users:update` |
| `404` | `user_not_found` | Target user does not exist |
| `409` | per §3 and §5.1 | A protection rule or system-role guard |

## 8. Audit and logging

- `user.role.change` with actor id, target id, previous role name, new role name and status.
- `user.protection.rejected` at `warn` level with the rule name, for every rejected attempt.
- Bootstrap script runs, with the email masked.

## 9. Non-functional and security requirements

- Role changes run in a single transaction, and the last-admin check happens inside it, using a
  serializable isolation level or a row lock.
- The RBAC change event is emitted after role changes so caches stay consistent.

## 10. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 17.1 | Unique index on `user_roles.user_id` | Makes "exactly one role" a database guarantee; an application check alone is racy | Application-only enforcement |
| 17.2 | Serializable transaction with a `40001` retry | The simplest correct answer for a count-based invariant on a low-frequency operation | Optimistic concurrency on a version column, which does not cover an invariant spanning rows; advisory locks (viable, more machinery) |
| 17.3 | `AdminProtectionPolicy` shared by three call sites | The rules span role change, deletion and update; one implementation means one test suite and no forgotten path | Duplicating checks per endpoint |
| 17.4 | Seed as idempotent SQL in a migration | Available wherever migrations run, including end-to-end environments, and idempotent by construction | A runtime seeder, which races across instances |
| 17.5 | CLI bootstrap sharing application DI | Avoids reimplementing hashing, role lookup and guards in a script | A raw SQL script |
| 17.6 | Role read from the JWT, with documented staleness | Keeps authorization off the database on every request; `/auth/me` covers the immediacy need | A role read per request |

## 11. Risks and open points

- **The 15-minute staleness window** means a demoted admin keeps admin rights until their token
  refreshes. If that is unacceptable, the lever is the refresh-time check discussed in
  [04](../04-authorization-session/spec.md) §7 — at the cost of a lookup per refresh.
- **`--force` is a loaded footgun**: it mints a second admin from shell access. Shell access
  already implies database access, so it adds little risk, but the run must be logged and visible.
- **A user with no role** is permitted by the unique index, which enforces at most one row, not
  exactly one. `/auth/me` returns `role: null` and no permissions; consider a consistency check.

## 12. Acceptance criteria

1. An admin can change another user's role; the response and `/auth/me` show the new role.
2. An admin cannot change their own role, delete themselves, or block themselves (`409`).
3. The last active admin cannot be demoted, deleted or blocked (`409`).
4. `ADMIN` and `USER` cannot be deleted or renamed; a role assigned to users cannot be deleted.
5. A fresh database after migrations contains the seed roles, permissions and grants; running
   migrations again changes nothing, including grants an administrator has edited.
6. `admin bootstrap` promotes a user only when no active admin exists, or with `--force`.
7. Unit tests cover each protection rule; integration tests cover role change, self-protection and
   last-admin cases, **including two concurrent demotions**.
