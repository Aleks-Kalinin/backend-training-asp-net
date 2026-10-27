# 25 — Background Jobs & Scheduling

## 1. Overview

- **Purpose:** One job runner, one enqueue port, and a job inventory with retry and observability
  rules.
- **Driver:** Hangfire (`Hangfire.Core`, `Hangfire.AspNetCore`, `Hangfire.PostgreSql`).

Several features require work that is *retryable and observable*:
[13](../13-avatar-upload/spec.md) (avatar object cleanup that must survive failure and be
retried), [07](../07-user-deletion/spec.md) (a `jobId` and a status endpoint), and
[12](../12-transformation-storage/spec.md) (a pruning worker synchronised with history retention).
A bare `BackgroundService` would mean hand-building retry, scheduling, persistence and a
dashboard.

## 2. Port

`FileConverter.Application/Common/Jobs/IJobEnqueuer.cs`:

```csharp
public interface IJobEnqueuer
{
    string Enqueue<TJob>(Expression<Action<TJob>> call);
    string Schedule<TJob>(Expression<Action<TJob>> call, TimeSpan delay);
}
```

Hangfire types never appear outside `Infrastructure`. Unit tests substitute the port; integration
tests either run the in-memory storage or execute the job class directly.

## 3. Job inventory

| Job | Trigger | Retries | Owner document |
| ---  | --- | --- | --- | 
| `SendEmailJob` | Enqueued by any verification/notification flow | 5, exponential | [01](../01-registration/spec.md), [03](../03-authentication/spec.md), [06](../06-user-profile-update/spec.md), [20](../20-password-management/spec.md) |
| `UserDeletionJob` | `DELETE /users/{id}` when async | 3, then dead-letter to `failed` status | [07](../07-user-deletion/spec.md) |
| `AvatarObjectCleanupJob` | After avatar replace/remove, and on user deletion | 5, exponential | [13](../13-avatar-upload/spec.md) |
| `PersistTransformationResultJob` | `save=true` on a successful conversion | 3 | [12](../12-transformation-storage/spec.md) |
| `TransformationRetentionJob` | Recurring, hourly | n/a (idempotent) | [11](../11-transformation-history/spec.md), [12](../12-transformation-storage/spec.md) |
| `VerificationAttemptCleanupJob` | Recurring, every 15 min | n/a | [01](../01-registration/spec.md) |

### 3.1 Retention job

Deletes expired history rows **and** their stored objects in the same pass, so [12](../12-transformation-storage/spec.md)'s
"delete physical storage assets simultaneously when corresponding history database records expire"
holds. Order: delete object, then delete row. If the object delete fails, leave the row and retry
next hour — an orphaned row is recoverable, an orphaned object is not discoverable.

Batched (e.g. 500 rows per iteration) so a backlog cannot hold a transaction open.

### 3.2 Our own job-status table

[07](../07-user-deletion/spec.md) exposes `GET /users/{userId}/deletion-status`. That reads `BackgroundJobRecord`
(`Id`, `Type`, `TargetUserId`, `Status`, `CreatedAt`, `CompletedAt`, `FailureCode`), which the job
writes to. It does **not** read Hangfire's tables: the storage schema is an implementation detail
and must not leak into an API contract.

## 4. Hosting

- Server runs in-process in the API by default (`AddHangfireServer`), with worker count bounded so
  conversion work and job work do not contend — see [09](../09-data-file-transformation/spec.md) §4.
- The dashboard is mounted at `/admin/jobs` behind an authorization filter requiring the
  `settings:read` permission. Never anonymous, never on by default outside development.
- Jobs take a `CancellationToken` and honour graceful shutdown.
- Every job is idempotent: re-running a completed deletion or cleanup is a no-op, matching [07](../07-user-deletion/spec.md)
  §5's idempotency requirement.

## 5. Decisions

| # | Decision | Rationale | Rejected |
| ---  | --- | --- | --- | 
| 25.1 | Hangfire + `Hangfire.PostgreSql` | Gives persistence, retry with backoff, cron, and a dashboard — exactly the "observable and retryable" wording in [13](../13-avatar-upload/spec.md). No new infrastructure: it reuses the existing Postgres | `Quartz.NET` (scheduling only, no retry/dashboard story without extra work); `BackgroundService` + `Channel` (loses work on restart) |
| 25.2 | `Hangfire.Core` is LGPL-3.0 | Referenced unmodified as a NuGet package. If the org forbids LGPL entirely, substitute Quartz.NET + a hand-written retry record; the `IJobEnqueuer` port makes that a one-adapter change | Hangfire Pro (paid, excluded by licence policy) |
| 25.3 | `IJobEnqueuer` port | Keeps the licence-sensitive dependency replaceable and the application layer testable | Injecting `IBackgroundJobClient` directly |
| 25.4 | Own `BackgroundJobRecord` table | [07](../07-user-deletion/spec.md)'s status contract must not be coupled to Hangfire's schema | Querying `JobStorage` monitoring API |
| 25.5 | In-process server, not a separate worker process | One deployable at this size; the server can be split out later by flag without code change | Separate worker container from day one |
| 25.6 | Email always goes through a job | [06](../06-user-profile-update/spec.md) requires non-blocking dispatch; making it universal means no caller has to remember | Inline send with fire-and-forget `Task` (lost on restart, unobservable) |

## 6. Risks

- **In-process server contention.** Conversion is CPU-bound and jobs share the thread pool. Worker
  count must be set explicitly, not left at the default `ProcessorCount * 5`.
- **Dashboard exposure.** A Hangfire dashboard left unauthenticated is a full remote-execution
  surface. The authorization filter is not optional, and an integration test asserts a `403` for an
  unprivileged caller.
- **At-least-once delivery.** Jobs can run twice (retry after a timeout that actually succeeded).
  Idempotency is a requirement of every job, not a nice-to-have.
