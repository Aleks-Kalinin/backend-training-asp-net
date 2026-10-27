# Feature: API Contract Fixes for Frontend Integration

## 1. Overview

- **Purpose:** Fix places where the OpenAPI document (`/docs-json`) or runtime behaviour does not match what clients need. The frontend generates its typed client from `/docs-json` with Orval, so inaccurate or untyped Swagger output turns into weak or wrong frontend types.
- **Target Roles:** All API consumers.
- **Dependencies:** Swagger setup (`src/core/swagger.ts`), conversion module, auth module, `src/main.ts`.

## 2. Technical Contract

### 2.1 Image Conversion Options

- **Problem:** `POST /images/convert` documents `width`, `height` and `quality` as separate multipart fields, but `ConversionService.convertMultipartRequest` reads them only from an `options` field containing JSON (`{"width":800,"quality":80,"save":true}`). Separate fields are ignored.
- **Fix:** Accept **both** forms. Separate `width`, `height`, `quality` and `save` fields are parsed and validated as numbers / boolean. If both forms are sent, separate fields win. Update the Swagger body schema to document the separate fields, `save`, and the `options` JSON field as deprecated.
- Validate ranges and return `400` outside them: width 1–1920, height 1–1080, quality 1–100.

### 2.2 Text Conversion `save` Field

- Document the `save` boolean field in the `POST /convert` Swagger body schema (it is already parsed but undocumented). Only items converted with `save=true` can be downloaded later ([spec 12](../12-transformation-storage/spec.md)).

### 2.3 Typed Responses

Replace example-only Swagger responses with DTO classes:

- `ConversionFormatDto` `{ source: string; target: string[] }` for `GET /convert/formats` and `GET /images/convert/formats`.
- `HistoryItemDto` `{ id, type, sourceFormat, targetFormat, status, fileSize, durationMs, errorCode, createdAt, downloadable }` and `PaginatedHistoryResponseDto` `{ items: HistoryItemDto[]; nextCursor: string | null }` for both history endpoints.
  - `downloadable` is `true` when a stored file exists for the item. Do not return the internal `file` relation or storage paths.
- `MessageResponseDto` `{ message: string }` for login, verify, refresh and logout.
- `VerificationRequiredResponseDto` `{ message; verificationRequired: true; attemptId }` for the `202` responses of login and sign-up.
- `DeletionChallengeResponseDto` `{ requiresConfirmation: true; challengeId; message }` for the first step of self-deletion.
- `EmailChangeChallengeResponseDto` for `POST /users/{userId}/email-change`.
- Binary download endpoints declare `application/octet-stream` (or the concrete MIME type) with `format: binary`.

### 2.4 Cookie Authentication in Swagger

- Controllers use `@ApiBearerAuth()`, but authentication is cookie-based (`access_token`). Replace with `@ApiCookieAuth('access_token')` and register the cookie security scheme in `swagger.ts`.

### 2.5 Admin History Audit IDs

- In `ConversionController.getAllTransformationHistory`, `targetUserId` receives the admin's own ID and `userId` the target. Swap them so logs record `actorUserId` = admin and `targetUserId` = viewed user, matching [spec 11](../11-transformation-history/spec.md).

### 2.6 CORS Origins From Configuration

- Replace the hardcoded origin list in `main.ts` with a `CORS_ORIGINS` environment variable (comma-separated), validated in `config.validation.ts`.
- Default in `.env.example`: `http://localhost:5173` (Vite's default port).

### 2.7 Consistent Error Body

- Document the standard NestJS error body `{ statusCode, message, error }` as `ErrorResponseDto` and reference it from `4xx` responses, so the frontend can read `message` reliably. `message` may be a string or a string array (validation errors).

## 3. Error Handling & Edge Cases

- Image option fields with non-numeric or out-of-range values return `400` with a field-specific message.
- Existing clients sending `options` JSON keep working.

## 4. Audit & Logging

- No new logs; 2.5 fixes the IDs in an existing log.

## 5. Non-Functional & Security Requirements

- No breaking change to request formats or response bodies, except that responses which previously had no schema now have one.
- The history DTO must not reveal storage locations or other users' data.

## 6. Acceptance Criteria

1. `POST /images/convert` honours separate `width`, `height`, `quality` and `save` fields, and still accepts `options` JSON.
2. Every JSON endpoint used by the frontend has a typed response schema in `/docs-json`; Orval generates no `unknown` response types for them.
3. Swagger UI authenticates with the `access_token` cookie scheme.
4. Admin history logs show the admin as actor and the viewed user as target.
5. CORS origins come from `CORS_ORIGINS`; startup fails clearly if the variable is malformed.
6. Existing unit and e2e tests pass; new tests cover option parsing in both forms.
