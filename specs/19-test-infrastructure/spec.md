# Feature: Test Infrastructure for Full-Stack E2E

## 1. Overview

- **Purpose:** Let the backend run as a real process for full-stack Playwright tests (frontend + backend + database) locally and in CI, without external services.
- **Current state:** Backend e2e tests (Jest, `test/e2e`) start the app in-process and replace avatar storage with an in-memory fake. Supabase settings are optional only when `NODE_ENV=test`. Emails cannot be read by tests, so OTP flows (sign-up, login 2FA, email change, self-deletion) cannot be tested end to end.
- **Target Roles:** Developers and CI.
- **Dependencies:** Mail module (`NodemailerMailTransport`), avatar storage port, config validation, `docker-compose.yml`, [spec 13](../13-avatar-upload/spec.md), frontend tooling spec (`specs-frontend/08-tooling-ci`).

## 2. Technical Contract

### 2.1 Mail Catcher (Mailpit)

- Add a `mailpit` service to `docker-compose.yml` (`axllent/mailpit`, SMTP port `1025`, web/API port `8025`), under a `test` profile so it does not start by default in development unless requested.
- `.env.test.example` and CI point `SMTP_HOST`/`SMTP_PORT` at Mailpit.
- Playwright tests read OTP codes through the Mailpit HTTP API (`GET /api/v1/messages`, filtered by recipient). The backend needs no test-only code for this.
- OTP emails must contain the 6-digit code in a stable, parseable form (for example a line `Your code: 123456`). Document the format in the mail module.

### 2.2 Explicit Test Avatar Storage

- Add `AVATAR_STORAGE_DRIVER` with values `supabase` (default) and `local`.
- `local` stores files under `AVATAR_LOCAL_DIR` and serves them from a static route (for example `/api/v1/test-assets/avatars/...`), so `photo` URLs work in the browser.
- `local` is allowed **only when `NODE_ENV` is `test` or `e2e`**. Startup fails if `local` is configured in any other environment. This keeps [spec 13](../13-avatar-upload/spec.md)'s rule: no *silent* fallback to local disk.
- Supabase variables remain required whenever the driver is `supabase`.

### 2.3 E2E Environment Profile

- Add `.env.e2e.example` for running the backend as a process for Playwright:
  - `NODE_ENV=e2e`, dedicated database, `POSTGRES_MIGRATIONS_RUN=true` (exercises real migrations and seed data from [spec 17](../17-user-role-assignment/spec.md)).
  - `AVATAR_STORAGE_DRIVER=local`, Mailpit SMTP settings, `CORS_ORIGINS` including the Vite preview port.
  - Throttling limits raised so parallel tests are not rate-limited.
- Add an npm script `start:e2e` that builds and starts the app with this profile.

### 2.4 Test Data Seeding

- Add `npm run e2e:seed` that, against the e2e database only (guarded by the existing `test-database-guard`), truncates user data and creates fixed accounts: one active `ADMIN`, one active `USER`, one `blocked` user, one `pending` user. Passwords come from environment variables, not source code.
- Verification settings are reset to known values (all off) so tests turn them on explicitly.

## 3. Error Handling & Edge Cases

- Starting with `AVATAR_STORAGE_DRIVER=local` outside `test`/`e2e` fails with a clear message.
- `e2e:seed` refuses to run unless the database guard confirms a test database.

## 4. Audit & Logging

- Log the selected avatar storage driver once at startup.

## 5. Non-Functional & Security Requirements

- No test-only HTTP endpoints that expose OTP codes or bypass verification.
- No real credentials in any `.env.*.example` file.

## 6. Acceptance Criteria

1. `docker compose --profile test up` starts Postgres and Mailpit; OTP emails sent by the backend appear in Mailpit.
2. The backend starts with `AVATAR_STORAGE_DRIVER=local` under `NODE_ENV=e2e`, and uploaded avatar URLs load in a browser.
3. The same setting under `NODE_ENV=development` or `production` stops startup.
4. `npm run e2e:seed` creates the fixed accounts on the e2e database and refuses to run against other databases.
5. Existing Jest e2e tests keep passing unchanged.
