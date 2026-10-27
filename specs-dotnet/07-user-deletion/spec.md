# 07 — User Account Deletion & Anonymization

## 1. Overview

- **Purpose:** Delete or anonymise a user's personal data on request, under privacy rules and
  strict access control.
- **Target roles:**
  - **Self** — a user requesting deletion of their own account. **Email verification is
    mandatory.**
  - **Admin** — a caller with `users:delete`, deleting any account. Admin deletion bypasses email
    verification.
- **Dependencies:** [04](../04-authorization-session/spec.md), [02](../02-rbac/spec.md),
  [01](../01-registration/spec.md) §4 (verification), [25](../25-background-jobs/spec.md) (job
  pipeline), [13](../13-avatar-upload/spec.md) (avatar cleanup),
  [17](../17-user-role-assignment/spec.md) §3 (admin self-protection).

## 2. Endpoints

| Endpoint | Method | Auth | Rate limit policy |
| --- | --- | --- | --- |
| `/api/v1/users/{userId}` | DELETE | `[AllowSelf("userId")] [RequirePermission("users","delete")]` | `deletion` |
| `/api/v1/users/{userId}/deletion-status` | GET | self or `users:delete` | `profile-read` |

Body (optional): `{ reason?: string, challengeId?: Guid, code?: string }`. `reason` is logged for
support analytics.

### 2.1 Self deletion is two-step

1. `DELETE` with no verification payload — creates a challenge, emails an OTP, and returns `200`
   with `DeletionChallengeResponseDto { requiresConfirmation: true, challengeId, message }`.
2. `DELETE` with `{ challengeId, code }` — proceeds with the pipeline.

**An unconfirmed self-deletion request must never proceed.** Admin deletion runs on the first call.

### 2.2 Responses

| Condition | Status | Body |
| --- | --- | --- |
| Self deletion, first step | `200` | `DeletionChallengeResponseDto` |
| Deletion accepted | `202` | `{ jobId, status: "pending" }` |

Deletion status: `GET /users/{userId}/deletion-status` returns
`{ status: "pending" | "in_progress" | "done" | "failed" }`.

## 3. Execution pipeline

Access rights are evaluated first: self (with a verified challenge) or `users:delete`. The
protection rules in §4 are then applied inside a serializable transaction. On success the request
returns `202` and the work runs in `UserDeletionJob`
([25](../25-background-jobs/spec.md) §3), which writes progress to a `BackgroundJobRecord` row —
the row the status endpoint reads.

| Step | Action | Idempotent because |
| --- | --- | --- |
| 1 | Set status `Deleting`; a second request now returns `409` | The status check is the guard |
| 2 | Invalidate the user-status cache, so access tokens stop working within 30 seconds ([04](../04-authorization-session/spec.md) §3.3), and block future authentication | Idempotent write |
| 3 | Scrub PII: `email` becomes `deleted+{id}@invalid`, name fields cleared, `PasswordHash` cleared | Re-running overwrites identically |
| 4 | Delete the avatar object from storage via the retryable cleanup job | Deleting a missing object succeeds |
| 5 | Apply the retention policy to transformation history and stored files for that user | Scoped by user id, re-runnable |
| 6 | Set status `Deleted`, stamp `DeletedAt`, mark the job `done` | Terminal |

**The user row is anonymised, not removed.** History rows and audit records reference the user id,
and a hard delete would either cascade them away or break referential integrity.

Re-running deletion on an already-deleted user returns `409` with a deterministic body, never an
unhandled exception.

## 4. Protection rules

Returned as `409` with a distinct code:

| Rule | Code |
| --- | --- |
| A caller with the `ADMIN` role cannot delete their own account, through either the admin path or the self-deletion flow | `admin_self_delete_blocked` |
| The last remaining active `ADMIN` cannot be deleted | `last_admin_protected` |

Implemented by the shared `AdminProtectionPolicy` ([17](../17-user-role-assignment/spec.md) §3).

## 5. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `401` | `not_authenticated` | Missing or invalid JWT |
| `403` | `permission_denied` | IDOR attempt |
| `403` | `verification_required` | Self deletion without a verified challenge |
| `404` | `user_not_found` | Target does not exist |
| `409` | `already_deleted` | Account already in a `deleting` or `deleted` state |
| `409` | per §4 | Protection rule |
| `429` | `rate_limited` | Deletion request limit exceeded |

## 6. Audit and logging

- `user.delete` with actor id, target id, mode (`self` / `admin`), `jobId` and final status.
- **Never log** deleted or scrubbed PII values, or confirmation secrets.

## 7. Open decision: ending the session

Step 2 requires that deletion immediately stop the account from being used. Access tokens die
within 30 seconds via the status cache, but **refresh tokens are stateless**
([04](../04-authorization-session/spec.md) §2.1): with no server-side state and no lookup on
refresh, a deleted user's client keeps minting new token pairs for up to 30 days.

The options and the proposal are set out in [04](../04-authorization-session/spec.md) §7.
**Proposed: check the cached user status during `POST /auth/refresh` only**, which kills the
session within 15 minutes at the cost of one cached read on a low-frequency endpoint. This
contradicts the stateless rule as written and needs sign-off before this document can be fully
implemented. The same decision unblocks [20](../20-password-management/spec.md).

## 8. Non-functional and security requirements

- **IDOR protection:** a non-admin cannot trigger deletion of another user id.
- **Idempotency:** re-executing deletion on an already-processed record returns a deterministic
  response.
- **Observability:** every pipeline step is retryable and its outcome visible through the job
  record.

## 9. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 07.1 | Asynchronous only, always `202` | The pipeline touches object storage, which is slow and failure-prone. One path means one set of tests and a status endpoint that is always meaningful | Synchronous `204` (ties the response to storage availability); supporting both (two code paths, two test matrices) |
| 07.2 | Anonymise rather than hard delete | Preserves the referential integrity that history and audit records depend on | `DELETE FROM users` with cascades |
| 07.3 | A dedicated `BackgroundJobRecord` for status | The status contract must not expose the job runner's internal schema | Querying the job runner's monitoring API |
| 07.4 | Two-step self deletion on one endpoint | Verification is a prerequisite of the delete call, not a separate resource | A separate `/delete/request` endpoint |
| 07.5 | Status-based idempotency guard | Makes repeat calls deterministic and job re-runs safe | Relying on job-runner deduplication |

## 10. Risks and open points

- **§7 blocks completion.** Without it, deletion does not actually end the session.
- **`reason` is untrusted user text.** Length-cap it, and never interpolate it into logs or emails.
- **A partially failed pipeline** can leave a user stuck in `Deleting`. The job retries and the
  record carries the failure code, but an operational runbook is needed for the terminal case.
- **Anonymisation keeps history rows keyed by user id.** Whether that satisfies the applicable
  erasure obligation is a legal question; raise it before shipping.

## 11. Acceptance criteria

1. Self deletion without a verification payload returns a challenge and sends a code; the account
   is untouched.
2. Self deletion with a valid code returns `202` and completes the pipeline.
3. Admin deletion runs without verification.
4. A non-admin deleting another user receives `403`.
5. An admin deleting their own account, or the last active admin, receives `409`.
6. A second deletion request returns `409` and no unhandled exception.
7. After completion, PII is scrubbed, the avatar object is gone, and the status endpoint reports
   `done`.
8. No log line contains scrubbed PII or the confirmation code.
