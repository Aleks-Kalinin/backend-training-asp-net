# 28 — OpenAPI & API Versioning

> How the OpenAPI document is produced and kept accurate. The contract rules endpoints must satisfy
> are in [18](../18-api-contract/spec.md).

## 1. Overview

- **Purpose:** How the OpenAPI document is produced, where it is served, and the rules that keep it
  accurate enough to generate a client from.
- **Stack:** `Microsoft.AspNetCore.OpenApi` for generation, `Scalar.AspNetCore` for the UI,
  `Asp.Versioning.Mvc.ApiExplorer` for versioning.

[18](../18-api-contract/spec.md) exists because an inaccurate Swagger document produced weak frontend types. The rules below
are the preventive version of that spec.

## 2. Versioning

- URL-segment versioning: `/api/v{version:apiVersion}/...`, default `1.0`, reported in
  `api-supported-versions`.
- `[ApiVersion("1.0")]` on every controller. One document per version, served at
  `/openapi/v1.json`; the UI at `/scalar`.
- Everything in this directory is v1. Adding v2 means a new controller, never an edit to a v1
  response shape.

## 3. Document quality rules

These are enforced by the CI check in §5, not by good intentions.

1. **Every action declares its responses.** `[ProducesResponseType<T>(200)]` for the success shape
   and `[ProducesResponseType<ProblemDetails>(4xx)]` for each documented failure in the owning
   spec. An action with an undeclared `200` body fails the check.
2. **No anonymous or `object` response types.** Every response is a named DTO, so the generated
   client gets a named type. This is [18](../18-api-contract/spec.md) applied universally — including
   `ConversionFormatDto`, `HistoryItemDto`, `PaginatedHistoryResponseDto`, `MessageResponseDto`,
   `VerificationRequiredResponseDto`, `DeletionChallengeResponseDto`,
   `EmailChangeChallengeResponseDto`, `CurrentUserResponseDto`, `AdminUserResponseDto`.
3. **Binary endpoints** declare `Content-Type: application/octet-stream` (or the concrete MIME
   type) with `format: binary`, via an operation transformer. Covers `POST /convert`,
   `POST /images/convert` and both download endpoints.
4. **Multipart endpoints** declare their form schema explicitly — `file`, `targetFormat`, `save`,
   and for images `width`, `height`, `quality` — with their validation ranges in the schema.
5. **`Optional<T>` unwrapping.** A schema transformer renders `Optional<T>` as `T`, optional, so
   PATCH bodies look normal to a client generator. See
   [23](../23-validation-and-partial-updates/spec.md) §3.
6. **Nullability is honest.** `photo` is `string | null`, `nextCursor` is `string | null`,
   `role` is nullable on `CurrentUserResponseDto` ([14](../14-current-user/spec.md)).
7. **Enums as strings**, with the full value set in the schema.

## 4. Security scheme

Authentication is cookie-based. The document declares:

```yaml
components:
  securitySchemes:
    accessTokenCookie:
      type: apiKey
      in: cookie
      name: access_token
```

Applied via an `IOpenApiDocumentTransformer`, with `[Authorize]`-decorated operations marked as
requiring it and `[AllowAnonymous]` ones left open. No `bearerAuth` anywhere — that was the defect
in [18](../18-api-contract/spec.md).

Browsers do not let scripted UI clients set cookies, so the Scalar "try it" flow works only after a
real login in the same origin. Document that in the API description rather than adding a bearer
fallback, which would create a second, weaker auth path.

## 5. CI enforcement

`dotnet run --project src/FileConverter.Api -- --dump-openapi <path>` writes the document at build
time. CI then:

1. Fails if the committed `openapi/v1.json` snapshot differs from the generated one — the document
   is reviewed in PRs like any other contract.
2. Runs a linter over the document asserting rules §3.1, §3.2 and §3.6.

A snapshot test in `FileConverter.IntegrationTests` covers the same ground locally.

## 6. Decisions

| # | Decision | Rationale | Rejected |
| ---  | --- | --- | --- | 
| 28.1 | `Microsoft.AspNetCore.OpenApi` | In-box from .NET 9, no third-party dependency, AOT-friendly, transformer model is enough for the cookie scheme and binary bodies | Swashbuckle (no longer in templates, heavier); NSwag (client generation we do not need here) |
| 28.2 | Scalar for the UI | Actively maintained, single package, reads the generated document directly | Swagger UI via Swashbuckle (drags in the generator we rejected) |
| 28.3 | URL-segment versioning | Visible, cacheable, matches the `/api/v1` prefix used throughout | Header or query-string versioning (invisible in logs and docs) |
| 28.4 | Committed document snapshot + diff check | Turns an accidental contract change into a reviewable diff. This is the control that would have caught every defect in [18](../18-api-contract/spec.md) | Generating docs only at runtime |
| 28.5 | Cookie security scheme only | Matches the actual mechanism; a bearer fallback would be a second auth path to secure | Documenting bearer for Swagger convenience |

## 7. Risks

- **Snapshot churn.** The document changes on many PRs. Keep the diff readable by sorting the
  document deterministically before writing it.
- **Transformer coverage.** Binary and multipart schemas are hand-written in transformers; they can
  drift from the real handlers. The integration tests that post real multipart bodies are the
  backstop.
