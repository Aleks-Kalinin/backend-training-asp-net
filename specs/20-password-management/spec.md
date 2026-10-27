# Feature: Password Management (Reset & Change)

> **Status: Planned for future implementation.** Not part of the current frontend milestone. The frontend counterpart is `specs-frontend/09-password-management`.

## 1. Overview

- **Purpose:**
  1. Let a user who forgot their password set a new one through an emailed code.
  2. Let a logged-in user change their password by confirming the current one.
  3. Let an administrator send a password reset email to a user (instead of typing a password for them).
- **Configurability:** The existing `password_reset_verification_enabled` setting ([settings module](../../src/modules/settings)) is currently unused. This spec gives it a meaning (see 2.1).
- **Target Roles:** Guests (reset), Self (change), Admin (send reset).
- **Dependencies:** Auth module, Verification service (OTP, 10 min TTL, max 5 attempts), Mail module, Settings module, [spec 16](../16-user-write-hardening/spec.md).

## 2. Technical Contract

### 2.1 Request Password Reset

- **Endpoint:** `POST /api/v1/auth/password-reset`
- **Access:** Public. Throttled (5 requests per 10 s per IP, plus 1 per 60 s per email).
- **Body:** `{ "email": "user@example.com" }`
- **Execution Logic:**
  1. Normalize the email and look up the user.
  2. If an active user exists, create a `PASSWORD_RESET` verification record and email a 6-digit OTP.
  3. Always return `202 Accepted` with `{ message, attemptId }`. For unknown or inactive emails, return a random `attemptId` that will never verify, so the response does not reveal whether an account exists.
- If `password_reset_verification_enabled` is `false`, the endpoint returns `403 Forbidden` (password reset is turned off by the administrator).

### 2.2 Confirm Password Reset

- **Endpoint:** `POST /api/v1/auth/password-reset/confirm`
- **Body:** `{ "attemptId": "<uuid>", "otp": "123456", "newPassword": "..." }`
- **Execution Logic:**
  1. Verify the OTP (`422` invalid/expired, `429` attempts exceeded).
  2. Validate the new password (same rules as sign-up, min 8 characters).
  3. Hash and save it; invalidate the verification record.
  4. Invalidate existing sessions (see section 5).
  5. Return `200 OK` with `{ message }`. The user must log in again; no cookies are set.

### 2.3 Change Password (Self)

- **Endpoint:** `POST /api/v1/users/{userId}/password`
- **Access:** Self only (`@AllowSelf('id')`, no admin override).
- **Body:** `{ "currentPassword": "...", "newPassword": "..." }`
- **Execution Logic:**
  1. Compare `currentPassword` with the stored hash (`403` if wrong; throttle 5 per 10 s).
  2. Reject a `newPassword` equal to the current one (`400`).
  3. Hash and save; invalidate other sessions; issue fresh cookies for the current session.
  4. Email a "your password was changed" notification.

### 2.4 Admin-Triggered Reset

- **Endpoint:** `POST /api/v1/admin/users/{userId}/password-reset`
- **Access:** `users:update`. Not allowed on the caller's own account (`409`; use 2.3).
- **Execution Logic:** Same as 2.1 for the target user, without the enumeration protection (the admin already knows the account). Returns `202`.

### 2.5 Resend Verification Code

- **Endpoint:** `POST /api/v1/auth/verification/resend`
- **Body:** `{ "attemptId": "<uuid>" }`
- Resends the OTP for an active registration, login, password reset, email change or deletion attempt, issuing a new code and resetting the TTL. Limit: one resend per 60 s and 3 per attempt (`429`).
- Included here because the frontend verification screens need it; it can be implemented independently of the rest of this spec.

## 3. Error Handling & Edge Cases

- `400 Bad Request`: Invalid payload or weak password.
- `401 Unauthorized`: Not authenticated (2.3, 2.4).
- `403 Forbidden`: Wrong current password, or password reset disabled by settings.
- `409 Conflict`: Admin triggers a reset on their own account.
- `422 Unprocessable Entity`: Invalid or expired OTP.
- `429 Too Many Requests`: Rate or attempt limits exceeded.

## 4. Audit & Logging

- Log reset requests (masked email, whether a user was found), confirmations, self changes and admin-triggered resets with actor and target IDs. Never log passwords, hashes or OTP codes.

## 5. Non-Functional & Security Requirements

- **Session invalidation:** Refresh tokens are currently stateless ([spec 04](../04-authorization-session/spec.md)). Add a `passwordChangedAt` (or token version) column to `users`, include it in the token payload, and reject access and refresh tokens issued before it. This needs a migration.
- Use constant-time responses for 2.1 regardless of whether the user exists.
- OTP codes for password reset are single-use.

## 6. Acceptance Criteria

1. A user can reset a forgotten password through the emailed code and log in with the new password.
2. The reset request response is identical for existing and non-existing emails.
3. A logged-in user can change their password only with the correct current password.
4. After a reset or change, previously issued refresh tokens no longer work.
5. An admin can send a reset email to another user, but not to themselves.
6. Verification codes can be resent within the stated limits.
7. When the setting is off, the reset endpoints return `403`.
8. Unit and e2e tests cover each endpoint, including enumeration protection and token invalidation.
