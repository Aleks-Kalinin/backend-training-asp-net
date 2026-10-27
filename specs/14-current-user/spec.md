# Feature: Current User Session Endpoint

## 1. Overview

- **Purpose:** Let a client discover who is logged in, which role they have, and what they are allowed to do. Auth tokens live in `HttpOnly` cookies ([spec 04](../04-authorization-session/spec.md)), so browser code cannot read the JWT payload, and the login endpoints return only a message. Without this endpoint the frontend cannot learn its own `userId` (required by every `/users/{userId}` call) or decide which admin screens to show.
- **Target Roles:** Any authenticated user.
- **Dependencies:** Auth module (`AuthGuard`), Users module, RBAC module (`RbacCacheService`), avatar URL resolution ([spec 13](../13-avatar-upload/spec.md)).

## 2. Technical Contract

### 2.1 Current User Endpoint

- **Endpoint:** `GET /api/v1/auth/me`
- **Access:** Authenticated (valid `access_token` cookie). No RBAC permission required.
- **Execution Logic:**
  1. Authenticate via `AuthGuard` (user must exist and be `active`).
  2. Load the user from the database. Do **not** trust the role list in the JWT, which may be up to 15 minutes old after a role change.
  3. Resolve the user's effective permissions from the RBAC cache: the union of every grant of the user's role, expanded to `permission:action` strings. A grant with no `actions` (or an empty list) expands to every action of that permission, matching `RbacGuard` behaviour.
  4. Return the response below.
- **Response `200 OK`** (`CurrentUserResponseDto`):

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

- `role` is a single object, because users have exactly one role ([spec 17](../17-user-role-assignment/spec.md)). It is `null` only if data is inconsistent; clients must treat `null` as "no permissions".
- `permissions` is sorted and contains no duplicates.
- `photo` is the resolved public URL or `null`, never the stored path.

### 2.2 Swagger

- Document the endpoint under the `Authentication` tag with `@ApiCookieAuth('access_token')` and the `CurrentUserResponseDto` schema, so the frontend's generated client gets a typed response.

## 3. Error Handling & Edge Cases

- `401 Unauthorized`: Missing, invalid or expired access token, or the user is no longer `active`. The client is expected to try `POST /auth/refresh` once and retry.
- The response must reflect role and grant changes immediately (it reads the database and the RBAC cache, not the JWT).

## 4. Audit & Logging

- Do not log on every call (the frontend calls this on each app load). Log only failures at `warn` level with the user ID.

## 5. Non-Functional & Security Requirements

- Never return `password`, verification data, or internal storage paths.
- The `permissions` list is a UI hint only. Every endpoint must keep enforcing permissions on the server.
- Keep the endpoint cheap: one user lookup plus an in-memory cache lookup.

## 6. Acceptance Criteria

1. An authenticated active user receives their `userId`, `email`, `status`, `photo`, `role` and expanded `permissions`.
2. After an admin changes a user's role or a role's grants, the next `/auth/me` call returns the new role and permissions without logging in again.
3. A request without a valid access token returns `401`.
4. A blocked or pending user receives `401`.
5. The response never contains `password` or the raw `photo` storage path.
6. Swagger shows the endpoint with a typed response schema.
7. Unit tests cover permission expansion (including "all actions" grants); e2e tests cover the success, `401` and role-change cases.
