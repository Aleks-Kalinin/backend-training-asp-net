# 19 — Test Infrastructure for Full-Stack E2E

## 1. Overview

- **Purpose:** Let the backend run as a real process for full-stack browser tests — frontend,
  backend and database together — locally and in CI, without external services.
- **The problem it solves:** OTP codes are delivered by email, so sign-up, login verification,
  email change, self-deletion and password reset cannot be tested end to end unless the test can
  read the mail.
- **Target roles:** developers and CI.
- **Scope:** this document covers running the service for browser-driven tests. The .NET test
  taxonomy and tooling are in [29](../29-testing-strategy/spec.md).
- **Dependencies:** the mail module, the avatar storage port
  ([13](../13-avatar-upload/spec.md) §4), configuration validation
  ([26](../26-configuration-and-settings/spec.md)), `docker-compose.yml`
  ([30](../30-build-ci-and-docker/spec.md) §3.1).

## 2. Mail catcher

- A `mailpit` service (`axllent/mailpit`) in `docker-compose.yml`, SMTP port `1025`, web and API
  port `8025`, under a **`test` profile** so it does not start by default in development.
- The end-to-end configuration points `Mail:SmtpHost` and `Mail:SmtpPort` at it.
- Browser tests read OTP codes through Mailpit's HTTP API (`GET /api/v1/messages`, filtered by
  recipient). **The backend needs no test-only code for this.**
- OTP emails must contain the six-digit code in a stable, parseable form — a line reading
  `Your code: 123456`. The format is part of the mail module's contract and is covered by a test,
  because the browser suite parses it ([01](../01-registration/spec.md) §4).

## 3. Avatar storage driver

`Storage:Avatars:Driver` takes `supabase` (default) or `local`.

- `local` stores files under `Storage:Avatars:LocalDirectory` and serves them from a static route,
  for example `/api/v1/test-assets/avatars/...`, so `photo` URLs load in a browser.
- **`local` is permitted only when the environment is `Test` or `E2E`.** Startup fails with a clear
  message if it is configured anywhere else. This preserves the rule in
  [13](../13-avatar-upload/spec.md) §5: no *silent* fallback to local disk. An explicit,
  environment-guarded switch is not a silent fallback.
- Supabase variables remain required whenever the driver is `supabase`.
- The selected driver is logged once at startup.

Validation lives in `IValidateOptions<AvatarStorageOptions>`
([26](../26-configuration-and-settings/spec.md) §2.1).

## 4. End-to-end environment profile

`appsettings.E2E.json` plus environment variables:

| Setting | Value |
| --- | --- |
| `DOTNET_ENVIRONMENT` | `E2E` |
| `Database:ConnectionString` | A dedicated e2e database |
| `Database:RunMigrationsOnStartup` | `true` — exercises real migrations and the seed data in [17](../17-user-role-assignment/spec.md) §5 |
| `Storage:Avatars:Driver` | `local` |
| `Mail:Smtp*` | Mailpit |
| `Cors:Origins` | Includes the frontend preview port |
| `RateLimits:*` | Raised so parallel tests are not throttled |

Started with `dotnet run --project src/FileConverter.Api -c Release` and
`DOTNET_ENVIRONMENT=E2E` ([30](../30-build-ci-and-docker/spec.md) §2).

Committed configuration files contain names and placeholders only — **no real credentials**.

## 5. Test data seeding

```
dotnet run --project src/FileConverter.Cli -- e2e seed
```

- Runs **only** against the e2e database, guarded by a database check: the command refuses unless
  the target database name matches the configured e2e pattern **and** the environment is `E2E`.
- Truncates user data, then creates fixed accounts: one active `ADMIN`, one active `USER`, one
  `blocked` user, one `pending` user.
- Passwords come from environment variables, **never from source**.
- Resets runtime settings ([26](../26-configuration-and-settings/spec.md) §3) to known values —
  **all verification toggles off** — so each test turns on only what it needs.
- Clears the local avatar directory as part of the truncate step.
- Shares the application's DI container, so seeded passwords are hashed by the same
  `IPasswordHasher` the API uses and roles go through the same lookup.

## 6. Error handling

- Starting with the local avatar driver outside `Test` or `E2E` fails at startup with a clear
  message naming the setting and the environment.
- `e2e seed` refuses to run unless the database guard confirms a test database. It **fails
  closed**: an unknown environment, an unrecognised database name, or missing configuration all
  refuse.

## 7. Audit and logging

Log the selected avatar storage driver once at startup.

## 8. Non-functional and security requirements

- **No test-only HTTP endpoints** that expose OTP codes or bypass verification. A code-reading
  endpoint is a production vulnerability waiting for a configuration mistake.
- **No real credentials** in any committed configuration file.

## 9. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 19.1 | Mailpit via compose for browser tests; Testcontainers for the .NET suite | Browser tests need a long-lived process to point at; the .NET integration suite benefits from per-collection isolation. Same image, same parsing, two lifecycles | One mechanism for both — either slow browser-suite startup or fragile shared state |
| 19.2 | No test-only endpoints | A code-reading endpoint is one misconfiguration away from being a production hole | A `/test/otp/{email}` helper |
| 19.3 | Environment-guarded local avatar driver | Explicit, and impossible outside test environments, which satisfies both the testability need and the no-silent-fallback rule | An unguarded driver switch; an in-memory fake only, whose URLs would not load in a browser |
| 19.4 | `RunMigrationsOnStartup` in E2E only | The e2e run should exercise real migrations and seed data; production applies them deliberately | Auto-migrating everywhere |
| 19.5 | Seed command shares application DI | Hashing, role lookup and guards stay identical to runtime behaviour | A SQL seed script |
| 19.6 | Stable OTP email format as a tested contract | The browser suite parses it; an unannounced template change would break it mysteriously | Treating the template as free text |

## 10. Risks and open points

- **The database guard is the only thing standing between `e2e seed` and real data.** Test the
  refusal path, not just the happy path.
- **Raised rate limits in E2E** mean the browser suite never exercises `429` paths. Cover those in
  the .NET integration tests, where limits can be set per test.
- **Mailpit retains every message** for the container's lifetime, so tests must filter by recipient
  and timestamp rather than taking the newest message, or they will race each other under parallel
  execution.

## 11. Acceptance criteria

1. `docker compose --profile test up` starts PostgreSQL and Mailpit, and OTP emails sent by the
   backend appear in Mailpit.
2. The backend starts with the local avatar driver under `E2E`, and uploaded avatar URLs load in a
   browser.
3. The same setting under `Development` or `Production` stops startup with a clear message.
4. `e2e seed` creates the fixed accounts on the e2e database and refuses to run against any other.
5. The .NET integration suite passes unchanged alongside this setup.
6. No committed file contains a real credential.
