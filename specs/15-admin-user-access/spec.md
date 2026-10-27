# Feature: Admin User List Access & Admin User View

## 1. Overview

- **Purpose:** Close an access gap in the admin user list and give administrators the user fields they need to manage accounts.
- **Problems in the current implementation:**
  1. `GET /admin/users` has no `@RequirePermission`, so `RbacGuard` allows **any authenticated user** to list every user. [Spec 08](../08-admin-user-list/spec.md) requires admin-only access.
  2. `UserMapper.toProfileResponseDto` distinguishes only "self" and "everyone else". Administrators therefore receive only `userId`, `email` and `photo` for other users, both in the list and in `GET /users/{userId}`. The admin UI cannot show `status`, `isVerified`, timestamps or role.
- **Target Roles:**
  - **Admin:** Callers with the `users:read` permission (list also requires the list permission, see 2.1).
  - **Self / other users:** Unchanged behaviour.
- **Dependencies:** RBAC module, Users module, [spec 05](../05-user-profile-view/spec.md), [spec 08](../08-admin-user-list/spec.md), [spec 17](../17-user-role-assignment/spec.md) (role field).

## 2. Technical Contract

### 2.1 Admin User List Permission

- **Endpoint:** `GET /api/v1/admin/users` (contract otherwise unchanged).
- Add `@RequirePermission('users', 'list')`. Seed the `list` action on the `users` permission and grant it to `ADMIN` ([spec 17](../17-user-role-assignment/spec.md), section 2.4).
  - Spec 08 names the permission `users.list.admin`. This spec uses the repository's existing `permission` + `action` model (`users` / `list`) instead of introducing a dotted permission name.
- Callers without the permission receive `403 Forbidden`.

### 2.2 Admin Field View

- Add a third field set to `src/modules/users/domain/user-profile-fields.config.ts`:

```ts
export const ADMIN_PROFILE_FIELDS = [
  UserField.USER_ID,
  UserField.EMAIL,
  UserField.STATUS,
  UserField.IS_VERIFIED,
  UserField.CREATED_AT,
  UserField.UPDATED_AT,
  UserField.PHOTO,
  UserField.ROLE,
] as const;
```

- Add `UserField.ROLE = 'role'`, exposed as `{ id, name }` (single role, see spec 17).
- Mapper selection order:
  1. **Self** → `SELF_PROFILE_FIELDS` (add `ROLE` to this set as well, so a user can see their own role).
  2. **Requester has `users:read`** (checked via `RbacCacheService`, not by role name) → `ADMIN_PROFILE_FIELDS`.
  3. Otherwise → `PUBLIC_PROFILE_FIELDS`.
- Apply the same mapper to: `GET /admin/users`, `GET /users/{userId}`, `PATCH /users/{userId}`, `POST /users`, `PUT` / `DELETE /users/{userId}/photo`.

### 2.3 Response DTOs

- Add `AdminUserResponseDto` (all admin fields) and use it as the item type of `PaginatedUsersResponseDto` in Swagger.
- Update `UserResponseDto` to document the optional `status` and `role` fields returned to self, so generated clients are typed correctly.

## 3. Error Handling & Edge Cases

- `401 Unauthorized`: Not authenticated.
- `403 Forbidden`: Caller lacks `users:list` (list) or lacks both self access and `users:read` (detail).
- Field filtering stays default-deny: any new user column is hidden until it is added to a field set.

## 4. Audit & Logging

- Keep the existing list audit log (`actorUserId`, query keys, status, length). Log `403` attempts on the list endpoint with the actor ID.

## 5. Non-Functional & Security Requirements

- Decide the field set from permissions, not from hardcoded role names, so custom roles behave correctly.
- Never expose `password`, verification records or the raw avatar storage path in any field set.

## 6. Acceptance Criteria

1. A user without `users:list` receives `403` from `GET /admin/users`; an admin receives the list.
2. Admin list items and admin detail responses include `status`, `isVerified`, `createdAt`, `updatedAt`, `photo` and `role`.
3. Self responses include `status` and `role`; public responses still contain only `userId`, `email`, `photo`.
4. A custom role granted `users:read` receives the admin field set; a role without it does not.
5. Swagger documents `AdminUserResponseDto` and the paginated list item type.
6. Unit tests cover the three mapper branches; e2e tests cover list access for admin, user and unauthenticated callers.
