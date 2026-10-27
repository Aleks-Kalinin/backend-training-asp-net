# 24 — Error Handling

## 1. Overview

- **Purpose:** One error body for every failure, a stable machine-readable error code vocabulary,
  and the exception taxonomy that produces the status codes the feature documents require.
- **Format:** RFC 9457 Problem Details (`application/problem+json`).

## 2. Response shape

```json
{
  "type": "https://docs.fileconverter.internal/errors/email-already-registered",
  "title": "Email already registered",
  "status": 409,
  "detail": "An account with this email address already exists.",
  "instance": "/api/v1/users",
  "code": "email_already_registered",
  "traceId": "00-4bf92f...-01"
}
```

- `code` is the contract. Clients branch on `code`, never on `title` or `detail`.
- `detail` is human-readable, safe to show, and never contains PII, field values, stack traces,
  storage paths, SQL or provider messages.
- `traceId` is the W3C trace id, so a support request maps to a log line — see
  [27](../27-observability-and-audit-logging/spec.md) §3.
- Validation failures add an `errors` object: `{ "email": ["must be a valid email address"] }`.

## 3. Exception taxonomy

`FileConverter.Domain/Exceptions/`:

| Exception | Status | Typical `code` |
| --- | --- | --- |
| `ValidationException` | 400 | `validation_failed`, `invalid_cursor`, `unknown_field` |
| `UnauthorizedException` | 401 | `not_authenticated`, `invalid_token` |
| `ForbiddenException` | 403 | `permission_denied`, `field_not_writable`, `self_access_required`, `feature_disabled` |
| `NotFoundException` | 404 | `user_not_found`, `history_item_not_found` |
| `ConflictException` | 409 | `email_already_registered`, `already_deleted`, `last_admin_protected`, `role_in_use` |
| `GoneException` | 410 | `file_expired` |
| `PayloadTooLargeException` | 413 | `file_too_large` |
| `UnsupportedMediaTypeException` | 415 | `unsupported_source_format` |
| `UnprocessableException` | 422 | `invalid_otp`, `expired_token` |
| `RateLimitedException` | 429 | `rate_limited`, `verification_attempts_exceeded` |
| `StorageException` | 500 | `storage_unavailable` |

Every one carries `Code` and optional `Extensions`. Application services throw these; controllers
contain no `try`/`catch` and no manual `StatusCode(...)` construction.

### 3.1 Handler chain

`IExceptionHandler` implementations registered in order:

1. `DomainExceptionHandler` — maps the table above.
2. `OperationCanceledHandler` — client disconnect -> `499`-equivalent logged at debug, no body;
   conversion timeout ([09](../09-data-file-transformation/spec.md)) -> `504` with code `conversion_timeout`.
3. `FallbackExceptionHandler` — logs at `error` with the full exception, returns a generic `500`
   with code `internal_error` and the `traceId`. Never leaks the message.

`app.UseExceptionHandler()` plus `AddProblemDetails()` with a customisation hook that stamps
`code`, `traceId` and `instance` onto every response, including the ones ASP.NET Core produces
itself (404 route miss, 405, 415 from content negotiation).

## 4. Error code vocabulary

Codes are a closed set in `Domain/Errors/ErrorCodes.cs` (`const string` fields), and a test asserts
that every code thrown anywhere appears in that file. This is what keeps the frontend's exhaustive
`switch` honest.

Naming: `snake_case`, noun-first, no HTTP status in the name
(`email_already_registered`, not `conflict_email`).

### 4.1 Enumeration-safe codes

Specs 01 §6, 03 §5 and 20 §2.1 require that responses not reveal whether an account exists.
Therefore:

- Login failure for unknown email and for wrong password both return `401` /
  `invalid_credentials`, with identical `detail` and comparable timing.
- Password reset request always returns `202` / no error, regardless of account existence.
- Registration with a taken email returns `409` / `email_already_registered` **only when the
  `registration_enumeration_defense` setting is off**; otherwise it returns the same `202` as a
  fresh registration. The setting is defined in [26](../26-configuration-and-settings/spec.md).

Those paths must not short-circuit: do the password-hash comparison against a dummy hash even when
the user does not exist, so response time does not distinguish the cases.

## 5. Decisions

| # | Decision | Rationale | Rejected |
| ---  | --- | --- | --- | 
| 24.1 | RFC 9457 `ProblemDetails` | Framework-native, understood by generated clients, carries extensions cleanly. No legacy consumer constrains the shape | A custom `{ statusCode, message, error }` envelope, which means fighting the framework for no benefit |
| 24.2 | `code` extension as the client contract | `title`/`detail` are prose and will be reworded; clients need something stable | Branching on `status` alone (too coarse — `409` means five different things here) |
| 24.3 | `IExceptionHandler` chain over an MVC exception filter | Catches failures from middleware too (rate limiter, auth, multipart), so the shape is truly universal | `IExceptionFilter` (MVC pipeline only) |
| 24.4 | Domain exceptions over result types | Every failure path here is genuinely exceptional and terminates the request | `Result<T>` / OneOf (plumbing cost across five layers for little gain at this size) |
| 24.5 | Closed error-code set with a guard test | Prevents codes drifting into string literals at throw sites | Ad-hoc strings |
| 24.6 | Generic `500` body, full detail only in logs | [13](../13-avatar-upload/spec.md) and [12](../12-transformation-storage/spec.md) forbid leaking storage/provider internals | Returning provider messages in `detail` |

## 6. Risks

- **Status-code drift.** Several specs allow alternatives (`200` vs `204` on deletion, optional
  `410`). Each feature document pins one; the integration tests assert it. Do not leave it to the
  implementer.
- **Timing side channels.** The constant-time requirement in §4.1 is easy to regress — any early
  `return` added to the login path reintroduces the oracle. Covered by a test that compares
  distributions, which is inherently flaky; treat a failure as a prompt to inspect the code path
  rather than as a hard gate.
