# 22 — Persistence & Migrations

## 1. Overview

- **Purpose:** EF Core conventions, the migration and seeding strategy, keyset pagination, and the
  transaction and isolation rules this service requires.
- **Dependencies:** PostgreSQL 17, EF Core 10, `Npgsql.EntityFrameworkCore.PostgreSQL`,
  `EFCore.NamingConventions`.

## 2. DbContext

One `AppDbContext` in `FileConverter.Infrastructure/Persistence/`. Entity configuration lives in
`IEntityTypeConfiguration<T>` classes, one per entity, applied with
`ApplyConfigurationsFromAssembly`. No data annotations on domain entities — `Domain` must not
reference EF Core.

```csharp
services.AddDbContext<AppDbContext>(o => o
    .UseNpgsql(cs, npg => npg.EnableRetryOnFailure(3))
    .UseSnakeCaseNamingConvention());
```

| Convention | Rule |
| ---  | --- | 
| Keys | `Guid`, generated in application code with `Guid.CreateVersion7()`, never by the database |
| Timestamps | `DateTimeOffset` -> `timestamptz`; `CreatedAt`/`UpdatedAt` set in `SaveChangesAsync` override |
| Enums | Stored as `text` with an explicit value converter, never as `int` and never as a PG enum type |
| Deletes | Hard delete for most entities; users are anonymised, not removed — [07](../07-user-deletion/spec.md) |
| Concurrency | `xmin` as a concurrency token on `users` and `user_roles` |
| Tracking | `QueryTrackingBehavior.NoTracking` by default; opt in with `AsTracking()` on write paths |

Guid v7 is monotonic, so primary-key inserts stay index-friendly and `id` works as a stable
tiebreaker in keyset pagination (§4).

## 3. Entities

| Entity | Owner document | Notes |
| --- | --- | --- |
| `User` | [01](../01-registration/spec.md), [16](../16-user-write-hardening/spec.md) | `Email` (citext), `PasswordHash`, `Status`, `IsVerified`, `Photo` (storage path, nullable), `LastLoginAt`, `PasswordChangedAt` |
| `Role`, `Permission`, `Grant`, `UserRole` | [02](../02-rbac/spec.md), [17](../17-user-role-assignment/spec.md) | `UserRole` is many-to-many in schema, one row per user by application rule |
| `VerificationAttempt` | [01](../01-registration/spec.md), [03](../03-authentication/spec.md) | Discriminated by `Purpose` (registration, login, email change, deletion, password reset); holds `CodeHash`, `ExpiresAt`, `AttemptCount`, `ConsumedAt` |
| `PendingEmailChange` | [06](../06-user-profile-update/spec.md) | |
| `TransformationHistoryItem` | [11](../11-transformation-history/spec.md) | |
| `StoredFile` | [12](../12-transformation-storage/spec.md) | `StorageKey`, `ExpiresAt`, nullable FK from history item |
| `BackgroundJobRecord` | [07](../07-user-deletion/spec.md), [25](../25-background-jobs/spec.md) | Our own status row; never expose Hangfire internals |
| `AppSetting` | [26](../26-configuration-and-settings/spec.md) | Admin-editable runtime settings |

Use the PostgreSQL `citext` extension for `users.email` so uniqueness and lookup are
case-insensitive without `LOWER()` indexes. Normalise to lowercase on write anyway.

### 3.1 Required indexes

Derived from [08](../08-admin-user-list/spec.md) and [11](../11-transformation-history/spec.md):

| Table | Index |
| ---  | --- | 
| `users` | unique `email`; `(status, created_at desc)`; `(last_login_at desc)`; GIN trigram on `email` for `q` search |
| `transformation_history` | `(user_id, created_at desc, id desc)`; `(type)`; `(status)`; `(expires_at)` for the pruning job |
| `stored_files` | `(expires_at)` |
| `verification_attempts` | `(user_id, purpose, expires_at)`; `(expires_at)` for cleanup |
| `user_roles` | unique `(user_id)` — enforces "exactly one role" at the database level |

The unique index on `user_roles.user_id` is what actually guarantees [17](../17-user-role-assignment/spec.md)'s one-role rule under
concurrency. The application check alone is a race.

## 4. Keyset (cursor) pagination

Specs 08 and 11 both mandate cursor pagination. One shared helper in
`Application/Common/Pagination/`.

- The cursor encodes the sort key and the tiebreaker id: `{ "k": <sortValue>, "id": <guid> }`,
  serialised compactly, base64url-encoded, then **HMAC-signed** with a server key.
- An unparseable, mis-signed or wrong-shaped cursor returns `400` with error code
  `invalid_cursor`. Signing is what makes that distinguishable from tampering.
- A cursor is bound to the sort field and direction it was issued for; changing `sort` or `order`
  mid-pagination invalidates it (`400`).
- Query shape: `WHERE (sort_col, id) < (@k, @id) ORDER BY sort_col DESC, id DESC LIMIT @n + 1`.
  Fetch `limit + 1` rows to compute `nextCursor` without a second count query. No `COUNT(*)`.

## 5. Migrations

- `dotnet ef migrations add <Name> --project src/FileConverter.Infrastructure --startup-project src/FileConverter.Api`
- Applied by `dotnet run --project src/FileConverter.Cli -- db migrate`, or automatically at startup
  when `Database:RunMigrationsOnStartup` is `true` (used in e2e, see
  [19](../19-test-infrastructure/spec.md) §2.3). Never automatic in production.
- Migrations are additive and never edited once merged.
- CI fails if the model and the migrations have drifted:
  `dotnet ef migrations has-pending-model-changes`.

### 5.1 Seed data

[17](../17-user-role-assignment/spec.md) requires idempotent seeding of roles, permissions and grants that must not overwrite
administrator changes. Implement as **raw SQL inside a migration**, not as a runtime seeder:

```sql
INSERT INTO roles (id, name, description) VALUES (...)
ON CONFLICT (name) DO NOTHING;
```

Grants are inserted the same way, keyed on `(role_id, permission_id)`. `DO NOTHING` — never
`DO UPDATE` — is what satisfies "must not overwrite grants an administrator has changed".
Re-running migrations on a seeded database changes zero rows.

## 6. Transactions & isolation

| Case | Rule |
| ---  | --- | 
| Profile update ([06](../06-user-profile-update/spec.md)) | Single `SaveChangesAsync` — implicitly transactional |
| Role change / last-admin check ([17](../17-user-role-assignment/spec.md)) | Explicit `IsolationLevel.Serializable` transaction wrapping both the count and the write, with retry on PostgreSQL `40001` |
| Avatar upload ([13](../13-avatar-upload/spec.md)) | Not a DB transaction: upload -> DB update -> compensating delete on failure. Documented in [13](../13-avatar-upload/spec.md) §4 |
| Deletion pipeline ([07](../07-user-deletion/spec.md)) | Per-step, resumable; the job record is the unit of progress, not one long transaction |

Serializable retry is a small shared helper (`IExecutionStrategy` plus a `40001` catch), used by
the role-change and user-block paths only. Do not apply it globally.

## 7. Decisions

| # | Decision | Rationale | Rejected |
| ---  | --- | --- | --- | 
| 22.1 | EF Core 10 as the only data access path | Migrations, change tracking and LINQ cover every query in this service; the heaviest query is a keyset page with filters | Dapper (would need a parallel migration story); EF + Dapper hybrid (premature) |
| 22.2 | Guid v7 application-generated keys | Index locality without a DB round trip; stable pagination tiebreaker; no identity-gap surprises | `bigint` identity (exposes row counts); Guid v4 (index fragmentation) |
| 22.3 | snake_case columns via `EFCore.NamingConventions` | The prevailing PostgreSQL idiom; zero per-property configuration | Default PascalCase columns (quoting noise in raw SQL) |
| 22.4 | HMAC-signed opaque cursors | [08](../08-admin-user-list/spec.md) requires `400` on an invalid cursor; signing distinguishes malformed from forged and stops clients depending on the encoding | Plain base64 offset (leaks shape, invites tampering); offset pagination (spec forbids) |
| 22.5 | Seed via idempotent SQL migration | [17](../17-user-role-assignment/spec.md) requires idempotence and migration-time availability (e2e runs real migrations) | Runtime `IHostedService` seeder (races across instances) |
| 22.6 | `citext` for email | Case-insensitive uniqueness without functional indexes | `LOWER()` unique index (every query must remember to match it) |
| 22.7 | Unique index on `user_roles.user_id` | Makes [17](../17-user-role-assignment/spec.md)'s one-role invariant a database guarantee | Application-only check (racy) |
| 22.8 | `NoTracking` default | Most endpoints are reads; avoids accidental writes through projections | Tracking default |

## 8. Risks

- **Serializable retries under load.** The last-admin path is low-traffic, so this is safe here.
  If the pattern spreads to hot paths, switch to advisory locks on a per-aggregate key instead.
- **Trigram search cost.** [08](../08-admin-user-list/spec.md)'s `q` parameter over `email`/`id`/name needs `pg_trgm`. On a
  large users table this index is not free; revisit if the admin list is used heavily.
- **`citext` is an extension.** It must be created in a migration (`CREATE EXTENSION IF NOT EXISTS
  citext`) and the database role needs rights to do so — check in managed-Postgres environments.
