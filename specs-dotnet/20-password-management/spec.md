# 20 — Password Management (Reset & Change)

> **Status: planned.** Not part of the current milestone. Documented now because the session
> invalidation in §5 adds a column and a token claim that are cheap to design in from the start and
> expensive to retrofit into already-issued tokens.

## 1. Overview

**Purpose:**

1. Let a user who has forgotten their password set a new one through an emailed code.
2. Let a logged-in user change their password by confirming the current one.
3. Let an administrator send a password reset email to a user, instead of typing a password for
   them.

**Configurability:** `password_reset_verification_enabled`
([26](../26-configuration-and-settings/spec.md) §3) controls whether the reset flow is available
at all — see §2.1.

**Target roles:** guests (reset), self (change), admin (send reset).

**Dependencies:** [01](../01-registration/spec.md) §4 (verification mechanism),
[03](../03-authentication/spec.md), [04](../04-authorization-session/spec.md),
[16](../16-user-write-hardening/spec.md) §4 (single hashing path),
[26](../26-configuration-and-settings/spec.md).

## 2. Endpoints

| Endpoint | Method | Auth | Rate limit policy |
| --- | --- | --- | --- |
| `/api/v1/auth/password-reset` | POST | anonymous | `password-reset` |
| `/api/v1/auth/password-reset/confirm` | POST | anonymous | `verification-confirm` |
| `/api/v1/users/{userId}/password` | POST | `[AllowSelf("userId")]`, **no admin override** | `profile-write` |
| `/api/v1/admin/users/{userId}/password-reset` | POST | `[RequirePermission("users","update")]` | `profile-write` |
| `/api/v1/auth/verification/resend` | POST | anonymous | `verification-resend` |

### 2.1 Request password reset

**Body:** `{ "email": "user@example.com" }`.
**Throttle:** 5 requests per 10 seconds per IP, plus 1 per 60 seconds per email.

1. If `password_reset_verification_enabled` is `false`, return `403` `feature_disabled` — password
   reset is turned off by the administrator.
2. Normalise the email and look up the user.
3. If an active user exists, create a `PasswordReset` verification challenge and email a 6-digit
   OTP.
4. **Always return `202 Accepted`** with `{ message, attemptId }`. For unknown or inactive emails,
   return a **random `attemptId` that will never verify**, so the response does not reveal whether
   an account exists.

The random `attemptId` is not persisted — a row would both leak the enumeration through timing and
accumulate garbage.

### 2.2 Confirm password reset

**Body:** `{ "attemptId": "<uuid>", "otp": "123456", "newPassword": "..." }`.

1. Verify the OTP — `422` invalid or expired, `429` attempts exceeded.
2. Validate the new password against the sign-up rules, minimum 8 characters.
3. Hash and save it through `IPasswordHasher`; invalidate the verification record.
4. Invalidate existing sessions (§5).
5. Return `200` with `{ message }`. **No cookies are set** — the user must log in again.

### 2.3 Change password (self)

**Access:** self only, with no admin override.
**Body:** `{ "currentPassword": "...", "newPassword": "..." }`.
**Throttle:** 5 per 10 seconds.

1. Compare `currentPassword` with the stored hash — `403` `invalid_current_password` if wrong.
2. Reject a `newPassword` equal to the current one — `400`.
3. Hash and save; invalidate other sessions; **issue fresh cookies for the current session**.
4. Email a "your password was changed" notification.

### 2.4 Admin-triggered reset

**Access:** `users:update`. **Not permitted on the caller's own account** — `409`; use §2.3.

Behaves as §2.1 for the target user, but **without** the enumeration protection: the administrator
already knows the account exists. Returns `202`.

### 2.5 Resend verification code

**Body:** `{ "attemptId": "<uuid>" }`.

Resends the OTP for an active registration, login, password reset, email change or deletion
attempt, issuing a new code and resetting the TTL. Limit: one resend per 60 seconds and three per
attempt, else `429`.

This endpoint is specified here because it belongs with verification, but it is implemented with
[01](../01-registration/spec.md) — the registration and login verification screens need it first,
and it can ship independently of the rest of this document.

## 3. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `400` | `validation_failed` | Invalid payload or weak password |
| `400` | `password_unchanged` | New password equals the current one |
| `401` | `not_authenticated` | Not authenticated, for §2.3 and §2.4 |
| `403` | `invalid_current_password` | Wrong current password |
| `403` | `feature_disabled` | Password reset disabled by settings |
| `409` | `admin_self_reset_blocked` | Admin triggers a reset on their own account |
| `422` | `invalid_otp` / `expired_token` | Invalid or expired OTP |
| `429` | `rate_limited` / `verification_attempts_exceeded` | Rate or attempt limits exceeded |

## 4. Audit and logging

- Reset requests with a **masked** email and whether a user was found.
- Confirmations, self changes and admin-triggered resets with actor and target ids.
- **Never log passwords, hashes or OTP codes.**

## 5. Session invalidation

Refresh tokens are stateless ([04](../04-authorization-session/spec.md) §2.1), so a password change
does not by itself stop previously issued tokens.

**Schema and claim:**

- Add `users.password_changed_at` (`timestamptz`).
- Include it as a `pwd_at` claim in both access and refresh tokens.
- Reject any token whose `pwd_at` predates the stored value.

**Where the check runs** determines the cost:

| Option | Effect | Cost |
| --- | --- | --- |
| On refresh only (**proposed**) | Other sessions die within 15 minutes, when their access token expires and refresh is refused | One cached user read on `POST /auth/refresh` |
| On every request | Immediate | A lookup per request |

**Proposed: refresh only**, which is the same mechanism and the same deviation proposed for
deletion in [07](../07-user-deletion/spec.md) §7 — one decision unblocks both. It satisfies §7.4
exactly: previously issued refresh tokens stop working, while access tokens survive at most their
remaining 15 minutes.

## 6. Non-functional and security requirements

- Responses to §2.1 are constant-time and identical regardless of whether the user exists.
- OTP codes for password reset are **single-use**.
- Password hashing goes through the single hashing path in
  [16](../16-user-write-hardening/spec.md) §4; no code path stores a password any other way.

## 7. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 20.1 | Design the schema and claim now, implement the flows later | `password_changed_at` and `pwd_at` are cheap to include in the first migration and in token issuance, and expensive to add to tokens already in circulation | Deferring the column too |
| 20.2 | `pwd_at` checked on refresh only | Satisfies the acceptance criterion at the cost of one cached read on a low-frequency endpoint, and reuses the deletion decision | A per-request check (expensive); no check (criterion unmet) |
| 20.3 | Always `202`, with a random non-verifying `attemptId` | Makes the existing and non-existing cases indistinguishable without issuing a usable challenge | Returning `404` for unknown emails |
| 20.4 | Self change issues fresh cookies | Logging a user out of the tab where they just changed their password is hostile; every *other* session still dies | Clearing all cookies |
| 20.5 | Admin reset cannot target self | Prevents an admin bypassing the current-password check on their own account | Allowing it |
| 20.6 | Resend ships with [01](../01-registration/spec.md) | The verification screens need it before any other part of this document is built | Shipping it with the rest of this document |
| 20.7 | `403` when the feature setting is off | Gives `password_reset_verification_enabled` a defined meaning rather than leaving it inert | Ignoring the setting |

## 8. Risks and open points

- **This document depends on an unresolved decision.** §5 and [07](../07-user-deletion/spec.md) §7
  propose the same deviation from the stateless-refresh rule. If it is rejected, neither deletion
  nor password reset can actually end a session, and acceptance criterion §9.4 becomes
  unimplementable. Resolve it once, before either is built.
- **Admin-triggered reset emails go to the address on file**, which an admin may have just changed
  ([06](../06-user-profile-update/spec.md) §10). Worth a confirmation step in the admin UI.
- **Change notifications go to the address on file.** If an account is already compromised and the
  address changed, the notification reaches the attacker — an argument for also notifying the
  previous address on email change.

## 9. Acceptance criteria

1. A user can reset a forgotten password through the emailed code and log in with the new one.
2. The reset request response is identical, in status, body shape and timing, for existing and
   non-existing emails.
3. A logged-in user can change their password only with the correct current password.
4. After a reset or change, previously issued refresh tokens no longer work.
5. An admin can send a reset email to another user, but not to themselves.
6. Verification codes can be resent within the stated limits, and are refused beyond them.
7. When `password_reset_verification_enabled` is off, the reset endpoints return `403`.
8. Unit and integration tests cover each endpoint, including enumeration protection and token
   invalidation.
