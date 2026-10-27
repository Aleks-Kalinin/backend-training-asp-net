# Feature: User Profile View

## 1. Overview
* **Purpose:** Retrieve and display user profile data according to strict role-based access rules and field-level permissions.
* **Target Roles:**
  * **Self:** Authenticated user requesting their own profile.
  * **Support / Admin:** Administrative roles requesting another user's profile (requires explicit RBAC permission).
* **Dependencies:** Access JWT (Cookie), RBAC Module, Supabase avatar URL resolution.

## 2. Technical Contract

### 2.1 Profile Retrieval Endpoint
* **Endpoint:** `GET /users/{userId}`
* **Input Parameters:**
  * `userId` (string, path parameter, required)
  * Authentication: `access_token` (JWT via HTTP-Only cookie)

* **Access Control & Authorization:**
  1. **Self Access:** Granted if path `userId` matches `req.user.id`.
  2. **Admin/Support Access:** Granted if `req.user` possesses the `users.read` permission.
  3. **Unauthorized Request:** Reject with `403 Forbidden` if neither condition is met.

* **Response Field Masking (Default-Deny Policy):**
  * The API must apply server-side field filtering based on the requester's role.
  * **Self Payload:** Full profile data (`id`, `email`, `photo`, personal settings, and permitted profile fields). `photo` is the public Supabase URL resolved from the avatar storage path stored in the `photo` column, or `null` when no avatar is set; it is never a user-supplied URL.
  * **Support/Admin Payload:** Return only fields explicitly allowed by the RBAC configuration policy. Unpermitted fields must be omitted from the output object.

* **Execution Logic:**
  1. Validate access JWT and extract caller context.
  2. Verify target user existence in the database.
  3. Validate access rights (Self vs. RBAC `users.read`).
  4. Fetch profile entity and filter response properties against role permissions.
  5. Return `200 OK` with filtered payload.

## 3. Error Handling & Edge Cases
* `401 Unauthorized`: Missing, invalid, or expired access token cookie.
* `403 Forbidden`: IDOR prevention triggered (caller requesting another user's profile without `users.read` permission).
* `404 Not Found`: Target `userId` does not exist in the system.
* `429 Too Many Requests`: Rate limit exceeded for profile read queries.

## 4. Audit & Logging
* Log profile view requests with `viewerUserId`, `targetUserId`, and HTTP status code.
* Do not log sensitive personal profile field values in application logs.

## 5. Non-Functional & Security Requirements
* **IDOR Prevention:** Enforce explicit checks matching `req.user.id` against `userId` parameter before executing database queries.
* **Default-Deny Exposure:** Ensure new profile fields added to database models are masked by default until explicitly added to role view policies.