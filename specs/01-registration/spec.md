# Feature: User Registration

## 1. Overview
* **Purpose:** Create a user account using email and password.
* **Target Roles:** Guest (unauthenticated user).
* **Configurability:** Admins can independently enable or disable email verification requirement for:
  * User Registration
  * Password Reset
  * Authentication / Login

## 2. User Stories & Scenarios
* **Registration without Verification:** User submits email and password -> account is created -> user can immediately log in.
* **Registration with Verification (Optional):** User submits email and password -> system sends a 6-digit OTP or Magic Link -> user confirms email -> account becomes active.

## 3. Technical Contract

### 3.1 Registration Request
* **Input Parameters:**
  * `email` (string, required, valid email format)
  * `password` (string, required)

* **Validation Rules:**
  * `email` must be valid and non-empty.
  * `password` must adhere to security rules (minimum length, character complexity).
  * Duplicate registration with an existing email must be rejected.

* **Execution Logic:**
  1. Validate input payload.
  2. Check email uniqueness.
  3. **If verification is DISABLED:**
     * Create active user and related entities.
     * Return success response with session/tokens (if configured).
  4. **If verification is ENABLED:**
     * Create user in `PENDING` state or register a "pending registration" record.
     * Trigger email verification dispatch via chosen strategy:
       * **OTP Strategy:** 6-digit code, 10-minute TTL, max 5 attempts, 60-second resend rate limit.
       * **Magic Link Strategy:** Unique single-use token link.

* **Response Variants:**
  * **Variant A (Verification Disabled):** `201 Created` with user payload and optional auth tokens.
  * **Variant B (Verification Enabled):** `202 Accepted` with `verificationRequired: true` flag and `attemptId` / token.

### 3.2 Verification Contract
* Executed only if the administrative `registration_verification_enabled` flag is `true`.
* Requires valid token/OTP payload matching active registration attempt.

## 4. Error Handling & Edge Cases
* `400 Bad Request`: Invalid email format or password complexity policy failure.
* `409 Conflict`: Email already registered (or generic response if user enumeration defense is enabled).
* `422 Unprocessable Entity`: Invalid or expired OTP/Magic Link token.
* `429 Too Many Requests`: Resend cooldown or maximum verification attempt limit exceeded.

## 5. Audit & Logging
* Log registration attempts (success / failure).
* Log verification code/link dispatches.
* Log successful and failed verification confirmations.

## 6. Non-Functional & Security Requirements
* **Rate Limiting:** Protect registration endpoints against spam and brute-force attacks.
* **User Enumeration Prevention:** Ensure error responses do not leak account existence details where required by security policy.