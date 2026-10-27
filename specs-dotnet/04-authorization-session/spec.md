# 04 — Session Authorization & JWT Cookie Handling

## 1. Overview

- **Purpose:** Authenticate incoming HTTP requests to protected endpoints using JWTs carried in
  secure, `HttpOnly` cookies.
- **Target roles:** Authenticated users requesting protected resources.
- **Dependencies:** A JWT signing key and algorithm, cookie options (domain, path, `Secure`,
  `HttpOnly`, `SameSite`), and [02](../02-rbac/spec.md) for the authorization step that follows
  authentication.

## 2. Token lifecycle

| | Access token | Refresh token |
| --- | --- | --- |
| TTL | 15 minutes | 30 days |
| Cookie | `access_token` | `refresh_token` |
| Cookie path | `/` | `/api/v1/auth` |
| `SameSite` | `Lax` | `Strict` |
| `HttpOnly`, `Secure` | yes, yes | yes, yes |

**Refresh token rotation is enabled:** every successful refresh issues a completely new pair.

### 2.1 Stateless refresh architecture

**Server-side storage of refresh tokens is forbidden.** No allowlists, no denylists, no database
session records, no persisted `jti` tracking.

*Architectural consequence:* server-side active revocation of issued tokens is not supported.
Token lifespan relies entirely on TTL and client-side cookie clearing. This is a deliberate
constraint, restated here so that nobody later "fixes" it by adding a token table — see §7 for the
one narrow exception under consideration.

### 2.2 Claims

| Claim | Content |
| --- | --- |
| `sub` | User id (Guid) |
| `role` | Single role name — users have exactly one ([17](../17-user-role-assignment/spec.md)) |
| `typ` | `access` or `refresh` |
| `jti` | Token id, logged on rejection, **never persisted** |
| `iat`, `exp`, `iss`, `aud` | Standard |

The `typ` claim is load-bearing: without it, an access token presented in the refresh cookie would
mint a new 30-day session from a 15-minute credential.

## 3. Request authorization

**Input:** the `access_token` cookie.

**Validation steps:**

1. Verify the cookie is present.
2. Validate the JWT signature against the active key.
3. Verify claims: `exp` not expired, `nbf`/`iat` valid, `iss` and `aud` match configuration.
4. Extract `sub`.
5. Confirm the user is active and unblocked (§3.3).
6. Inject the user id and auth context into the request's `ClaimsPrincipal`.

Success continues the pipeline to the controller, where [02](../02-rbac/spec.md) applies the
permission check. Failure returns `401`.

### 3.1 Handler configuration

```csharp
options.Events = new JwtBearerEvents
{
    OnMessageReceived = ctx =>
    {
        ctx.Token = ctx.Request.Cookies["access_token"];
        return Task.CompletedTask;
    }
};
options.MapInboundClaims = false;   // keep 'sub' as 'sub'
options.TokenValidationParameters = new()
{
    ValidateIssuer = true, ValidateAudience = true,
    ValidateLifetime = true, ValidateIssuerSigningKey = true,
    ClockSkew = TimeSpan.FromSeconds(30),
    NameClaimType = "sub"
};
```

`MapInboundClaims = false` matters: the default mapping rewrites `sub` to a long URI claim type and
quietly breaks every `User.FindFirst("sub")` lookup.

`ITokenService` lives in `Application/Auth/Ports` and is implemented with
`Microsoft.IdentityModel.JsonWebTokens.JsonWebTokenHandler` — the modern handler.
`JwtSecurityTokenHandler` is legacy and carries the claim-mapping behaviour above.

### 3.2 Cookie issuance

Cookie expiry is set from the token's `exp` so the two cannot drift. Full cookie and transport
policy, including HSTS and security headers, is in
[31](../31-security-rate-limiting-and-cors/spec.md) §3.

### 3.3 Active-user confirmation

Step 5 confirms the user is still active. A literal database read on every authenticated request
would put a query on the hot path of every endpoint, so it is served by `IUserStatusCache` — a
`HybridCache` entry keyed by user id with a 30-second TTL, invalidated eagerly when a user is
blocked or deleted ([07](../07-user-deletion/spec.md) §3).

The bounded window is a deliberate trade; see §7.

## 4. Token refresh

**Endpoint:** `POST /api/v1/auth/refresh`. **Input:** the `refresh_token` cookie.

1. Verify the cookie is present and the signature is valid.
2. Verify `typ` is `refresh` and `exp` is valid.
3. Generate a new access token (15 min) and a new refresh token (30 days).
4. Return `200` with `Set-Cookie` headers for both.

No database lookup is performed, per §2.1. Missing, malformed or expired token returns `401`.

## 5. Logout

**Endpoint:** `POST /api/v1/auth/logout`.

Sets expired clear headers for both cookies — with **matching** `Path` and `Domain` attributes, or
the browser keeps the original cookie — and returns `200` unconditionally. The client performs any
redirect.

## 6. Audit and logging

- `auth.token.rejected` with a reason (`missing`, `signature`, `expired`), on failed validation.
- Access attempts to protected endpoints are covered by request logging
  ([27](../27-observability-and-audit-logging/spec.md) §2).
- **Never log** raw JWT strings, signing keys, signature details or token payloads.

## 7. Open decision: revocation

Two other documents need to end a session: deletion ([07](../07-user-deletion/spec.md)) must stop a
deleted user from continuing, and password reset ([20](../20-password-management/spec.md)) must
invalidate previously issued refresh tokens. The stateless rule in §2.1 forbids the state that
would make that possible.

| Option | Effect | Cost |
| --- | --- | --- |
| **A — accept the gap** | Access tokens die within 30 s via the status cache; refresh keeps minting new pairs for up to 30 days | A deleted user retains API access. Not acceptable. |
| **B — check on refresh only** (proposed) | One cached status / `pwd_at` read during `POST /auth/refresh`; deleted, blocked or password-reset sessions cannot rotate, so they die within 15 minutes | Deviates from §4 step 2 on one low-frequency endpoint |
| **C — token version on every request** | Immediate | A lookup per request — a far larger deviation than B |

**Proposed: option B.** Refresh runs roughly once per 15 minutes per session, so a cached read
there is cheap, and it is the narrowest change that makes deletion and password reset mean
anything. This needs explicit sign-off because §2.1 is marked as a hard constraint.

## 8. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 04.1 | `JwtBearer` with cookie extraction | Reuses the framework's validated token pipeline while keeping the cookie transport | Cookie authentication scheme (server-side ticket store, contradicts §2.1); custom middleware (reimplements validation) |
| 04.2 | HS256 with a 256-bit key | One service signs and validates; symmetric is simpler and faster. Moving to RS256 is a configuration and key-provisioning change, not a code change | RS256 from day one (key management cost with no current consumer) |
| 04.3 | `typ` claim separating access and refresh | Without it the two are interchangeable, turning a stolen access token into a 30-day session | Relying on the cookie name, which is attacker-controlled input |
| 04.4 | `MapInboundClaims = false` | Avoids the silent `sub` rewrite that breaks claim lookups | Default mapping plus URI constants everywhere |
| 04.5 | Active-user check via a 30-second cache | Honours the intent of §3 step 5 without a query per request | A literal per-request read; no check at all |
| 04.6 | Refresh cookie scoped to `/api/v1/auth`, `SameSite=Strict` | Limits where the long-lived credential travels | Sending it on every request |
| 04.7 | No refresh-token reuse detection | It requires exactly the state §2.1 forbids. Recorded as an accepted risk, not an oversight | A `jti` denylist |

## 9. Risks and open points

- **No revocation at all under option A.** A stolen refresh token is valid for 30 days with no
  server-side remedy. This follows directly from §2.1 and is worth re-examining before
  implementation rather than after.
- **The §7 decision blocks two features.** Resolve it once; both deletion and password reset
  depend on the same mechanism.
- **Clock skew is 30 seconds.** Shorter values cause spurious `401`s on under-synchronised hosts.

## 10. Acceptance criteria

1. A request with a valid `access_token` cookie reaches the controller with `sub` populated.
2. Missing, malformed, expired or wrongly signed tokens return `401`.
3. An access token presented in the refresh cookie is rejected (`typ` mismatch).
4. `POST /auth/refresh` returns a new pair and performs no database query under option A.
5. A blocked user's requests fail within 30 seconds of being blocked.
6. `POST /auth/logout` clears both cookies such that a subsequent request is unauthenticated.
7. No log line in any test run contains a raw JWT.
