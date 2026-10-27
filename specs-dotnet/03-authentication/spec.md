# 03 — User Authentication (Login)

## 1. Overview

- **Purpose:** Authenticate an existing user with email and password.
- **Target roles:** Registered users.
- **Configurability:** Administrators can enable or disable the email-verification step for login
  independently of registration, via `login_verification_enabled`
  ([26](../26-configuration-and-settings/spec.md) §3).
- **Dependencies:** [01](../01-registration/spec.md) §4 (verification mechanism),
  [04](../04-authorization-session/spec.md) (token and cookie issuance),
  [26](../26-configuration-and-settings/spec.md) (settings).

## 2. Endpoints

| Endpoint | Method | Auth | Rate limit policy |
| --- | --- | --- | --- |
| `/api/v1/auth/login` | POST | anonymous | `auth-login` |
| `/api/v1/auth/login/verify` | POST | anonymous | `verification-confirm` |
| `/api/v1/auth/confirm` | GET | anonymous | `verification-confirm` |

```csharp
public sealed record LoginRequest(string Email, string Password);
public sealed record LoginVerifyRequest(Guid AttemptId, string OtpCode);
```

### 2.1 Responses

| Condition | Status | Body | Cookies |
| --- | --- | --- | --- |
| Verification disabled | `200 OK` | `MessageResponseDto` | both auth cookies set |
| Verification enabled | `202 Accepted` | `VerificationRequiredResponseDto { message, verificationRequired: true, attemptId }` | **none** |

The login response carries **no user payload**. Tokens live in `HttpOnly` cookies, so the client
discovers who it is from `GET /auth/me` ([14](../14-current-user/spec.md)). Returning a profile
here would duplicate the field-masking rules in [05](../05-user-profile-view/spec.md).

## 3. Execution logic

1. Validate payload completeness and email format.
2. Normalise the email and look up the user.
3. **Always** run the bcrypt verification — against the stored hash, or against a fixed dummy hash
   when no user exists. This keeps the timing of the two branches comparable, which is what the
   enumeration defence in §6 actually requires.
4. Blocked or deleted account: `403` `account_blocked`.
5. Read `login_verification_enabled` and `login_verification_strategy`.
6. **Verification disabled:** update `LastLoginAt`, reset the failure counter, issue access and
   refresh cookies, return `200`.
7. **Verification enabled:** do **not** issue tokens. Create a pending login attempt bound to the
   user, dispatch the challenge through `IVerificationService`
   ([01](../01-registration/spec.md) §4), return `202` with the `attemptId`.
8. **Confirmation** (`/auth/login/verify` with an OTP, or `/auth/confirm` with a magic-link token):
   validate against the active attempt, check expiry and attempt limits, mark it verified, issue
   cookies, initialise the session context, update `LastLoginAt`.

The magic-link confirmation endpoint is shared with registration and dispatches on the challenge's
stored purpose. It returns `200` and does not redirect.

## 4. Brute-force protection

Two independent layers.

**Request volume** — the `auth-login` rate-limit policy, partitioned by IP and by email
([31](../31-security-rate-limiting-and-cors/spec.md) §1).

**Account lockout** — a consecutive-failure counter on the user row:

| Field | Purpose |
| --- | --- |
| `FailedLoginCount` | Consecutive failed credential attempts |
| `LockedUntil` | Lockout expiry, null when not locked |

After 10 consecutive failures the account locks for 15 minutes. A successful login resets the
counter. **A locked account still returns `401 invalid_credentials`**, not a distinct status or
code — a lockout message is itself an account-existence oracle.

The counter lives in the database rather than in cache because it must survive a restart and apply
across source IPs.

## 5. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `400` | `validation_failed` | Missing or malformed payload |
| `401` | `invalid_credentials` | Unknown email, wrong password, or locked account — one generic response |
| `403` | `account_blocked` | Account disabled or deleted |
| `403` | `account_not_verified` | Pending account attempting to log in — see §8 |
| `422` | `invalid_otp` / `expired_token` | Invalid or expired OTP or magic-link token |
| `429` | `rate_limited` | Login or resend volume limit exceeded |
| `429` | `verification_attempts_exceeded` | More than 5 verification attempts |

## 6. Non-functional and security requirements

- **Enumeration defence:** generic credential errors must never reveal whether an email exists.
  This requires both an identical response body and comparable timing — hence the dummy-hash
  verification in §3 step 3.
- **Brute-force protection:** strict rate limiting plus the temporary account lockout in §4.
- **No raw credentials in logs**, ever.

## 7. Audit and logging

- `auth.login` with outcome (success / failure).
- `auth.verification.dispatch` with purpose and strategy.
- `auth.verification.confirm` with purpose and outcome.

## 8. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 03.1 | Dummy-hash verification for unknown emails | The enumeration defence is only real if timing matches; an early return is a measurable oracle | Short-circuit return on user-not-found |
| 03.2 | Lockout state on the user row | Must survive restarts and apply across IPs | In-memory counters; relying on the rate limiter alone |
| 03.3 | Lockout returns `401`, not `423` or `429` | A distinct status confirms the account exists | A specific `account_locked` code |
| 03.4 | One shared magic-link confirm endpoint | The token already encodes its purpose; separate endpoints would duplicate validation | A confirm endpoint per flow |
| 03.5 | No redirect from `/auth/confirm` | Redirect targets from configuration or tokens invite open-redirect bugs | Redirecting to a frontend URL |
| 03.6 | Login returns no user payload | The client reads `/auth/me`; one place owns field masking | Returning the profile |
| 03.7 | Lockout implemented directly, not via ASP.NET Core Identity | Identity's lockout is coupled to its user store and schema, which this service does not use | `SignInManager` lockout |
| 03.8 | Pending users receive `403 account_not_verified` and a resent code | The alternative — a generic `401` — leaves a verified-but-forgotten user with no path forward | Treating pending as invalid credentials |

## 9. Risks and open points

- **Decision 03.8 is a visible UX branch** and slightly weakens enumeration defence: it confirms
  that an unverified account exists for that address. The trade is deliberate, since the address
  owner is the only one who receives the resent code, but confirm it with the team.
- **Lockout is a denial-of-service vector.** An attacker who knows an address can keep it locked.
  The 15-minute window bounds it; a CAPTCHA step is the usual next lever if abuse appears.
- **`LastLoginAt` is written on every login**, adding a write to a hot path. Accepted —
  [08](../08-admin-user-list/spec.md) sorts on it.

## 10. Acceptance criteria

1. Valid credentials with verification disabled return `200` and set both cookies.
2. Valid credentials with verification enabled return `202`, set no cookies, and send a code.
3. Unknown email and wrong password return identical bodies, statuses and comparable timings.
4. A blocked account returns `403`.
5. Ten consecutive failures lock the account for 15 minutes, and the locked response is
   indistinguishable from a wrong password.
6. A correct OTP after a `202` issues cookies; a wrong one returns `422`; the sixth returns `429`.
7. Unit tests cover lockout transitions; integration tests cover both branches and the
   enumeration-defence response equality.
