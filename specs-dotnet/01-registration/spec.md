# 01 — User Registration

## 1. Overview

- **Purpose:** Create a user account from an email address and password.
- **Target roles:** Guest (unauthenticated).
- **Configurability:** Administrators can independently enable or disable the email-verification
  requirement for registration, password reset and login. This document covers registration; the
  setting is `registration_verification_enabled`
  ([26](../26-configuration-and-settings/spec.md) §3).
- **Dependencies:** [04](../04-authorization-session/spec.md) (cookie issuance),
  [17](../17-user-role-assignment/spec.md) (default `USER` role),
  [25](../25-background-jobs/spec.md) (email dispatch),
  [26](../26-configuration-and-settings/spec.md) (settings).

### 1.1 Scenarios

- **Without verification:** the user submits email and password, the account is created active, and
  they can log in immediately.
- **With verification:** the user submits email and password, the system sends a 6-digit OTP or a
  magic link, the user confirms, and the account becomes active.

## 2. Endpoints

| Endpoint | Method | Auth | Rate limit policy |
| --- | --- | --- | --- |
| `/api/v1/auth/signup` | POST | anonymous | `auth-register` |
| `/api/v1/auth/signup/verify` | POST | anonymous | `verification-confirm` |
| `/api/v1/auth/confirm` | GET | anonymous | `verification-confirm` |
| `/api/v1/auth/verification/resend` | POST | anonymous | `verification-resend` |

```csharp
public sealed record SignUpRequest(string Email, string Password);
public sealed record VerifyOtpRequest(Guid AttemptId, string Code);
public sealed record ResendRequest(Guid AttemptId);
```

### 2.1 Validation rules

- `email` — required, non-empty, valid email format. Normalised to lowercase before any lookup or
  write; the `citext` column type makes uniqueness case-insensitive
  ([22](../22-persistence-and-migrations/spec.md) §3).
- `password` — required, minimum 8 characters, complexity per `.StrongPassword()`
  ([23](../23-validation-and-partial-updates/spec.md) §2.2).
- Registration with an email that already exists must be rejected — see §5.1 for how, which depends
  on the enumeration-defence setting.

### 2.2 Responses

| Condition | Status | Body | Cookies |
| --- | --- | --- | --- |
| Verification disabled | `201 Created` | `SelfUserResponseDto` ([05](../05-user-profile-view/spec.md) §3) | both auth cookies set |
| Verification enabled | `202 Accepted` | `VerificationRequiredResponseDto { message, verificationRequired: true, attemptId }` | **none** |

## 3. Execution logic

1. Validate the payload (binding and syntactic stages,
   [23](../23-validation-and-partial-updates/spec.md) §2).
2. Normalise the email and check uniqueness.
3. Read `registration_verification_enabled` and `registration_verification_strategy` from
   `ISettingsProvider`.
4. Hash the password with bcrypt, work factor 12, through `IPasswordHasher`.
5. **Verification disabled:** create the user with status `Active`, assign the default `USER` role,
   issue access and refresh cookies, return `201`.
6. **Verification enabled:** create the user with status `Pending`, issue a verification challenge
   (§4), enqueue the dispatch email, return `202`. No cookies are issued to a pending account.
7. Record the audit events in §7.

Verification confirmation (`/auth/signup/verify` for OTP, `/auth/confirm` for a magic link)
consumes the challenge, transitions the user to `Active`, issues cookies and returns `200`.

## 4. Shared verification mechanism

Registration, login ([03](../03-authentication/spec.md)), email change
([06](../06-user-profile-update/spec.md)), self-deletion ([07](../07-user-deletion/spec.md)) and
password reset ([20](../20-password-management/spec.md)) all use the same challenge mechanism. It
is specified here and implemented once.

```csharp
public enum VerificationPurpose
{
    Registration, Login, EmailChange, AccountDeletion, PasswordReset
}

public enum VerificationStrategy { Otp, MagicLink }

public interface IVerificationService
{
    Task<VerificationChallenge> IssueAsync(VerificationPurpose purpose, Guid? userId, string email,
                                           VerificationStrategy strategy, CancellationToken ct);
    Task<VerificationResult> ConsumeAsync(Guid attemptId, string code, CancellationToken ct);
    Task<VerificationChallenge> ResendAsync(Guid attemptId, CancellationToken ct);
}
```

| Parameter | Value |
| --- | --- |
| OTP format | 6 digits, generated with `RandomNumberGenerator.GetInt32(0, 1_000_000)` |
| OTP TTL | 10 minutes |
| Max attempts | 5 per challenge |
| Resend | 1 per 60 seconds, 3 per challenge |
| Magic-link token | 32 random bytes, base64url, single-use |
| Magic-link TTL | 10 minutes, bound to the specific attempt |
| Storage | Codes and tokens stored **hashed** (SHA-256 salted with the attempt id) |
| Comparison | `CryptographicOperations.FixedTimeEquals` |

- `ConsumeAsync` increments the attempt counter atomically and stamps `ConsumedAt` on success.
  Expired, attempt-exceeded, already-consumed and mismatched are distinct result values, mapping to
  `422` and `429` per §6.
- Magic-link URLs have the form `https://<host>/api/v1/auth/confirm?token=...`. The endpoint is
  shared across purposes and dispatches on the stored `VerificationPurpose`. It returns `200` and
  does **not** redirect — redirect targets taken from configuration or a token are an
  open-redirect risk.
- Expired challenges are swept by `VerificationAttemptCleanupJob`
  ([25](../25-background-jobs/spec.md) §3).
- OTP emails must contain the six digits in a stable, parseable form — a line reading
  `Your code: 123456`. The end-to-end suite parses it ([19](../19-test-infrastructure/spec.md) §2).

## 5. Components

| Type | Project | Note |
| --- | --- | --- |
| `AuthController` | `Api/Auth` | Thin: bind, call, map |
| `SignUpRequestValidator` | `Api/Auth/Validators` | Email format, `.StrongPassword()` |
| `RegistrationService` | `Application/Auth` | Owns the flow |
| `IUserRepository` | `Application/Users/Ports` | `FindByEmailAsync`, `AddAsync` |
| `IVerificationService` | `Application/Verification/Ports` | §4 |
| `IPasswordHasher` | `Application/Auth/Ports` | `BCryptPasswordHasher` adapter |
| `User`, `VerificationAttempt` | `Domain` | [22](../22-persistence-and-migrations/spec.md) §3 |

### 5.1 Enumeration defence

Controlled by `registration_enumeration_defense` (default on).

- **On:** a duplicate email returns the same `202` as a fresh registration, and the system emails
  the existing account to say someone attempted to register with their address. No usable challenge
  is created for the caller.
- **Off:** a duplicate email returns `409` `email_already_registered`.

Either way the password hash is computed before branching, so response timing does not distinguish
the cases ([24](../24-error-handling/spec.md) §4.1).

## 6. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `400` | `validation_failed` | Invalid email format or password complexity failure |
| `400` | `unknown_field` | Body contains an unrecognised property |
| `409` | `email_already_registered` | Email in use and enumeration defence is off |
| `422` | `invalid_otp` / `expired_token` | Wrong, expired or already-consumed OTP or magic link |
| `429` | `rate_limited` | Request-volume limit exceeded |
| `429` | `verification_attempts_exceeded` | More than 5 attempts, or resend cooldown violated |

All bodies are `ProblemDetails` ([24](../24-error-handling/spec.md)).

## 7. Audit and logging

Events, per [27](../27-observability-and-audit-logging/spec.md) §3:

- `auth.register` with outcome (success / failure).
- `auth.verification.dispatch` with purpose and strategy.
- `auth.verification.confirm` with purpose and outcome.

Never log the password, its hash, the OTP code or the magic-link token.

## 8. Non-functional and security requirements

- **Rate limiting** protects the registration endpoints against spam and brute force
  ([31](../31-security-rate-limiting-and-cors/spec.md) §1).
- **Enumeration defence** per §5.1.
- **Email dispatch is always a background job**, never inline, so an SMTP stall cannot block the
  HTTP response ([25](../25-background-jobs/spec.md) §3).

## 9. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 01.1 | `BCrypt.Net-Next`, work factor 12 | Widely reviewed, tunable, and the stored format is self-describing, which makes a future algorithm migration detectable | ASP.NET Core `PasswordHasher<T>` (PBKDF2, tied to Identity conventions); Argon2id (stronger, but adds a native dependency for a training-scale service) |
| 01.2 | One `IVerificationService` for all five purposes | The five flows have identical mechanics; one implementation means one place to get TTL, attempt limits and hashing right | A verification flow per feature |
| 01.3 | OTP codes and tokens stored hashed | A readable code in the database is a readable code in a backup | Plain-text storage |
| 01.4 | Pending users are real `users` rows | Keeps email uniqueness enforceable by one index | A separate `pending_registrations` table (uniqueness must then span two tables) |
| 01.5 | Email dispatch always via a job | Non-blocking by construction, retried and observable | Inline SMTP send |
| 01.6 | Enumeration defence is a setting | The right behaviour differs by deployment; both branches must be supported and tested | Hardcoding either |
| 01.7 | `/auth/confirm` returns `200`, never redirects | A configurable redirect target is an open-redirect vector | Redirecting to a frontend URL |

## 10. Risks and open points

- **Abandoned pending registrations hold their email address forever.** Add a sweep that deletes
  `Pending` users with no active challenge after 24 hours, or accept and document it.
- **`201` with auth cookies** is unusual but intended: a registration without verification logs the
  user straight in.
- **The resend endpoint is shared** with login, email change, deletion and password reset. It is
  specified here because registration needs it first.

## 11. Acceptance criteria

1. With verification disabled, a valid sign-up creates an active user, returns `201` with auth
   cookies, and the user can immediately call an authenticated endpoint.
2. With verification enabled, sign-up returns `202` with an `attemptId` and no cookies; the account
   is `Pending` until confirmed.
3. A correct OTP within 10 minutes activates the account and issues cookies.
4. A wrong OTP returns `422`; the sixth attempt returns `429`; an expired OTP returns `422`.
5. Resend is refused within 60 seconds and after 3 resends (`429`).
6. With enumeration defence on, registering an existing email is indistinguishable from a fresh
   registration in status, body shape and timing.
7. Stored passwords are bcrypt hashes; stored OTPs are hashes, not codes.
8. Unit tests cover challenge issue/consume/expiry/attempt-limit; integration tests cover both
   verification branches end to end, reading the OTP from the mail catcher.
