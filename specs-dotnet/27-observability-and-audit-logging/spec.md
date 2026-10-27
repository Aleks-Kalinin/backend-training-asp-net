# 27 — Observability & Audit Logging

> Every feature document states audit requirements and a list of values that must never be logged.
> This document makes those requirements one mechanism rather than scattered ad-hoc log calls.

## 1. Overview

- **Purpose:** Structured logging, the audit-event catalogue, redaction rules, metrics and traces.
- **Stack:** Serilog (`Serilog.AspNetCore`), OpenTelemetry (traces + metrics),
  `Microsoft.Extensions.Diagnostics.HealthChecks`.

## 2. Logging

- Serilog replaces the default provider. Console sink with the compact JSON formatter in
  production, human-readable template in development.
- Enrichers: `TraceId`, `SpanId`, `UserId` (from the authenticated principal), `RequestPath`,
  `Environment`.
- `UseSerilogRequestLogging()` for one summary line per request (method, path, status, elapsed).
  `GET /api/v1/auth/me` is excluded from request logging and succeeds silently — [14](../14-current-user/spec.md) states
  it is called on every app load and must not log on success.
- Levels: `Information` for lifecycle and audit, `Warning` for rejected-but-expected outcomes
  (`403` field writes, protection-rule hits, failed JWT validation), `Error` for unexpected
  failures only. A `429` is `Warning`, not `Error`.

## 3. Audit events

Audit is not ordinary logging: each event has a named actor, subject and outcome. One abstraction:

```csharp
public interface IAuditLogger
{
    void Record(AuditEvent e);
}

public sealed record AuditEvent(
    string Operation,            // "user.profile.update"
    Guid? ActorUserId,
    Guid? TargetUserId,
    int StatusCode,
    IReadOnlyDictionary<string, object?>? Extras = null);
```

Emitted as a structured log entry with `AuditOperation` as a first-class property, so the audit
stream is filterable (`AuditOperation is not null`) without a separate sink.

### 3.1 Catalogue

| Operation | Extras | Source |
| --- | --- | --- |
| `auth.register` | `outcome` | [01](../01-registration/spec.md) |
| `auth.verification.dispatch` | `purpose`, `strategy` | [01](../01-registration/spec.md), [03](../03-authentication/spec.md) |
| `auth.verification.confirm` | `purpose`, `outcome` | [01](../01-registration/spec.md), [03](../03-authentication/spec.md) |
| `auth.login` | `outcome` | [03](../03-authentication/spec.md) |
| `auth.token.rejected` | `reason` (`signature`, `expired`, `missing`) | [04](../04-authorization-session/spec.md) |
| `rbac.admin.mutate` | `entity`, `entityId`, `action` | [02](../02-rbac/spec.md) |
| `rbac.cache.reload` | `trigger` | [02](../02-rbac/spec.md) |
| `user.profile.view` | — | [05](../05-user-profile-view/spec.md) |
| `user.profile.update` | `fields` (**names only**) | [06](../06-user-profile-update/spec.md) |
| `user.create` | `roleName` | [16](../16-user-write-hardening/spec.md) |
| `user.delete` | `mode` (`self`\|`admin`), `jobId`, `outcome` | [07](../07-user-deletion/spec.md) |
| `user.role.change` | `previousRole`, `newRole` | [17](../17-user-role-assignment/spec.md) |
| `user.protection.rejected` | `rule` | [17](../17-user-role-assignment/spec.md) (warn level) |
| `admin.users.list` | `filterKeys`, `count` | [08](../08-admin-user-list/spec.md), [15](../15-admin-user-access/spec.md) |
| `transformation.execute` | `type`, `sourceFormat`, `targetFormat`, `fileSize`, `durationMs` | [09](../09-data-file-transformation/spec.md), [10](../10-image-transformation/spec.md) |
| `transformation.history.query` | `filterKeys`, `count` | [11](../11-transformation-history/spec.md) |
| `storage.operation` | `action` (`save`\|`download`), `fileId`, `size`, `durationMs` | [12](../12-transformation-storage/spec.md) |
| `avatar.operation` | `action`, `bytes`, `outcome` | [13](../13-avatar-upload/spec.md) |
| `password.reset.request` | `emailMasked`, `userFound` | [20](../20-password-management/spec.md) |

`actorUserId` is always the authenticated caller and `targetUserId` always the subject. [18](../18-api-contract/spec.md)
Transposing them on an admin endpoint makes the audit trail actively misleading, so `AuditEvent`
uses named parameters and an integration test asserts the orientation.

## 4. Redaction — what must never be logged

Collected from every spec's "Forbidden" clause:

| Never logged | Source |
| ---  | --- | 
| Passwords, password hashes | [16](../16-user-write-hardening/spec.md), [20](../20-password-management/spec.md) |
| OTP codes, magic-link tokens, `attemptId` secrets | [20](../20-password-management/spec.md) |
| Raw JWT strings, signing keys, signature details | [04](../04-authorization-session/spec.md) |
| Raw file contents (data or image bytes) | [09](../09-data-file-transformation/spec.md), [10](../10-image-transformation/spec.md), [11](../11-transformation-history/spec.md) |
| Profile field **values** (names only) | [05](../05-user-profile-view/spec.md), [06](../06-user-profile-update/spec.md) |
| Storage paths, bucket keys, signed URLs, Supabase service-role key | [12](../12-transformation-storage/spec.md), [13](../13-avatar-upload/spec.md) |
| Raw full-text search parameters that may contain PII | [08](../08-admin-user-list/spec.md) |
| Deleted/scrubbed PII | [07](../07-user-deletion/spec.md) |

Enforcement, in order of strength:

1. **Types that cannot be logged.** `Password`, `OtpCode` and `StorageKey` are wrapper structs
   whose `ToString()` returns `"***"`. A value that cannot be stringified cannot leak through an
   interpolated log message.
2. **Serilog destructuring policies** that mask known property names (`password`, `token`, `otp`,
   `code`, `key`, `secret`, `photo`, `storageKey`).
3. **Email masking** helper (`a***@example.com`) used by the password-reset and bootstrap paths.
4. A unit test asserting each wrapper type's `ToString()` and each destructuring policy.

## 5. Metrics & traces

OpenTelemetry, OTLP exporter, disabled by default in development.

- Auto-instrumentation: ASP.NET Core, HttpClient, Npgsql.
- Custom `Meter` `FileConverter`:
  - `conversion.duration` (histogram, ms; tags `type`, `source_format`, `target_format`, `status`)
  - `conversion.bytes_in` (histogram)
  - `conversion.rejected` (counter; tag `reason`)
  - `verification.dispatched` / `verification.failed` (counters; tag `purpose`)
  - `storage.operation.duration` (histogram; tags `action`, `driver`)
- `durationMs` in the audit event and in the transformation history row comes from the same
  stopwatch as the metric, so the API and the dashboard never disagree.

### 5.1 Health checks

`/health/live` (process up) and `/health/ready` (Postgres, plus the configured avatar storage
driver and SMTP when they are required). Both anonymous, neither exposes exception detail.

## 6. Decisions

| # | Decision | Rationale | Rejected |
| ---  | --- | --- | --- | 
| 27.1 | Serilog over the built-in logger | Destructuring policies are the cleanest way to enforce the forbidden-field lists, and structured sinks are needed for the audit stream | `Microsoft.Extensions.Logging` alone (no redaction hooks) |
| 27.2 | Audit as structured logs, not an audit table | Nothing requires querying audit records through the API; log aggregation is the right home. An `audit_events` table can be added later behind the same `IAuditLogger` | DB audit table from day one (schema churn, retention burden) |
| 27.3 | `IAuditLogger` with a typed `AuditEvent` | Makes the actor/target orientation structural rather than positional — the exact bug [18](../18-api-contract/spec.md) reports | Free-form `logger.LogInformation("...")` |
| 27.4 | Unloggable wrapper types for secrets | The only mechanism that survives a careless `$"{...}"` | Review and discipline |
| 27.5 | OpenTelemetry, exporter off by default | Several endpoints report `durationMs`; a `Meter` provides it once. No infrastructure is forced on a developer running locally | Serilog-only timing (no aggregation) |
| 27.6 | `/auth/me` excluded from request logging | Direct requirement of [14](../14-current-user/spec.md) | Logging everything uniformly |

## 7. Risks

- **Redaction is a denylist.** A new sensitive field is unprotected until someone adds it. The
  wrapper types cover the worst offenders; everything else relies on review.
- **Audit volume.** `user.profile.view` and `transformation.execute` are high-frequency. If log
  cost becomes a problem, sample non-audit request logs — never the audit stream.
