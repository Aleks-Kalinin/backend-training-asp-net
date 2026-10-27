# File Converter Backend — Specifications (ASP.NET Core)

The complete specification for the File Converter backend: a .NET 10 / ASP.NET Core service
providing authentication, database-driven RBAC, user management, data-file and image conversion,
transformation history, and result storage.

Each document is self-contained and authoritative for its feature. Documents `01`–`20` specify
features: purpose, endpoints, validation, execution logic, error handling, audit requirements,
implementation decisions and risks. Documents `21`–`31` specify cross-cutting concerns that every
feature depends on.

Read `21` first, then `22`–`24`, then any feature document.

## 1. Contents

### Features

| # | Document | Covers |
| --- | --- | --- |
| 01 | [registration](./01-registration/spec.md) | Sign-up, optional email verification (OTP / magic link) |
| 02 | [rbac](./02-rbac/spec.md) | Roles, permissions, grants, authorization pipeline, cache |
| 03 | [authentication](./03-authentication/spec.md) | Login, optional login verification, brute-force defence |
| 04 | [authorization-session](./04-authorization-session/spec.md) | JWT cookies, refresh rotation, logout |
| 05 | [user-profile-view](./05-user-profile-view/spec.md) | Profile read, field-level masking |
| 06 | [user-profile-update](./06-user-profile-update/spec.md) | Profile patch, email-change flow |
| 07 | [user-deletion](./07-user-deletion/spec.md) | Deletion and anonymisation pipeline |
| 08 | [admin-user-list](./08-admin-user-list/spec.md) | Paginated, filterable admin user list |
| 09 | [data-file-transformation](./09-data-file-transformation/spec.md) | CSV / JSON / XML / YAML conversion |
| 10 | [image-transformation](./10-image-transformation/spec.md) | PNG / JPEG / SVG conversion |
| 11 | [transformation-history](./11-transformation-history/spec.md) | Conversion audit history |
| 12 | [transformation-storage](./12-transformation-storage/spec.md) | Opt-in result storage and download |
| 13 | [avatar-upload](./13-avatar-upload/spec.md) | Supabase-hosted user avatars |
| 14 | [current-user](./14-current-user/spec.md) | `GET /auth/me` session endpoint |
| 15 | [admin-user-access](./15-admin-user-access/spec.md) | Admin list permission, admin field set |
| 16 | [user-write-hardening](./16-user-write-hardening/spec.md) | Writable-field policy, user creation |
| 17 | [user-role-assignment](./17-user-role-assignment/spec.md) | Role change, admin self-protection, seed data |
| 18 | [api-contract](./18-api-contract/spec.md) | API contract quality rules |
| 19 | [test-infrastructure](./19-test-infrastructure/spec.md) | Running the backend for full-stack E2E |
| 20 | [password-management](./20-password-management/spec.md) | Reset and change (planned) |

### Cross-cutting

| # | Document | Covers |
| --- | --- | --- |
| 21 | [architecture-solution-layout](./21-architecture-solution-layout/spec.md) | Projects, layering, DI, service shape |
| 22 | [persistence-and-migrations](./22-persistence-and-migrations/spec.md) | EF Core, entities, indexes, pagination, transactions |
| 23 | [validation-and-partial-updates](./23-validation-and-partial-updates/spec.md) | Validation pipeline, `Optional<T>`, writable-field policy |
| 24 | [error-handling](./24-error-handling/spec.md) | Problem Details, exception taxonomy, error codes |
| 25 | [background-jobs](./25-background-jobs/spec.md) | Job runner, job inventory, retention |
| 26 | [configuration-and-settings](./26-configuration-and-settings/spec.md) | Deployment config and admin-editable settings |
| 27 | [observability-and-audit-logging](./27-observability-and-audit-logging/spec.md) | Logging, audit catalogue, redaction, metrics |
| 28 | [openapi-and-versioning](./28-openapi-and-versioning/spec.md) | Document generation, security scheme, CI enforcement |
| 29 | [testing-strategy](./29-testing-strategy/spec.md) | Test taxonomy, tooling, required scenarios |
| 30 | [build-ci-and-docker](./30-build-ci-and-docker/spec.md) | Build properties, CLI verbs, images, pipeline |
| 31 | [security-rate-limiting-and-cors](./31-security-rate-limiting-and-cors/spec.md) | Rate limits, CORS, cookies, CSRF, size limits |

## 2. Target stack

| Concern | Choice | Package | Licence |
| --- | --- | --- | --- |
| Runtime / TFM | .NET 10 (LTS), `net10.0`, C# 14 | — | MIT |
| Web | ASP.NET Core, controller-based | — | MIT |
| DI / config / logging | `Microsoft.Extensions.*` | — | MIT |
| Database | PostgreSQL 17 | — | PostgreSQL |
| ORM | EF Core 10 | `Npgsql.EntityFrameworkCore.PostgreSQL` | PostgreSQL / MIT |
| Naming convention | snake_case columns | `EFCore.NamingConventions` | MIT |
| Validation | FluentValidation | `FluentValidation.DependencyInjectionExtensions` | Apache-2.0 |
| Mapping | Mapster, or hand-written | `Mapster` | MIT |
| AuthN | JWT in HttpOnly cookies | `Microsoft.AspNetCore.Authentication.JwtBearer` | MIT |
| Password hashing | bcrypt, work factor 12 | `BCrypt.Net-Next` | MIT |
| AuthZ / RBAC | Dynamic policy provider + handlers | — (built-in) | MIT |
| Cache | `HybridCache` (+ Redis for multi-instance) | `Microsoft.Extensions.Caching.Hybrid`, `StackExchange.Redis` | MIT |
| Rate limiting | Built-in rate limiter middleware | — | MIT |
| Background jobs | Hangfire + Postgres storage | `Hangfire.Core`, `Hangfire.PostgreSql` | **LGPL-3.0** / MIT |
| CSV | CsvHelper | `CsvHelper` | MS-PL / Apache-2.0 |
| JSON | `System.Text.Json` | — | MIT |
| XML | `System.Xml` with DTD/XXE disabled | — | MIT |
| YAML | YamlDotNet | `YamlDotNet` | MIT |
| Images | SkiaSharp (+ Svg.Skia for SVG rasterisation) | `SkiaSharp`, `Svg.Skia` | MIT |
| Object storage | Supabase Storage REST (avatars), local FS + S3 (results) | `AWSSDK.S3` | Apache-2.0 |
| Mail | MailKit / MimeKit | `MailKit` | MIT |
| OpenAPI | Built-in document generation + Scalar UI | `Microsoft.AspNetCore.OpenApi`, `Scalar.AspNetCore` | MIT |
| API versioning | URL-segment versioning | `Asp.Versioning.Mvc.ApiExplorer` | MIT |
| Logging | Serilog | `Serilog.AspNetCore` | Apache-2.0 |
| Tracing / metrics | OpenTelemetry | `OpenTelemetry.Extensions.Hosting` | Apache-2.0 |
| Tests | xUnit v3, NSubstitute, Shouldly, Testcontainers, Respawn | — | MIT / Apache-2.0 |

### 2.1 Licence policy

Permissive (MIT / Apache-2.0 / MS-PL) plus LGPL server libraries linked unmodified. **Nothing paid.**
Explicitly excluded and why:

| Excluded | Reason | Chosen instead |
| --- | --- | --- |
| `AutoMapper` >= 15 | Commercial licence | Mapster, or hand-written mappers |
| `MediatR` >= 13 | Commercial licence | Plain application services (no mediator needed here) |
| `FluentAssertions` >= 8 | Commercial licence | Shouldly |
| `SixLabors.ImageSharp` >= 3 | Split licence, commercial tier for most orgs | SkiaSharp + Svg.Skia |
| `Moq` | SponsorLink history, team preference | NSubstitute |
| Hangfire **Pro** | Paid | Hangfire Core (LGPL) is sufficient |

`Hangfire.Core` is LGPL-3.0. It is consumed as an unmodified NuGet reference, which keeps this
project's own source under its own terms. If the LGPL is unacceptable to the consuming org,
`Quartz.NET` (Apache-2.0) is the drop-in substitute — see [25](./25-background-jobs/spec.md) §6.

## 3. Global conventions

**Solution** — `FileConverter.sln`, projects described in [21](./21-architecture-solution-layout/spec.md):
`FileConverter.Domain`, `.Application`, `.Infrastructure`, `.Api`, `.Cli`, plus
`FileConverter.UnitTests` and `FileConverter.IntegrationTests`.

| Convention | Rule |
| --- | --- |
| Route prefix | `/api/v{version:apiVersion}`, v1 for everything in this directory |
| Identifiers | `Guid` created with `Guid.CreateVersion7()`, stored as `uuid` |
| Timestamps | `DateTimeOffset` in UTC, stored as `timestamptz`, serialised ISO-8601 |
| JSON | camelCase, `JsonSerializerOptions.UnmappedMemberHandling = Disallow` on request bodies |
| DB columns | snake_case via `EFCore.NamingConventions` |
| Errors | RFC 9457 `ProblemDetails` — see [24](./24-error-handling/spec.md) |
| Auth cookies | `access_token` (15 min), `refresh_token` (30 d); `HttpOnly`, `Secure`, `SameSite` per [31](./31-security-rate-limiting-and-cors/spec.md) |
| Nullable | `<Nullable>enable</Nullable>`, `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` |
| Async | Every I/O path is `async`, every public async method takes a `CancellationToken` |

## 4. Verification mechanics (shared)

Five flows need an emailed challenge: registration ([01](./01-registration/spec.md)), login
([03](./03-authentication/spec.md)), email change ([06](./06-user-profile-update/spec.md)),
self-deletion ([07](./07-user-deletion/spec.md)) and password reset
([20](./20-password-management/spec.md)). They share one mechanism, specified once in
[01](./01-registration/spec.md) §4:

| Parameter | Value |
| --- | --- |
| OTP | 6 digits, 10-minute TTL, max 5 attempts |
| Magic link | Single-use token, 10-minute TTL, bound to the attempt |
| Resend | 1 per 60 s, 3 per attempt |
| Storage | Codes and tokens stored hashed, compared in constant time |

## 5. Open decisions

Three questions are not settled by these documents and need a team decision before the affected
features are built. Each is argued in the document named.

| # | Question | Where |
| --- | --- | --- |
| 1 | Stateless refresh forbids server-side revocation, but deletion and password reset both need to end sessions. Proposed: a cached status/`pwd_at` check on the refresh endpoint only. | [07](./07-user-deletion/spec.md) §7, [20](./20-password-management/spec.md) §4 |
| 2 | Confirming the active-user status on every authenticated request costs a lookup per request. Proposed: a 30-second cached check with eager invalidation. | [04](./04-authorization-session/spec.md) §3.3 |
| 3 | Cookie authentication needs a CSRF defence. Proposed: `SameSite=Strict` on the refresh cookie plus a required custom header. | [31](./31-security-rate-limiting-and-cors/spec.md) §4 |

## 6. Notes for anyone coming from the Node implementation

This service replaces a Node + NestJS implementation. Where the two differ deliberately:

1. **Error bodies** are RFC 9457 `ProblemDetails` with a stable `code` extension, not
   `{ statusCode, message, error }`. Clients branch on `code`.
2. **Image conversion options** are typed multipart fields (`width`, `height`, `quality`, `save`).
   The legacy `options` JSON blob is not accepted.
3. **CPU-bound conversion** needs no worker-thread offloading on .NET. The real constraints —
   bounded concurrency and a hard timeout — are specified in
   [09](./09-data-file-transformation/spec.md) §6.
4. **ASP.NET Core Identity is not used.** It imposes its own schema, password hasher and
   server-side token stores, which contradicts the stateless-refresh architecture in
   [04](./04-authorization-session/spec.md).
5. **npm scripts** are verbs on the `FileConverter.Cli` project — see
   [30](./30-build-ci-and-docker/spec.md) §2.
6. **The database is new.** No migration of legacy avatar paths or plain-text passwords is needed;
   the correct schema and hashing exist from the first migration.
