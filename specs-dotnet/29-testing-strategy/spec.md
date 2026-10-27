# 29 — Testing Strategy

> The test-side counterpart to [19](../19-test-infrastructure/spec.md), which covers running the
> service as a process for browser-driven tests.

## 1. Overview

- **Purpose:** Test taxonomy, tooling, fixtures, and the specific scenarios the feature documents
  list as acceptance criteria.
- **Stack:** xUnit v3, NSubstitute, Shouldly, Testcontainers, Respawn,
  `Microsoft.AspNetCore.Mvc.Testing`.

## 2. Taxonomy

| Layer | Project | Scope | Speed |
| ---  | --- | --- | --- | 
| Unit | `FileConverter.UnitTests` | Domain policies, services with substituted ports, converters | ms, no I/O |
| Integration | `FileConverter.IntegrationTests` | Full HTTP pipeline via `WebApplicationFactory`, real Postgres in a container | seconds |
| Architecture | `FileConverter.UnitTests/Architecture` | Project dependency rule from [21](../21-architecture-solution-layout/spec.md) §2 | ms |

There is no separate "e2e" .NET project. Full-stack browser testing is Playwright against a running
process, configured in [19](../19-test-infrastructure/spec.md).

### 2.1 Unit tests — the policy oracles

These carry the most security weight and must be exhaustive, one case per table row:

- `UserWritePolicy` — every cell of the writable-field table ([23](../23-validation-and-partial-updates/spec.md) §4).
- Profile field sets — `SELF`, `ADMIN`, `PUBLIC` ([05](../05-user-profile-view/spec.md), [15](../15-admin-user-access/spec.md)).
- Permission expansion — a grant with empty `actions` expands to every action of that permission
  ([14](../14-current-user/spec.md), [02](../02-rbac/spec.md)).
- Protection rules — each row of [17](../17-user-role-assignment/spec.md).
- Cursor encode/decode, including tampered and wrong-sort cursors.
- Conversion mapping rules — CSV->JSON row shape, XML attribute `@` prefix, JSON->XML `<root>`/
  `<item>`, BOM handling ([09](../09-data-file-transformation/spec.md)).

### 2.2 Integration tests

`CustomWebApplicationFactory` overrides:

| Real service | Test substitute |
| ---  | --- | 
| PostgreSQL | Testcontainers `postgres:17`, migrations applied once per collection |
| SMTP | Testcontainers `axllent/mailpit`, asserted through its HTTP API |
| Avatar storage | `local` driver under `NODE_ENV`-equivalent `E2E`, or an in-memory fake |
| Result storage | In-memory fake implementing `IResultStorage` |
| Hangfire | In-memory storage; jobs executed inline by a test `IJobEnqueuer` |
| `TimeProvider` | `FakeTimeProvider` for TTL and expiry tests |

- One container per test collection, `Respawn` between tests to reset data while keeping the
  schema and the seeded roles/permissions.
- Authentication helpers (`AsAdmin()`, `AsUser()`, `AsAnonymous()`) issue real cookies through the
  real login endpoint — never by forging a JWT, so the cookie and guard paths are exercised.

### 2.3 Scenarios required by the feature documents

| Scenario | Source |
| ---  | --- | 
| Role change reflected immediately by `/auth/me`, and only after refresh in `RbacGuard` | [14](../14-current-user/spec.md), [17](../17-user-role-assignment/spec.md) |
| Two **concurrent** demotions cannot remove the last two admins | [17](../17-user-role-assignment/spec.md) |
| `GET /admin/users` returns `403` for a user, `200` for an admin, `401` anonymous | [15](../15-admin-user-access/spec.md) |
| Custom role granted `users:read` receives the admin field set | [15](../15-admin-user-access/spec.md) |
| Avatar replace cleans up the prior object; DB failure does not lose the active avatar | [13](../13-avatar-upload/spec.md) |
| User deletion removes the avatar object | [13](../13-avatar-upload/spec.md) |
| Unrelated profile PATCH preserves the avatar | [13](../13-avatar-upload/spec.md) |
| Oversized / corrupt / mismatched uploads produce 413 / 400 / 415 | [13](../13-avatar-upload/spec.md), [10](../10-image-transformation/spec.md) |
| XXE payload in an XML source file is inert | [09](../09-data-file-transformation/spec.md) |
| SVG containing a script is rejected or sanitised | [10](../10-image-transformation/spec.md) |
| Password reset response identical for existing and unknown email | [20](../20-password-management/spec.md) |
| Expired history item download returns 404/410 | [12](../12-transformation-storage/spec.md) |
| Hangfire dashboard returns 403 without `settings:read` | [25](../25-background-jobs/spec.md) §6 |

### 2.4 Architecture tests

Assert the dependency rule: `Domain` references no project; `Application` references only `Domain`;
`Infrastructure` and `Api` do not reference each other; no type outside `Infrastructure` references
`Hangfire`, `Npgsql`, `SkiaSharp` or `Supabase`.

## 3. Conventions

- Naming: `MethodOrEndpoint_Condition_ExpectedResult`.
- One assertion subject per test; Shouldly for readable failures.
- No `Thread.Sleep` anywhere — `FakeTimeProvider` for time, explicit waits for containers.
- Test data through builders (`UserBuilder.Active().WithRole(Roles.Admin).Build()`), not shared
  mutable fixtures.
- Coverage is measured but not gated. The gate is §2.3: those scenarios must exist and pass.

## 4. Decisions

| # | Decision | Rationale | Rejected |
| ---  | --- | --- | --- | 
| 29.1 | xUnit v3 | Current generation, first-class `IAsyncLifetime`, best Testcontainers ergonomics | NUnit; MSTest |
| 29.2 | Testcontainers over a shared compose database | Tests are self-contained and parallel-safe; no developer has to remember to start anything. [19](../19-test-infrastructure/spec.md)'s compose profile still exists for running the app, not for these tests | compose `test` profile as the test dependency (ordering and cleanup problems) |
| 29.3 | NSubstitute | Readable syntax, no licence or telemetry history | Moq (SponsorLink episode); FakeItEasy |
| 29.4 | Shouldly | Permissive licence, good failure messages | FluentAssertions >= 8 (commercial); raw `Assert` (poor diagnostics) |
| 29.5 | Respawn between tests, container per collection | Container startup dominates runtime; checkpoint reset is milliseconds | Fresh container per test (slow); transaction rollback per test (breaks code under test that manages its own transactions) |
| 29.6 | Real login for auth in tests | Forged tokens would skip the cookie extraction and guard code that specs 04 and 02 are mostly about | Injecting a fake `ClaimsPrincipal` |
| 29.7 | Mailpit as a container, asserted via HTTP API | [19](../19-test-infrastructure/spec.md) already chose Mailpit for Playwright; reusing it means one OTP-reading mechanism for both suites, and no test-only endpoint that exposes codes | Fake `IEmailSender` capturing in memory (would not exercise the templates the Playwright suite parses) |

## 5. Risks

- **Container startup in CI.** Testcontainers needs a Docker daemon on the runner. If an
  environment cannot provide one, the fallback is a service container plus a connection string, and
  the factory must support both.
- **Concurrency test flakiness.** The two-concurrent-demotions test depends on real serialisation
  conflicts. Write it to assert the *invariant* (exactly one succeeds, an admin still exists)
  rather than which request wins.
- **Timing-based assertions.** The constant-time login check from
  [24](../24-error-handling/spec.md) §6 is inherently noisy; keep it out of the blocking suite.
