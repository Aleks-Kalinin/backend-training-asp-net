# 18 — API Contract Rules

## 1. Overview

- **Purpose:** The rules that keep the published API contract accurate enough for a client to be
  generated from it, and the conventions every endpoint follows. The mechanics of document
  generation are in [28](../28-openapi-and-versioning/spec.md); this document states the rules
  endpoints must satisfy.
- **Target roles:** all API consumers.
- **Why it exists separately:** an inaccurate or untyped OpenAPI document produces weak or wrong
  client types, and the failure is silent — the document still generates, the client still
  compiles, and the mismatch surfaces at runtime.

## 2. Request conventions

| Rule | Detail |
| --- | --- |
| Route prefix | `/api/v{version:apiVersion}`, v1 throughout |
| Body format | JSON, camelCase |
| Unknown properties | Rejected with `400` `unknown_field` ([23](../23-validation-and-partial-updates/spec.md) §2.1) |
| Partial updates | `Optional<T>` presence semantics; absent and explicit-null are distinct |
| Multipart options | **Typed fields**, one per option, each independently validated |
| Enums | Serialised as strings, with the full value set in the schema |
| Dates | ISO-8601 with offset |

### 2.1 Multipart option fields

Conversion endpoints take their options as discrete, typed multipart fields rather than a nested
JSON blob:

| Endpoint | Fields |
| --- | --- |
| `POST /convert` | `file`, `targetFormat`, `save` |
| `POST /images/convert` | `file`, `targetFormat`, `quality`, `width`, `height`, `background`, `save` |

Ranges are validated and documented in the schema: `width` 1–1920, `height` 1–1080,
`quality` 1–100. A value outside its range returns `400` with a field-specific message.

One input shape per piece of data means one parsing path and no precedence rule to specify or
test.

## 3. Response conventions

| Rule | Detail |
| --- | --- |
| Named types | **Every** response body is a named DTO. No anonymous shapes, no `object` |
| Declared responses | Every action declares its success type and each documented failure status |
| Nullability | Honest: `photo`, `nextCursor` and `role` are nullable where they can be null |
| Binary | Download and conversion endpoints declare `application/octet-stream` or the concrete MIME type, with `format: binary` |
| Errors | RFC 9457 `ProblemDetails` with a stable `code` extension ([24](../24-error-handling/spec.md)) |

### 3.1 Shared response DTOs

| DTO | Shape | Used by |
| --- | --- | --- |
| `MessageResponseDto` | `{ message }` | Login, verify, refresh, logout |
| `VerificationRequiredResponseDto` | `{ message, verificationRequired: true, attemptId }` | `202` from sign-up and login |
| `DeletionChallengeResponseDto` | `{ requiresConfirmation: true, challengeId, message }` | First step of self-deletion |
| `EmailChangeChallengeResponseDto` | `{ requiresConfirmation: true, challengeId, message }` | Email-change initiation |
| `ConversionFormatDto` | `{ source, target[] }` | Both format-discovery endpoints |
| `HistoryItemDto` | See [11](../11-transformation-history/spec.md) §2.1 | Both history endpoints |
| `PaginatedHistoryResponseDto` | `{ items[], nextCursor }` | Both history endpoints |
| `PaginatedUsersResponseDto` | `{ items: AdminUserResponseDto[], nextCursor }` | Admin user list |
| `CurrentUserResponseDto` | See [14](../14-current-user/spec.md) §2.1 | `/auth/me` |
| `SelfUserResponseDto`, `AdminUserResponseDto`, `PublicUserResponseDto` | See [15](../15-admin-user-access/spec.md) §4 | Every profile-returning endpoint |
| `ProblemDetails` | RFC 9457 plus `code` | Every `4xx` and `5xx` |

History items expose `downloadable` and never the internal file relation or a storage path.

## 4. Authentication in the contract

Authentication is **cookie-based**. The document declares a single security scheme:

```yaml
components:
  securitySchemes:
    accessTokenCookie:
      type: apiKey
      in: cookie
      name: access_token
```

Every `[Authorize]` operation references it; `[AllowAnonymous]` operations do not. **There is no
bearer scheme**, because there is no bearer auth path — documenting one would advertise a second,
weaker way in.

Browsers do not let a scripted documentation UI set cookies, so interactive "try it" works only
after a real login in the same origin. State that in the API description rather than adding a
bearer fallback.

## 5. Cross-origin configuration

CORS origins come from configuration (`Cors:Origins`), never from a hardcoded list. Startup fails
on a malformed entry ([26](../26-configuration-and-settings/spec.md) §2.1). `AllowCredentials` is
required because auth rides on cookies, which makes a wildcard origin illegal.

`Content-Disposition` must be in the exposed-headers list or browsers cannot read download
filenames ([31](../31-security-rate-limiting-and-cors/spec.md) §2).

## 6. Audit identifiers

Every audited operation records `actorUserId` as the authenticated caller and `targetUserId` as the
subject. On admin endpoints these are different users, and transposing them makes the audit trail
actively misleading.

`AuditEvent` is a named-parameter record, so the orientation is structural rather than positional,
and an integration test asserts it on the admin history endpoint
([27](../27-observability-and-audit-logging/spec.md) §3).

## 7. The enforcing control

All of the above is enforced by one mechanism rather than by review: the OpenAPI document is
generated at build time, committed as `openapi/v1.json`, and CI fails when the generated document
differs from the committed one ([28](../28-openapi-and-versioning/spec.md) §5).

Every contract change then arrives as a reviewable diff. Without it, these rules decay silently.

## 8. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 18.1 | `ProblemDetails` with a `code` extension | Framework-native, carries extensions cleanly, and `code` is a stronger contract than a `message` string that may also be an array | A custom `{ statusCode, message, error }` envelope |
| 18.2 | Typed multipart fields only | One parsing path, one schema, no precedence rule | Also accepting a nested `options` JSON blob |
| 18.3 | Cookie security scheme only | Matches the real mechanism; a bearer fallback would be a second auth path to secure | Documenting bearer for documentation-UI convenience |
| 18.4 | Committed OpenAPI snapshot diffed in CI | Converts silent contract drift into a build failure | Trusting runtime generation and reviewer attention |
| 18.5 | Named DTOs for every response | Generated clients need named types; anonymous shapes produce `unknown` | Example-only response documentation |
| 18.6 | `/api/v1` prefix retained | Explicit versioning, visible in logs and docs, and cheap to add a v2 beside | Unversioned routes |

## 9. Risks and open points

- **Snapshot diff noise.** Many pull requests will touch `openapi/v1.json`. Sort the document
  deterministically before writing it, or reviewers learn to ignore the file and the control stops
  working.
- **Transformer coverage.** Binary and multipart schemas are produced by hand-written document
  transformers and can drift from the real handlers. Integration tests that post real multipart
  bodies are the backstop.
- **`downloadable` must not become a storage hint.** Keep it a boolean; never add a URL or key to
  the history item.

## 10. Acceptance criteria

1. `POST /images/convert` honours `width`, `height`, `quality` and `save` as separate typed fields,
   and rejects out-of-range values with `400`.
2. Every JSON endpoint has a typed response schema in the generated document; a client generator
   produces no `unknown` response types.
3. The document declares the cookie security scheme and no bearer scheme.
4. Admin audit entries show the admin as actor and the viewed user as target.
5. CORS origins come from configuration, and startup fails on a malformed value.
6. CI fails when the committed OpenAPI snapshot is stale.
7. Every `4xx` response in integration tests is a `ProblemDetails` body carrying a `code`.
