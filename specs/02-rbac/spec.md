# Feature: Role-Based Access Control (RBAC)

## 1. Overview
* **Purpose:** Provide centralized, dynamic access management based on roles and permissions stored in the database without requiring application restarts.
* **Target Roles:** 
  * **Admin:** Manages roles, permissions, and grants.
  * **User:** Subject of access verification, possessing one or multiple assigned roles.
* **Context:** Authorizes requests post-JWT authentication.
* **Dependencies:** Database storage, cache mechanism (e.g., Redis or in-memory), cache invalidation pipeline.

## 2. Core Entities
* **Role:** Grouping of permissions (`id`, `name`, `description`).
* **Permission:** Target resource identifier and allowed actions (`id`, `name`, `actions[]`).
* **Grant:** Mapping entity connecting `roleId` -> `permissionId` -> optional `actions[]`. 
  * *Note:* If `actions` is omitted or empty in a grant, all actions under that permission are allowed.

## 3. Technical Contract

### 3.1 Access Check (Internal Module Contract)
* **Input Parameters:**
  * `userId` (string): Authenticated user ID.
  * `roles` (string[]): Roles assigned to the user.
  * `permission` (string): Target permission key (e.g., `users`).
  * `action` (string): Action attempted (e.g., `create`, `update`, `delete`).

* **Execution Logic:**
  1. Verify user authentication status.
  2. Match user roles against loaded RBAC configuration.
  3. Retrieve all grants associated with the user's roles.
  4. Evaluate permission and action match:
     * Grant matches `permission` AND (`actions` is empty OR contains requested `action`) -> **Allow**.
     * No matching grant found -> **Deny (HTTP 403 Forbidden)**.

### 3.2 Administrative Management API

#### Roles Management
* `GET /admin/rbac/roles` — List all roles.
* `POST /admin/rbac/roles` — Create new role.
* `PUT /admin/rbac/roles/{roleId}` — Update existing role.
* `DELETE /admin/rbac/roles/{roleId}` — Delete role (prevented if active grants exist).
* **Payload:** `{ id: string, name: string, description?: string }`

#### Permissions Management
* `GET /admin/rbac/permissions` — List all permissions.
* `POST /admin/rbac/permissions` — Create new permission.
* `PUT /admin/rbac/permissions/{permissionId}` — Update permission.
* `DELETE /admin/rbac/permissions/{permissionId}` — Delete permission (prevented if active grants exist).
* **Payload:** `{ id: string, name: string, actions: string[] }`

#### Grants Management
* `GET /admin/rbac/grants` — List all grants.
* `POST /admin/rbac/grants` — Create new grant.
* `PUT /admin/rbac/grants/{grantId}` — Update grant.
* `DELETE /admin/rbac/grants/{grantId}` — Delete grant.
* **Payload:** `{ id: string, roleId: string, permissionId: string, actions?: string[] }`

## 4. Cache & Dynamic Updates
* Configuration is loaded from DB into memory/cache on application bootstrap.
* Any create, update, or delete operation on Roles, Permissions, or Grants must trigger:
  1. Database mutation.
  2. Cache invalidation signal.
  3. Dynamic reload of active access configuration across application instances.

## 5. Error Handling & Edge Cases
* `400 Bad Request`: Invalid action array or structural payload error.
* `403 Forbidden`: Non-admin attempt to modify RBAC data, or unauthorized resource access.
* `404 Not Found`: Target role, permission, or grant ID does not exist.
* `409 Conflict`: Unique constraint violation (duplicate role name, permission key, or existing grant mapping) or deletion attempt on entities with active bindings.

## 6. Audit & Logging
* Log administrative operations with `actorUserId`, operation type, target entity, and HTTP status.
* Log cache invalidation and RBAC configuration reload events.