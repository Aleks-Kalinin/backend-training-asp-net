# Feature: User Authentication (Login)

## 1. Overview
* **Purpose:** Authenticate an existing user via email and password.
* **Target Roles:** Registered users.
* **Configurability:** Admins can independently enable or disable email verification triggers for login/authentication attempts.

## 2. Technical Contract

### 2.1 Login Request
* **Endpoint:** `POST /auth/login`
* **Input Parameters:**
  * `email` (string, required, valid email format)
  * `password` (string, required)

* **Validation & Execution Logic:**
  1. Validate payload completeness and email format.
  2. Authenticate credentials against stored hashed credentials.
  3. **If login verification is DISABLED:**
     * Issue access and refresh JWT cookies immediately.
     * Return `200 OK` with user details.
  4. **If login verification is ENABLED:**
     * Do NOT issue JWT tokens yet.
     * Create a `pending_login_attempt` record.
     * Dispatch verification payload via configured strategy:
       * **OTP Strategy:** 6-digit code, 10-minute TTL, max 5 failed attempts, 60-second resend rate limit.
       * **Magic Link Strategy:** Single-use URL (`https://<host>/auth/confirm?token=...`), 10-minute TTL, bound to the specific login attempt.
     * Return `202 Accepted` with `attemptId` and `verificationRequired: true`.

### 2.2 Login Verification Confirmation
* **Endpoint:** `POST /auth/login/verify` (OTP) or `GET /auth/confirm` (Magic Link)
* **Input Parameters:** `attemptId` / `token`, `otpCode` (if OTP).
* **Execution Logic:**
  1. Validate token/code against active `pending_login_attempt`.
  2. Verify expiration status and attempt limits.
  3. On success: Mark attempt as verified, issue auth cookies, and initialize session context.

## 3. Error Handling & Edge Cases
* `401 Unauthorized`: Invalid email or password (return generic credential error to prevent account enumeration).
* `403 Forbidden`: Account disabled or blocked.
* `422 Unprocessable Entity`: Invalid or expired OTP code/Magic Link token.
* `429 Too Many Requests`: Rate limit exceeded for login attempts or verification code resends.

## 4. Audit & Logging
* Log login attempts (success / failure).
* Log dispatch events for OTP and Magic Links.
* Log successful and failed verification steps.

## 5. Non-Functional & Security Requirements
* **Brute-Force Protection:** Implement strict rate-limiting and temporary IP/account lockout policies on consecutive failed credentials.
* **Enumeration Defense:** Generic error messages must not leak whether an email exists in the system.