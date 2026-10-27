# Feature: User Profile Update

## 1. Overview
* **Purpose:** Update user profile fields according to access rules and permission scopes.
* **Target Roles:**
  * **Self:** Can modify allowed personal profile fields. *Direct email modification is prohibited.*
  * **Admin:** Can directly modify any profile field for any user, including email addresses.
* **Dependencies:** Access JWT (Cookie), RBAC Module, Email Dispatcher (for verification workflows), Supabase avatar upload endpoint.

* **Avatar behavior:** Avatar files are uploaded and removed only through the dedicated endpoints in [spec 13](../13-avatar-upload/spec.md). The profile PATCH contract does not accept a `photo` URL and updates to unrelated fields must preserve the current avatar.

## 2. Technical Contract

### 2.1 Profile Patch Endpoint (Non-Email Fields)
* **Endpoint:** `PATCH /users/{userId}`
* **Input Parameters:**
  * `userId` (string, path parameter, required)
  * `patch` (JSON object containing permitted fields to update)

* **Validation Rules:**
  1. Validate input types and constraints for all fields in the payload.
  2. If caller is **Self** (`userId` == `req.user.id`):
     * Payloads containing `email` must be rejected with `403 Forbidden` (email changes require the verification endpoint).
  3. If caller is **Admin** (possesses `users.update` permission):
     * Direct modification of all fields (including `email`) is permitted without confirmation.

* **Execution Logic:**
  1. Authenticate JWT and identify caller.
  2. Check `userId` scope against caller identity or `users.update` permission.
  3. Validate payload against role-allowed writable field list (Default-Deny).
  4. Apply mutation in a single database transaction and return updated profile payload.

### 2.2 Email Change Initiation (Self)
* **Endpoint:** `POST /users/{userId}/email-change`
* **Input Parameters:**
  * `userId` (string, path parameter, required)
  * `newEmail` (string, required, valid email format)
  * `method` (enum: `"otp"` | `"magic_link"`, optional)

* **Execution Logic:**
  1. Enforce **Self** access restriction (`userId` must equal `req.user.id`).
  2. Validate `newEmail` format and verify it is not currently occupied by another user (`409 Conflict`).
  3. Generate a pending email change challenge record with TTL (10 minutes).
  4. Dispatch verification code (6-digit OTP, 60s resend limit) or single-use Magic Link to `newEmail`.
  5. Return `200 OK` with `requiresConfirmation: true` and `challengeId`.

### 2.3 Email Change Confirmation (Self)
* **Endpoint:** `POST /users/{userId}/email-change/confirm`
* **Input Parameters:**
  * OTP Strategy: `{ challengeId: string, code: string }`
  * Magic Link Strategy: `{ token: string }`

* **Execution Logic:**
  1. Validate caller identity and active challenge record.
  2. Check challenge expiration (10 min TTL) and verification attempt counters (max 5 attempts).
  3. Update user's `email` field in the database upon successful verification.
  4. Invalidate the challenge token immediately.
  5. Return `200 OK`.

## 3. Error Handling & Edge Cases
* `400 Bad Request`: Invalid payload structure or invalid field formats.
* `401 Unauthorized`: Missing or invalid JWT.
* `403 Forbidden`: IDOR attempt, unpermitted field modification attempt, or direct `email` patch attempt by non-admin.
* `404 Not Found`: Target `userId` or email change `challengeId` not found.
* `409 Conflict`: Proposed `newEmail` is already in use.
* `422 Unprocessable Entity`: Expired OTP code, invalid token, or attempt limit exceeded.
* `429 Too Many Requests`: Rate limit exceeded for update attempts or verification dispatches.

## 4. Audit & Logging
* Log profile update operations: `actorUserId`, `targetUserId`, changed field names (*strictly omit changed values to protect PII*), and status code.
* Log lifecycle events for email change dispatches and confirmations (initiated, dispatched, confirmed, expired).

## 5. Non-Functional & Security Requirements
* **Atomicity:** Profile updates must execute within a single database transaction.
* **Asynchronous Messaging:** Email notifications for address verification must be dispatched via background task queues to avoid blocking HTTP requests.
* **Field Masking:** Strict default-deny whitelist enforcement on patchable fields per role.