# Feature: Session Authorization & JWT Cookie Handling

## 1. Overview
* **Purpose:** Authenticate incoming HTTP requests to protected endpoints using JWTs transmitted via secure HTTP-Only cookies.
* **Target Roles:** Authenticated users requesting protected resources.
* **Dependencies:**
  * JWT signing key & algorithm (e.g., RS256 or HS256).
  * Cookie options (Domain, Path, Secure, HttpOnly, SameSite).

## 2. Core Constraints & Token Lifecycle

* **Token Types & TTL:**
  * **Access Token:** 15 minutes TTL.
  * **Refresh Token:** 30 days TTL.
* **Refresh Token Rotation:** Enabled. Every successful refresh request yields a completely new pair of access and refresh tokens.
* **Stateless Refresh Architecture (CRITICAL):**
  * **Server-side storage of refresh tokens is STRICTLY FORBIDDEN.**
  * Do NOT maintain allowlists, denylists, database session records, or track `jti` in persistent storage for refresh tokens.
  * *Architectural Consequence:* Server-side active revocation of issued tokens is not supported. Token lifespan relies entirely on TTL and client-side cookie clearing.

## 3. Cookie Configuration Defaults
All tokens must be set via `Set-Cookie` headers with the following enforcement:
* `HttpOnly`: `true` (prevents JavaScript access / XSS token theft).
* `Secure`: `true` (enforced over HTTPS).
* `SameSite`: `Lax` or `Strict` (configured per cross-domain requirements).
* Cookie expiration must match token `exp` timestamp.

## 4. Technical Contract

### 4.1 Request Authorization (Protected Endpoints)
* **Input:** Cookie named `access_token`.
* **Validation Steps:**
  1. Verify cookie presence.
  2. Validate JWT signature against active secret/public key.
  3. Verify token claims (`exp` is not expired, `nbf`/`iat` valid, `iss`/`aud` match config).
  4. Extract `sub` (User ID).
  5. Fetch user record to confirm active/unblocked status.
  6. Inject `userId` and auth context into NestJS execution context (e.g., `req.user`).
* **Responses:**
  * **Success:** Continue pipeline execution to controller.
  * **Failure:** Return `401 Unauthorized`.

### 4.2 Token Refresh Endpoint
* **Endpoint:** `POST /auth/refresh`
* **Input:** Cookie named `refresh_token`.
* **Execution Logic:**
  1. Verify `refresh_token` cookie presence and valid cryptographic signature.
  2. Ensure token `exp` is valid (no database lookup performed).
  3. Generate new Access Token (15-min TTL) and new Refresh Token (30-day TTL).
  4. Return response with updated `Set-Cookie` headers for both tokens.
* **Responses:**
  * `200 OK`: Tokens rotated successfully.
  * `401 Unauthorized`: Missing, malformed, or expired refresh token.

### 4.3 Logout Endpoint
* **Endpoint:** `POST /auth/logout`
* **Execution Logic:**
  1. Set expired clear headers for `access_token` and `refresh_token` cookies.
  2. Return `200 OK` (Client must perform redirect to login page).

## 5. Audit & Logging
* Log failed JWT signature or expiration validations (exclude token payloads or raw signature details).
* Log access attempts to protected endpoints for auditing.
* **Forbidden:** Never log raw JWT strings, secret keys, or sensitive PII.