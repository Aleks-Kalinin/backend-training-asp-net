# 31 — Security, Rate Limiting & CORS

> Rate-limit policies, cross-origin configuration, cookie and transport policy, input size limits,
> and the CSRF defence this service requires.

## 1. Rate limiting

Built-in `Microsoft.AspNetCore.RateLimiting`. Policies are named, declared centrally, and applied
with `[EnableRateLimiting("policy")]`.

Partition key: authenticated user id when present, otherwise the client IP resolved through
`ForwardedHeadersOptions` with a configured list of known proxies. An unconfigured forwarded-header
setup makes IP partitioning trivially spoofable, so `KnownProxies`/`KnownNetworks` is required
configuration in any environment behind a load balancer.

| Policy | Limit | Applied to | Source |
| ---  | --- | --- | --- | 
| `auth-register` | 5 / 10 min per IP, 3 / h per email | `POST /auth/signup` | [01](../01-registration/spec.md) |
| `auth-login` | 10 / 5 min per IP, 5 / 15 min per email | `POST /auth/login` | [03](../03-authentication/spec.md) |
| `verification-confirm` | 5 attempts per attempt record (enforced in domain, not middleware) | verify endpoints | [01](../01-registration/spec.md) |
| `verification-resend` | 1 / 60 s, 3 per attempt | resend endpoints | [01](../01-registration/spec.md), [20](../20-password-management/spec.md) |
| `password-reset` | 5 / 10 s per IP, 1 / 60 s per email | `POST /auth/password-reset` | [20](../20-password-management/spec.md) |
| `profile-read` | 120 / min per user | `GET /users/{id}`, `/auth/me` | [05](../05-user-profile-view/spec.md) |
| `profile-write` | 20 / min per user | `PATCH /users/{id}` | [06](../06-user-profile-update/spec.md) |
| `admin-search` | 60 / min per user | `GET /admin/users`, history endpoints | [08](../08-admin-user-list/spec.md), [11](../11-transformation-history/spec.md) |
| `conversion` | 30 / min per user | `POST /convert`, `POST /images/convert` | [09](../09-data-file-transformation/spec.md), [10](../10-image-transformation/spec.md) |
| `upload` | 10 / min per user | avatar upload/remove | [13](../13-avatar-upload/spec.md) |
| `download` | 60 / min per user | stored-file downloads | [12](../12-transformation-storage/spec.md) |
| `deletion` | 3 / h per user | deletion request | [07](../07-user-deletion/spec.md) |

Rejection returns `429` with a `ProblemDetails` body (code `rate_limited`) and a `Retry-After`
header, via `OnRejected`.

**Attempt counters are not rate limits.** The "max 5 OTP attempts" rules in specs 01/03/06/20 are
per-attempt-record state in the database, because they must survive process restarts and apply
across IPs. The middleware limits request *volume*; the domain limits *attempts*. Both exist.

### 1.1 Multi-instance

The in-memory limiter partitions per process, so N instances means N times the limit. For
multi-instance deployments, back the limiter with Redis (a custom `RateLimiterPolicy` over
`StackExchange.Redis`). Single-instance is the documented default; the startup log states which is
active, as with the RBAC notifier ([26](../26-configuration-and-settings/spec.md) §5).

## 2. CORS

- Origins come from configuration (`Cors:Origins`), never hardcoded — [18](../18-api-contract/spec.md). Startup fails
  on a malformed entry ([26](../26-configuration-and-settings/spec.md) §2.1).
- `AllowCredentials()` is required, since auth rides on cookies. That makes a wildcard origin
  illegal, which is the correct constraint rather than an inconvenience.
- Exposed headers: `Content-Disposition` (the download endpoints need it for the filename).
- Default development origin: `http://localhost:5173`.

## 3. Cookies & transport

| Setting | Value | Note |
| --- | --- | --- |
| `HttpOnly` | `true` | [04](../04-authorization-session/spec.md) |
| `Secure` | `true` | relaxed only when the environment is `Development` over plain HTTP |
| `SameSite` | `Strict` for `refresh_token`, `Lax` for `access_token` | see §4 |
| `Path` | `/api/v1/auth` for `refresh_token`, `/` for `access_token` | the refresh cookie is never sent to endpoints that do not need it |
| Expiry | matches the token `exp` | [04](../04-authorization-session/spec.md) |

Plus HSTS in production, `UseHttpsRedirection`, and response headers
`X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`,
`Content-Security-Policy: default-src 'none'; frame-ancestors 'none'` (an API serves no documents;
Scalar's UI route gets its own relaxed policy).

## 4. CSRF

Authentication rides on a cookie that the browser attaches automatically, and there is no bearer
token or custom-header requirement, so any state-changing endpoint is reachable from a cross-site
form post unless something stops it. `SameSite=Lax` blocks cross-site POSTs in current browsers,
but it is a single defence, it does not cover same-site subdomains, and it is a browser behaviour
rather than a server guarantee.

Required here:

1. `SameSite=Strict` on `refresh_token` — token rotation is the highest-value target.
2. A **required custom header** (`X-Requested-With: fileconverter`) on every state-changing request.
   A cross-origin form post cannot set a custom header, and a cross-origin XHR that tries triggers
   a preflight that CORS will refuse. Missing header -> `403`, code `csrf_check_failed`.
3. Option 2 is the lightweight form. If the frontend can carry a token, switch to ASP.NET Core
   antiforgery (`AddAntiforgery` + a `/auth/csrf` endpoint issuing the token). Decided per
   frontend capability; the header check is the default because it costs the client one line.

This is listed as open decision 3 in the [README](../README.md) §5, because it is a requirement
added here rather than one inherited from the original service. Shipping cookie authentication
with no CSRF defence at all is not a defensible default.

## 5. Input-size limits

| Scope | Limit | Mechanism |
| --- | --- | --- |
| Global request body | 1 MiB | Kestrel `MaxRequestBodySize` |
| Avatar upload | 5 MiB | `[RequestSizeLimit]` on the action — [13](../13-avatar-upload/spec.md) |
| Conversion upload | per-format admin setting | checked while streaming, before buffering — [09](../09-data-file-transformation/spec.md) |
| Multipart headers | default | — |

The per-format limit must be enforced **during** the stream read, not after buffering the file;
otherwise the limit does not protect memory, which is the only reason it exists. Exceeding it is
`413`, code `file_too_large`.

## 6. Format-specific hardening

Pointers to the documents that own each rule:

- XML external entities and DTDs disabled — [09](../09-data-file-transformation/spec.md) §4.
- Decompression-bomb pixel caps — [10](../10-image-transformation/spec.md) §4.
- SVG sanitisation — [10](../10-image-transformation/spec.md) §4.
- Storage keys never exposed; downloads proxied — [12](../12-transformation-storage/spec.md) §3.
- Supabase service-role key server-side only — [13](../13-avatar-upload/spec.md) §5.

## 7. Decisions

| # | Decision | Rationale | Rejected |
| ---  | --- | --- | --- | 
| 31.1 | Built-in rate limiter | In-box, composable, no dependency; policy-per-endpoint matches the per-endpoint limits each feature states | `AspNetCoreRateLimit` (third party, superseded) |
| 31.2 | Named policies in one file | The limits are scattered across twenty specs; one table is reviewable | `[EnableRateLimiting]` with inline options per controller |
| 31.3 | Attempt counters in the database, separate from the limiter | They must survive restarts and cross IPs — a memory-partitioned limiter cannot express "5 attempts for this OTP record" | Using the limiter for both |
| 31.4 | Custom-header CSRF check by default | Cheapest defence that actually blocks cross-site form posts; no token plumbing | Relying on `SameSite` alone (insufficient); full antiforgery tokens (more frontend work — available as the upgrade) |
| 31.5 | `refresh_token` scoped to the auth path | Limits exposure of the long-lived credential | Sending it on every request |
| 31.6 | Forwarded-headers proxy allowlist is mandatory config | Without it, IP-partitioned limits are bypassed with one header | Trusting `X-Forwarded-For` blindly |
| 31.7 | Size limits enforced mid-stream | A limit applied after buffering protects nothing | `IFormFile` with a post-hoc length check |

## 8. Risks

- **Per-instance limits.** Documented default is single-instance. Scaling out without the Redis
  limiter silently multiplies every limit — a deployment checklist item, not a code concern.
- **CSRF header and non-browser clients.** Any CLI or service client must send the header too.
  Document it in the API description.
- **`SameSite=Strict` on refresh** breaks a refresh triggered immediately after a cross-site
  navigation into the app. Acceptable: the user re-logs in once. Revisit if it shows up in support.
