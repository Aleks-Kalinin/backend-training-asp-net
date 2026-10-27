# 02 — Role-Based Access Control

## 1. Overview

- **Purpose:** Centralised, dynamic access management based on roles and permissions stored in the
  database, changeable without an application restart.
- **Target roles:**
  - **Admin** — manages roles, permissions and grants.
  - **User** — subject of access verification, holding exactly one role.
- **Context:** Authorizes requests after JWT authentication
  ([04](../04-authorization-session/spec.md)).
- **Dependencies:** PostgreSQL, a cache with a cross-instance invalidation path, and the seed data
  in [17](../17-user-role-assignment/spec.md) §5.

## 2. Core entities

```csharp
public sealed class Role       { Guid Id; string Name; string? Description; }
public sealed class Permission { Guid Id; string Name; string[] Actions; }
public sealed class Grant      { Guid Id; Guid RoleId; Guid PermissionId; string[]? Actions; }
public sealed class UserRole   { Guid UserId; Guid RoleId; }   // unique index on UserId
```

- **Role** — a named grouping of permissions.
- **Permission** — a target resource identifier plus the actions it supports, for example
  `users` with `create`, `read`, `update`, `delete`, `list`.
- **Grant** — maps a role to a permission, optionally narrowed to specific actions.
  **If `Actions` is null or empty, every action of that permission is allowed.**
- **UserRole** — each user has exactly one role. The table is many-to-many in shape, and the unique
  index on `UserId` makes "exactly one" a database guarantee rather than an application hope
  ([22](../22-persistence-and-migrations/spec.md) §3.1).

The empty-actions rule is implemented in exactly one place, `RbacSnapshot.IsAllowed`, and reused by
[14](../14-current-user/spec.md) so the authorization decision and the advertised permission list
can never disagree.

## 3. Access check

**Inputs:** the authenticated user's id and role, the target permission key (for example `users`),
and the attempted action (for example `create`).

**Logic:**

1. Confirm the caller is authenticated.
2. Resolve the caller's role.
3. Retrieve all grants associated with that role from the cached snapshot.
4. A grant matching the permission, whose actions are empty **or** contain the requested action,
   allows the request.
5. No matching grant denies it — `403 Forbidden`, code `permission_denied`.

### 3.1 Authorization pipeline

Expressed as `[RequirePermission("users", "read")]` on a controller action. Resolution:

1. `RequirePermissionAttribute : IAuthorizeData` produces the policy name `perm:users:read`.
2. `RbacPolicyProvider : IAuthorizationPolicyProvider` parses that name on first use and builds a
   policy containing a `PermissionRequirement(permission, action)`. Policies are cached by name.
3. `PermissionAuthorizationHandler` reads the caller's role from the JWT claims, obtains the
   snapshot from `IRbacCache`, and applies §3.
4. No match produces `403`.

A **dynamic** policy provider is required because permissions are database rows that
administrators can create at runtime; registering policies at startup cannot enumerate them.

### 3.2 Self access

Several endpoints are "self **or** permission" — profile read and update, avatar upload and
removal. These carry both attributes:

```csharp
[AllowSelf("userId")]
[RequirePermission("users", "read")]
```

`AllowSelf` names the route parameter compared against the caller's `sub` claim. Authorization
succeeds if **either** condition holds, and the controller is told which one matched, because the
response field set depends on it ([05](../05-user-profile-view/spec.md) §3).

Endpoints that must never allow self access — role change
([17](../17-user-role-assignment/spec.md)) and the admin user list
([08](../08-admin-user-list/spec.md)) — simply omit `AllowSelf`.

## 4. Cache and dynamic updates

```csharp
public interface IRbacCache
{
    ValueTask<RbacSnapshot> GetAsync(CancellationToken ct);
    Task InvalidateAsync(CancellationToken ct);
}
```

- `RbacSnapshot` is immutable: roles, permissions, grants, and a precomputed
  `IReadOnlyDictionary<Guid, FrozenSet<string>>` of role id to `permission:action` strings.
  Precomputation turns every authorization check into a set lookup.
- Backed by `HybridCache` (in-memory L1, optional Redis L2), loaded in one query set on a miss.
- Loaded at application bootstrap by an `IHostedService`, so the first request is not slow.

### 4.1 Invalidation

Every create, update or delete of a role, permission or grant must, in this order:

1. Write to the database.
2. Invalidate the cache.
3. Signal every other instance to reload.

```csharp
public interface IRbacChangeNotifier
{
    Task PublishAsync(CancellationToken ct);
    event Func<Task> Changed;
}
```

Two adapters: `InProcessRbacChangeNotifier` (default, single instance) and
`RedisRbacChangeNotifier` (publishes to a channel; every instance invalidates on receipt). The
active notifier is logged once at startup, because running multiple instances with the in-process
notifier is a silent authorization-staleness bug
([26](../26-configuration-and-settings/spec.md) §5).

## 5. Administrative API

All under `/api/v1/admin/rbac/`, each action guarded by the matching permission.

### Roles

| Endpoint | Method | Permission |
| --- | --- | --- |
| `/admin/rbac/roles` | GET | `roles:read` |
| `/admin/rbac/roles` | POST | `roles:create` |
| `/admin/rbac/roles/{roleId}` | PUT | `roles:update` |
| `/admin/rbac/roles/{roleId}` | DELETE | `roles:delete` |

Payload: `{ id, name, description? }`.

### Permissions

| Endpoint | Method | Permission |
| --- | --- | --- |
| `/admin/rbac/permissions` | GET | `permissions:read` |
| `/admin/rbac/permissions` | POST | `permissions:create` |
| `/admin/rbac/permissions/{permissionId}` | PUT | `permissions:update` |
| `/admin/rbac/permissions/{permissionId}` | DELETE | `permissions:delete` |

Payload: `{ id, name, actions: string[] }`.

### Grants

| Endpoint | Method | Permission |
| --- | --- | --- |
| `/admin/rbac/grants` | GET | `grants:read` |
| `/admin/rbac/grants` | POST | `grants:create` |
| `/admin/rbac/grants/{grantId}` | PUT | `grants:update` |
| `/admin/rbac/grants/{grantId}` | DELETE | `grants:delete` |

Payload: `{ id, roleId, permissionId, actions?: string[] }`.

### 5.1 Deletion guards

| Rule | Status | Code |
| --- | --- | --- |
| Delete a role or permission that still has grants | `409` | `rbac_entity_in_use` |
| Delete a role still assigned to users | `409` | `role_in_use` |
| Delete or rename the `ADMIN` or `USER` system roles | `409` | `system_role_protected` |
| Duplicate role name, permission key or grant mapping | `409` | `duplicate_rbac_entity` |

The system-role guards exist because `ADMIN` and `USER` are referenced by the `SystemRole` enum
and the default-role lookup ([17](../17-user-role-assignment/spec.md) §5.1).

## 6. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `400` | `validation_failed` | Invalid actions array or structural payload error |
| `403` | `permission_denied` | Non-admin attempting an RBAC mutation, or any denied resource access |
| `404` | `role_not_found` / `permission_not_found` / `grant_not_found` | Unknown target id |
| `409` | per §5.1 | Unique-constraint violation or deletion of a bound entity |

## 7. Audit and logging

- `rbac.admin.mutate` with actor, entity type, entity id, action and resulting status.
- `rbac.cache.reload` with the trigger, on every invalidation and reload.

## 8. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 02.1 | Dynamic `IAuthorizationPolicyProvider` | Permissions are runtime database rows; startup policy registration cannot cover them | Static policies (cannot enumerate); a custom action filter (bypasses framework authorization semantics and `[AllowAnonymous]`) |
| 02.2 | Role read from the JWT, permissions from the cache | A grant change takes effect immediately; a role change takes effect at the user's next token refresh, which is the documented behaviour in [17](../17-user-role-assignment/spec.md) §4 | A database role read on every request |
| 02.3 | Precomputed `permission:action` sets per role | Authorization runs on every request; a set lookup is the right cost | Evaluating grants per request |
| 02.4 | `HybridCache` plus a pluggable notifier | Satisfies cross-instance reload without forcing Redis on a single-instance deployment | In-memory only (wrong when scaled out); always-Redis (infrastructure for nothing) |
| 02.5 | `[AllowSelf]` composed with `[RequirePermission]` | Half the user endpoints are self-or-admin; encoding it in attributes keeps the rule out of controller bodies, where it gets forgotten | Manual `if (callerId == targetId)` per action |
| 02.6 | One `IsAllowed` shared with [14](../14-current-user/spec.md) | The advertised permission list must match the enforced one, including empty-actions expansion | Separate expansion logic per consumer |

## 9. Risks and open points

- **Roles are stale for up to 15 minutes** after a change, because the role comes from the JWT.
  This is accepted and documented ([17](../17-user-role-assignment/spec.md) §4); `/auth/me` is the
  immediate-read escape hatch.
- **Multi-instance correctness depends on configuration.** Consider failing startup when more than
  one instance is declared and the in-process notifier is selected.
- **Empty-actions grants widen automatically.** Adding an action to a permission silently extends
  every all-actions grant on it. That is the specified semantics; the admin UI should make it
  visible.

## 10. Acceptance criteria

1. A caller whose role has a matching grant is allowed; one without receives `403`.
2. A grant with an empty `actions` list permits every action of its permission.
3. Creating, updating or deleting a role, permission or grant takes effect on the next request
   without a restart.
4. Deleting a role with grants, a role assigned to users, or a system role returns `409`.
5. A non-admin receives `403` from every `/admin/rbac/` endpoint.
6. Unit tests cover the grant-matching rules including empty actions; integration tests cover
   mutation followed by an immediately changed authorization outcome.
