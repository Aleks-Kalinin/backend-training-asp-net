# Feature: User Account Deletion & Anonymization

## 1. Overview
* **Purpose:** Delete or anonymize user personal data (PII) upon request, adhering to privacy rules and strict access controls.
* **Target Roles:**
  * **Self:** User requesting deletion of their own account.
  * **Admin:** Administrative user deleting any specified user account.
* **Dependencies:** Access JWT (Cookie), RBAC Module, Email Dispatcher (for verification), Background Job Queue (if processed asynchronously).

## 2. Technical Contract

### 2.1 Deletion Request Endpoint
* **Endpoint:** `DELETE /users/{userId}` (or `POST /users/{userId}/delete`)
* **Input Parameters:**
  * `userId` (string, path parameter, required)
  * `reason` (string, body parameter, optional — logged for audit/support analytics)
  * Verification Payload (required for **Self** deletion: OTP code or Magic Link token)

* **Access Control & Authorization Rules:**
  1. **Self Deletion:** Allowed only if path `userId` equals `req.user.id`.
     * **Mandatory Prerequisite:** Self-deletion **must** require email verification (OTP or Magic Link). Unconfirmed self-deletion requests must be rejected.
  2. **Admin Deletion:** Allowed if caller possesses the `users.delete` permission. Admin deletion bypasses email verification requirements.

* **Execution Logic:**
  1. Authenticate caller and evaluate access rights (`Self` vs. `Admin`).
  2. Verify email confirmation token/code for Self-deletion requests.
  3. Immediately revoke active auth sessions and refresh token validity.
  4. Block future authentication attempts for target account.
  5. Execute deletion/anonymization pipeline:
     * Scrub or mask all PII profile fields (email, name, personal data).
     * Delete user-uploaded assets (including Supabase-hosted avatars) from storage.
     * Process or cascade domain-specific related entities according to data retention policies.
  6. Pipeline can be executed **synchronously** or delegated **asynchronously** to a job queue.

* **Response Variants:**
  * `200 OK` / `204 No Content`: Synchronous deletion completed successfully.
  * `202 Accepted`: Asynchronous deletion enqueued -> returns `{ jobId: string, status: "pending" }`.

### 2.2 Deletion Status Endpoint (Asynchronous Handling)
* **Endpoint:** `GET /users/{userId}/deletion-status` (or `GET /jobs/{jobId}`)
* **Response Payload:** `{ status: "pending" | "in_progress" | "done" | "failed" }`

## 3. Error Handling & Edge Cases
* `401 Unauthorized`: Missing or invalid JWT.
* `403 Forbidden`: IDOR attempt or unconfirmed self-deletion attempt.
* `404 Not Found`: Target user ID does not exist.
* `409 Conflict`: Target user account is already in a `deleting` or `deleted` state.
* `429 Too Many Requests`: Rate limit exceeded for deletion requests.

## 4. Audit & Logging
* Log deletion events: `actorUserId`, `targetUserId`, execution type (`self` vs. `admin`), `jobId` (if applicable), and final status.
* **Forbidden:** Never log deleted/scrubbed PII values or confirmation secrets in application logs.

## 5. Non-Functional & Security Requirements
* **IDOR Protection:** Strict enforcement preventing non-admin users from triggering account deletion on other user IDs.
* **Idempotency:** Re-executing deletion on an already deleted/anonymized record must return a deterministic response without throwing unhandled internal exceptions.