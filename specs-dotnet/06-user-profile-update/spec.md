# 06 — User Profile Update

## 1. Overview

- **Purpose:** Update user profile fields under access rules and permission scopes.
- **Target roles:**
  - **Self** — may modify allowed personal fields. **Direct email modification is prohibited**;
    email changes go through the verification flow in §4.
  - **Admin** — a caller with `users:update` may modify any profile field of any user, including
    the email address, without confirmation.
- **Dependencies:** [04](../04-authorization-session/spec.md), [02](../02-rbac/spec.md),
  [01](../01-registration/spec.md) §4 (verification mechanism),
  [23](../23-validation-and-partial-updates/spec.md) (writable-field policy),
  [17](../17-user-role-assignment/spec.md) §3 (admin self-protection rules).

**Avatar behaviour:** avatars are uploaded and removed only through the dedicated endpoints in
[13](../13-avatar-upload/spec.md). This endpoint does not accept a `photo` value, and updating
unrelated fields must preserve the current avatar.

## 2. Profile patch endpoint

| Endpoint | Method | Auth | Rate limit policy |
| --- | --- | --- | --- |
| `/api/v1/users/{userId}` | PATCH | `[AllowSelf("userId")] [RequirePermission("users","update")]` | `profile-write` |

```csharp
public sealed record UpdateUserRequest
{
    public Optional<string> Email { get; init; }
    public Optional<UserStatus> Status { get; init; }
    public Optional<bool> IsVerified { get; init; }
}
```

There is no `password` field and no `photo` field. `UpdateUserRequest` is defined independently and
does **not** derive from the creation request — inheritance is how `password` leaks into an update
contract ([16](../16-user-write-hardening/spec.md) §4).

### 2.1 Writable-field policy

Presence is detected with `Optional<T>`, so a field sent as `null` is distinguishable from a field
not sent ([23](../23-validation-and-partial-updates/spec.md) §3).

| Field | Self | Admin (`users:update`) |
| --- | --- | --- |
| `email` | `403` — use the flow in §4 | allowed |
| `status` | `403` | allowed, subject to §5 |
| `isVerified` | `403` | allowed |
| `password` | not in the contract | not in the contract |
| `roleId` | not in the contract | not in the contract — use [17](../17-user-role-assignment/spec.md) §2 |
| anything else | `400` `unknown_field` | `400` `unknown_field` |

A field present but not writable for the caller is **rejected with `403`**, never silently
dropped. The offending field *names* appear in the problem extensions and in a warn-level log;
values never do.

"Admin" is decided by the `users:update` permission, not by the role name.

## 3. Execution logic (PATCH)

1. Bind with `UnmappedMemberHandling = Disallow`; an unrecognised property is `400`.
2. Validate present fields only (types, formats, constraints).
3. Resolve the caller's scope (self versus `users:update`) and evaluate the writable-field policy;
   a violation is `403`.
4. Apply the admin self-protection rules in §5; a violation is `409`.
5. Apply the mutation in a **single database transaction** and return the updated profile through
   the mapper in [05](../05-user-profile-view/spec.md) §3.
6. A missing target user returns `404` `user_not_found`.

## 4. Email change (self)

### 4.1 Initiation

| Endpoint | Method | Access | Rate limit |
| --- | --- | --- | --- |
| `/api/v1/users/{userId}/email-change` | POST | **self only** | `profile-write` |

Body: `{ newEmail: string, method?: "otp" | "magic_link" }`.

1. Enforce self access — `userId` must equal the caller's id. This endpoint has no admin path;
   an administrator changes an email directly through PATCH.
2. Validate the format and confirm the address is not in use (`409`).
3. Create a pending email-change challenge with a 10-minute TTL
   ([01](../01-registration/spec.md) §4).
4. Dispatch a 6-digit OTP or a single-use magic link **to the new address**.
5. Return `200` with
   `EmailChangeChallengeResponseDto { requiresConfirmation: true, challengeId, message }`.

A pending change does **not** reserve the address; a race is resolved at confirmation by the unique
index, returning `409`.

### 4.2 Confirmation

| Endpoint | Method | Access |
| --- | --- | --- |
| `/api/v1/users/{userId}/email-change/confirm` | POST | **self only** |

Body: `{ challengeId, code }` for OTP, or `{ token }` for a magic link.

1. Validate the caller's identity and the active challenge.
2. Check expiry (10 minutes) and the attempt counter (max 5).
3. Update the user's email.
4. Invalidate the challenge immediately.
5. Return `200`.

Steps 3 and 4 run in one transaction.

## 5. Admin self-protection

Enforced here as well as on the deletion and role-change paths, inside a serializable transaction
([17](../17-user-role-assignment/spec.md) §3):

| Rule | Status | Code |
| --- | --- | --- |
| An admin may not set **their own** `status` to anything other than `active` | `409` | `admin_self_block_blocked` |
| The last remaining active `ADMIN` may not be blocked | `409` | `last_admin_protected` |

## 6. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `400` | `validation_failed` / `unknown_field` | Invalid payload structure or field format |
| `401` | `not_authenticated` | Missing or invalid JWT |
| `403` | `permission_denied` | IDOR attempt |
| `403` | `field_not_writable` | A field present that the caller may not write, including `email` for self |
| `404` | `user_not_found` / `challenge_not_found` | Unknown target or challenge |
| `409` | `email_already_registered` | Proposed `newEmail` already in use |
| `409` | per §5 | Admin self-protection rule |
| `422` | `invalid_otp` / `expired_token` | Expired code, invalid token, or attempt limit reached |
| `429` | `rate_limited` | Update or verification-dispatch limit exceeded |

## 7. Audit and logging

- `user.profile.update` with actor id, target id, changed field **names** and status code.
  **Strictly omit changed values** — they are PII.
- Email-change lifecycle events: initiated, dispatched, confirmed, expired.

## 8. Non-functional and security requirements

- **Atomicity:** profile updates execute in a single transaction.
- **Asynchronous messaging:** verification emails dispatch through the background job queue so
  they never block the HTTP request ([25](../25-background-jobs/spec.md)).
- **Field masking:** strict default-deny whitelist per role, per §2.1.

## 9. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 06.1 | `Optional<T>` for every patchable field | The reject-don't-drop rule requires presence detection | Nullable-means-absent (cannot express explicit null); JSON Patch (changes the public contract) |
| 06.2 | Email change is self-only, with no admin path | Admins already have direct PATCH; a second admin route would duplicate the rule | Admin access to the challenge endpoints |
| 06.3 | A pending change does not reserve the address | A reservation needs its own expiry and uniqueness story; the unique index already resolves the race correctly | Soft-reserving in `users` |
| 06.4 | Verification reuses the shared mechanism | Same TTL, attempt and resend semantics as every other flow | A bespoke email-change challenge |
| 06.5 | Self-protection rules evaluated on this path too | Self-blocking is reachable through PATCH, so the rule must live here as well as on the role endpoint | Enforcing only on the role-change endpoint |
| 06.6 | Missing target returns `404` | A generic exception here produces a `500`, which hides a routine client error | Propagating an unhandled exception |

## 10. Risks and open points

- **Self PATCH currently has no writable fields at all.** Email, status, `isVerified` and password
  are each forbidden for self. The endpoint exists for admins and for future self-writable profile
  fields; confirm the frontend does not expect otherwise.
- **The old address is not notified** on an email change. Not required here, but most products do
  it, and it is the main defence against a silent account takeover. Recommended.
- **Admin email changes bypass verification entirely**, so an admin can set an address the owner
  does not control. Intended, but worth surfacing in the admin UI.

## 11. Acceptance criteria

1. A self caller sending `email`, `status` or `isVerified` receives `403` naming the field.
2. An admin can change another user's `email`, `status` and `isVerified`.
3. An unrecognised property returns `400`.
4. A self email change requires confirmation; the address changes only after a valid code.
5. Requesting a change to an address already in use returns `409`.
6. An expired or over-attempted confirmation returns `422` or `429`.
7. Updating unrelated fields leaves `photo` unchanged.
8. Updating a non-existent user returns `404`.
9. An admin blocking themselves, or blocking the last active admin, returns `409`.
10. Audit logs contain changed field names and no values.
