# 26 — Configuration & Runtime Settings

> Two different things travel under the name "configuration": deployment values fixed at process
> start, and **admin-editable runtime settings** stored in the database. They have different
> lifetimes and need different mechanisms.

## 1. Overview

- **Tier 1 — Deployment configuration.** Immutable for the process lifetime. Options pattern,
  validated at startup, fails fast.
- **Tier 2 — Runtime settings.** Changed by administrators through the API without a restart.
  Database-backed, cached, invalidated on write.

## 2. Tier 1: deployment configuration

Sources, in precedence order: `appsettings.json` -> `appsettings.{Environment}.json` ->
user secrets (development only) -> environment variables (`__` as the section separator) ->
command line.

| Options class | Section | Keys (abbreviated) |
| --- | --- | --- |
| `JwtOptions` | `Jwt` | `SigningKey`, `Issuer`, `Audience`, `AccessTokenMinutes` (15), `RefreshTokenDays` (30) |
| `AuthCookieOptions` | `Auth:Cookies` | `Domain`, `SameSite`, `Secure` |
| `DatabaseOptions` | `Database` | `ConnectionString`, `RunMigrationsOnStartup` |
| `MailOptions` | `Mail` | `SmtpHost`, `SmtpPort`, `FromAddress`, `FromName` |
| `AvatarStorageOptions` | `Storage:Avatars` | `Driver` (`supabase` \| `local`), `LocalDirectory`, `PublicBaseUrl` |
| `SupabaseOptions` | `Storage:Supabase` | `ProjectUrl`, `ServiceRoleKey`, `Bucket` |
| `ResultStorageOptions` | `Storage:Results` | `Driver` (`local` \| `s3`), `Bucket`, `Endpoint` |
| `ConversionOptions` | `Conversion` | `MaxConcurrency`, `TimeoutSeconds` (30), `MaxImageDimension` |
| `RateLimitOptions` | `RateLimits` | per-policy windows and permits — [31](../31-security-rate-limiting-and-cors/spec.md) |
| `CorsOptions` | `Cors` | `Origins` (array; [18](../18-api-contract/spec.md)'s comma-separated `CORS_ORIGINS` becomes a bound array) |
| `CursorOptions` | `Pagination` | `SigningKey` — [22](../22-persistence-and-migrations/spec.md) §4 |

Registration pattern:

```csharp
services.AddOptions<JwtOptions>()
        .BindConfiguration("Jwt")
        .ValidateDataAnnotations()
        .Validate(o => o.SigningKey.Length >= 32, "Jwt:SigningKey must be at least 32 bytes")
        .ValidateOnStart();
```

`ValidateOnStart()` is what makes misconfiguration a startup failure: the process
refuses to start rather than failing on first use.

### 2.1 Conditional requirements (fail-fast rules)

Implemented as `IValidateOptions<T>`:

1. `Storage:Supabase` keys are **required** when `Storage:Avatars:Driver` is `supabase`
   ([13](../13-avatar-upload/spec.md)). Missing or malformed -> startup failure with a clear message.
2. `Storage:Avatars:Driver = local` is **only** permitted when the environment is `Test` or `E2E`
   ([19](../19-test-infrastructure/spec.md)). In any other environment, startup fails. There is no silent fallback to local
   disk — [13](../13-avatar-upload/spec.md) forbids it.
3. `Cors:Origins` must be non-empty and every entry a well-formed absolute URI ([18](../18-api-contract/spec.md)).
4. `Jwt:SigningKey` must not equal any value shipped in `appsettings.json` — a committed default
   key must be impossible to run with.

### 2.2 Secrets

- Development: `dotnet user-secrets`.
- CI / e2e: environment variables from the runner's secret store.
- Committed files (`appsettings.json`, `appsettings.Development.json`, `.env.example`,
  `docker-compose.yml`) contain **names and placeholders only**, never a real key. The Supabase
  service-role key is server-side only and never reaches a response or a log
  ([13](../13-avatar-upload/spec.md)).

## 3. Tier 2: runtime settings

Backed by an `app_settings` table (`Key`, `Value` as `jsonb`, `UpdatedAt`, `UpdatedBy`).

| Setting | Type | Default | Used by |
| ---  | --- | --- | --- | 
| `registration_verification_enabled` | bool | `false` | [01](../01-registration/spec.md) |
| `registration_verification_strategy` | `otp` \| `magic_link` | `otp` | [01](../01-registration/spec.md) |
| `registration_enumeration_defense` | bool | `true` | [01](../01-registration/spec.md), [24](../24-error-handling/spec.md) §4.1 |
| `login_verification_enabled` | bool | `false` | [03](../03-authentication/spec.md) |
| `login_verification_strategy` | `otp` \| `magic_link` | `otp` | [03](../03-authentication/spec.md) |
| `password_reset_verification_enabled` | bool | `true` | [20](../20-password-management/spec.md) |
| `file_size_limit_{csv,json,xml,yaml}` | bytes | 10 MiB | [09](../09-data-file-transformation/spec.md) |
| `file_size_limit_{png,jpeg,svg}` | bytes | 10 MiB | [10](../10-image-transformation/spec.md) |
| `transformation_retention_days` | int | 90 | [11](../11-transformation-history/spec.md), [12](../12-transformation-storage/spec.md) |

Accessed through `ISettingsProvider` (`Application/Settings/Ports/`), backed by `HybridCache` with
a short TTL plus explicit invalidation on write — the same mechanism and the same cross-instance
notifier as the RBAC cache in [02](../02-rbac/spec.md) §4. Admin endpoints
`GET`/`PUT /api/v1/admin/settings` require `settings:read` / `settings:update`.

A setting read must never hit the database on the hot path; a cache miss loads the whole settings
map in one query.

## 4. Decisions

| # | Decision | Rationale | Rejected |
| ---  | --- | --- | --- | 
| 26.1 | Options pattern + `ValidateOnStart` for tier 1 | Validates the whole configuration surface at startup; typed access everywhere | Reading `IConfiguration` ad hoc (no validation, stringly typed) |
| 26.2 | Separate DB-backed tier 2 | The toggles in specs 01/03/20 must change without a restart and are audited; environment variables cannot do that | `Microsoft.FeatureManagement` (file/config-sourced, no admin API, no audit trail) |
| 26.3 | `appsettings` + env vars, `.env` only for compose | .NET configuration already layers sources properly; compose still uses a `.env` file only to feed container environment variables | Porting `.env` loading into the app (`DotNetEnv`) |
| 26.4 | Conditional `IValidateOptions` for driver rules | [19](../19-test-infrastructure/spec.md) requires a *clear* failure, and [13](../13-avatar-upload/spec.md) forbids silent fallback — both are startup concerns | Runtime checks on first use (fails late, in a request) |
| 26.5 | Settings cached through the same notifier as RBAC | One invalidation mechanism to reason about and test | A second, parallel cache design |

## 5. Risks

- **Two tiers confuse contributors.** The rule of thumb belongs in the README: *if an administrator
  can change it from the UI, it is tier 2; otherwise tier 1.*
- **Settings cache staleness across instances.** Single-instance deployments are fine with the
  in-process notifier. Multi-instance requires the Redis adapter; running multi-instance without it
  is a silent correctness bug, so the startup log must state which notifier is active.
