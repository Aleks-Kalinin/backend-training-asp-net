# 15 — Admin User Access & Field Visibility

## 1. Overview

- **Purpose:** Define what administrators may see of other users' accounts, and which permission
  gates each administrative view. This document owns the **visibility policy**; the endpoints that
  apply it are specified in [05](../05-user-profile-view/spec.md) (detail) and
  [08](../08-admin-user-list/spec.md) (list).
- **Target roles:**
  - **Admin** — callers holding `users:read` for detail, and `users:list` for the list.
  - **Self and other users** — covered by the self and public field sets below.
- **Dependencies:** [02](../02-rbac/spec.md), [05](../05-user-profile-view/spec.md),
  [08](../08-admin-user-list/spec.md), [17](../17-user-role-assignment/spec.md) (the `role` field).

## 2. Permission requirements

| Surface | Permission | Self access |
| --- | --- | --- |
| `GET /api/v1/admin/users` | `users:list` | **no** |
| `GET /api/v1/users/{userId}` | `users:read` | yes |
| `PATCH /api/v1/users/{userId}` | `users:update` | yes |
| `POST /api/v1/users` | `users:create` | n/a |
| `PUT`/`DELETE /api/v1/users/{userId}/photo` | `users:update` | yes |
| `PUT /api/v1/admin/users/{userId}/role` | `users:update` | **no** |

A caller without the required permission receives `403 Forbidden`. The `users:list` action is
seeded on the `users` permission and granted to `ADMIN`
([17](../17-user-role-assignment/spec.md) §5).

**Every action under an `/admin/` route must carry a permission attribute.** Its absence is
invisible in code review — nothing looks wrong — so this is asserted by an architecture test
([29](../29-testing-strategy/spec.md) §2.4), not left to discipline.

## 3. Field sets

Three sets, selected in this order for any response carrying a user profile:

| # | Condition | Set |
| --- | --- | --- |
| 1 | Requester is the subject (`callerId == targetId`) | **Self** |
| 2 | Requester holds `users:read`, checked through the RBAC cache | **Admin** |
| 3 | Otherwise | **Public** |

```csharp
public static class UserProfileFields
{
    // UserId, Email, Status, IsVerified, CreatedAt, UpdatedAt, Photo, Role
    public static readonly FrozenSet<UserField> Self;

    // UserId, Email, Status, IsVerified, CreatedAt, UpdatedAt, Photo, Role
    public static readonly FrozenSet<UserField> Admin;

    // UserId, Email, Photo
    public static readonly FrozenSet<UserField> Public;
}
```

- `UserField.Role` is exposed as `{ id, name }` — a single role.
- Self includes `role`, so a user can see their own.
- Fields outside the selected set are **omitted** from the response, not nulled.

**The set is decided by permission, never by role name**, so a custom role granted `users:read`
receives the admin view, and a role named `ADMIN` without the permission does not.

The same mapper applies to `GET /admin/users`, `GET /users/{userId}`, `PATCH /users/{userId}`,
`POST /users`, and `PUT`/`DELETE /users/{userId}/photo`.

## 4. Response DTOs

| DTO | Fields | Used by |
| --- | --- | --- |
| `SelfUserResponseDto` | Self set | Self-scoped responses |
| `AdminUserResponseDto` | Admin set | Admin detail, and the item type of `PaginatedUsersResponseDto` |
| `PublicUserResponseDto` | Public set | Everything else |

All three are named schemas in the OpenAPI document, so generated clients are typed
([28](../28-openapi-and-versioning/spec.md) §3).

A unit test asserts each DTO's property names equal its field set exactly — the control that keeps
the policy data and the wire shape from drifting apart
([05](../05-user-profile-view/spec.md) §3.1).

## 5. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `401` | `not_authenticated` | Not authenticated |
| `403` | `permission_denied` | Lacks `users:list` for the list, or lacks both self access and `users:read` for detail |

Field filtering stays default-deny: any new user column is hidden until it is added to a field set
and to a DTO.

## 6. Audit and logging

- Keep the list audit event with actor id, filter keys, status and returned count
  ([08](../08-admin-user-list/spec.md) §5).
- Log `403` attempts on the list endpoint with the actor id. That signal is what surfaces a
  misconfigured or missing permission in production.

## 7. Non-functional and security requirements

- Decide the field set from permissions, not from hardcoded role names, so custom roles behave
  correctly.
- Never expose `password`, verification records or the raw avatar storage path in any field set.

## 8. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 15.1 | Permission-driven field selection | Custom roles must work correctly, and role-name checks silently diverge from the grant model | `if (role == "ADMIN")` |
| 15.2 | Self and Admin kept as separate sets although currently identical | They will diverge; separate sets make that an edit rather than a refactor | Merging them now |
| 15.3 | Default-deny enforced structurally | A new column is invisible until added to both a set and a DTO, and the equality test fails if only one is done | Opt-out masking, where forgetting to exclude a field leaks it |
| 15.4 | Architecture test for `/admin/` permission attributes | Catches the whole class of missing-authorization mistakes, not one instance | Review discipline |

## 9. Risks and open points

- **A missing permission attribute is an authorization bypass that looks like nothing.** The
  architecture test in §2 is the primary control; the `403` audit logging in §6 is the backstop.
- **Admin and public sets both expose `email`.** That is intended, but it means any authenticated
  caller who can reach a profile endpoint learns the subject's address. Confirm this is acceptable.

## 10. Acceptance criteria

1. A user without `users:list` receives `403` from `GET /admin/users`; an admin receives the list.
2. Admin list items and admin detail responses include `status`, `isVerified`, `createdAt`,
   `updatedAt`, `photo` and `role`.
3. Self responses include `status` and `role`; public responses contain only `userId`, `email` and
   `photo`.
4. A custom role granted `users:read` receives the admin field set; a role without it does not.
5. The OpenAPI document contains `AdminUserResponseDto` and types the paginated list item.
6. Unit tests cover the three mapper branches; integration tests cover list access for admin, user
   and unauthenticated callers.
7. An architecture test fails if any `/admin/` action lacks a permission attribute.
