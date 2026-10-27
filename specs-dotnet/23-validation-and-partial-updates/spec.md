# 23 — Validation & Partial Updates

> Covers the mechanism behind the writable-field rule in [16](../16-user-write-hardening/spec.md):
> a request carrying a field the caller may not write must be **rejected with `403`**, not silently
> dropped. That requires knowing which fields were *present* in the JSON body, which
> `System.Text.Json` does not report by default.

## 1. Overview

- **Purpose:** Request validation pipeline, the presence-aware partial update type, and the
  default-deny writable-field policy shared by specs 06, 15, 16 and 17.
- **Dependencies:** FluentValidation, `System.Text.Json`.

## 2. Validation pipeline

Three distinct stages, in this order. Keeping them separate is what makes the status codes in the
specs come out right.

| Stage | Checks | Failure |
| --- | --- | --- |
| 1. Binding | JSON is well-formed; types match; no unknown properties | `400`, `malformed_body` / `unknown_field` |
| 2. Syntactic validation | FluentValidation: formats, lengths, ranges, enum membership | `400`, `validation_failed` with per-field `errors` |
| 3. Authorization of fields | Writable-field policy for this caller (§4) | `403`, `field_not_writable` |

Stage 3 runs in the application service, not in a filter, because it needs the caller's resolved
permissions and the target user's identity.

### 2.1 Binding

- `JsonSerializerOptions.UnmappedMemberHandling = JsonUnmappedMemberHandling.Disallow` on request
  bodies, so a typo or an injected field is a `400` rather than a silent no-op.
- `PropertyNameCaseInsensitive = false`, camelCase naming policy.
- `[ApiController]`'s automatic 400 model-state response is **suppressed**
  (`SuppressModelStateInvalidFilter = true`) so every error body comes from the one handler in
  [24](../24-error-handling/spec.md).

### 2.2 FluentValidation

- One validator per request DTO, in `FileConverter.Api/<Feature>/Validators/`, registered with
  `AddValidatorsFromAssemblyContaining<Program>()`.
- Invoked by a single `IAsyncActionFilter` (`ValidationFilter`) rather than per-controller code.
- Validators are pure: no database access. Uniqueness (`409` on taken email) and existence (`404`,
  `400` on unknown `roleId`) are application-service concerns, not validator concerns — they need
  to run inside the same transaction as the write.
- Shared rules live in extension methods: `.ValidEmail()`, `.StrongPassword()` (min 8 per [16](../16-user-write-hardening/spec.md)
  §2.3 and [20](../20-password-management/spec.md)), `.ValidOtp()` (exactly 6 digits), `.PageLimit()` (1–100).

## 3. `Optional<T>`: presence-aware fields

```csharp
public readonly struct Optional<T>
{
    public bool IsSet { get; }
    public T? Value { get; }
    public static Optional<T> Unset => default;
}
```

- A custom `JsonConverterFactory` sets `IsSet = true` for any property present in the payload,
  including one explicitly sent as `null`.
- PATCH DTOs use it for every field:

```csharp
public sealed record UpdateUserRequest
{
    public Optional<string> Email { get; init; }
    public Optional<UserStatus> Status { get; init; }
    public Optional<bool> IsVerified { get; init; }
}
```

- `request.PresentFields()` (source-generated or reflection-cached) returns the set of property
  names where `IsSet` is true. That set is the input to the writable-field policy and to the audit
  log, which records **field names only, never values** ([06](../06-user-profile-update/spec.md)).
- OpenAPI: a schema transformer unwraps `Optional<T>` to `T` with `nullable` as appropriate, so
  generated clients see a normal optional field — see [28](../28-openapi-and-versioning/spec.md) §3.

## 4. Writable-field policy (default-deny)

From [16](../16-user-write-hardening/spec.md), with role resolved by permission and never by role name.

```csharp
public enum UserWriteScope { SelfPatch, AdminPatch, AdminCreate }

public static class UserWritePolicy
{
    public static IReadOnlySet<string> Allowed(UserWriteScope scope);
}
```

| Field | Self PATCH | Admin PATCH | Admin POST |
| ---  | --- | --- | --- | 
| `email` | no (use email-change flow) | yes | yes, required |
| `status` | no | yes | optional, default `pending` |
| `isVerified` | no | yes | required |
| `password` | no | no | required |
| `roleId` | no | no (use [17](../17-user-role-assignment/spec.md)) | optional, default `USER` |

Evaluation:

1. Compute `present = request.PresentFields()`.
2. Compute `allowed = UserWritePolicy.Allowed(scope)` where scope comes from the caller's
   permissions (`users:update` / `users:create`), not their role name.
3. `forbidden = present.Except(allowed)`. If non-empty: log the **names** at `warn`, throw
   `ForbiddenException("field_not_writable", forbidden)` -> `403`.
4. Apply only `present` fields.

The same `Allowed` sets are the test oracle in [29](../29-testing-strategy/spec.md): one test per
row of the table above, asserting both directions.

### 4.1 Read-side counterpart

Field *visibility* (specs 05 and 15) uses the mirror-image construct — `SELF_PROFILE_FIELDS`,
`ADMIN_PROFILE_FIELDS`, `PUBLIC_PROFILE_FIELDS` — but is implemented with concrete response DTOs
rather than dynamic projection, for OpenAPI typing. See [05](../05-user-profile-view/spec.md) §3.

## 5. Query and form validation

- Query parameters bind to a record with `[FromQuery]` and are validated by the same filter. Out-of
  range `limit` (>100) is `400`, matching [08](../08-admin-user-list/spec.md).
- Enum query parameters (`status`, `sort`, `order`, `type`) bind as strings and are validated
  against the allowed set, so an unknown value is a clean `400` instead of a binding failure.
- Multipart conversion options (`width` 1–1920, `height` 1–1080, `quality` 1–100 per [18](../18-api-contract/spec.md))
  are validated by the same mechanism after being read from the multipart section.

## 6. Decisions

| # | Decision | Rationale | Rejected |
| ---  | --- | --- | --- | 
| 23.1 | FluentValidation over DataAnnotations | Conditional and cross-field rules (OTP vs magic-link payload shapes, admin vs self) are unreadable as attributes; closest in spirit to class-validator | DataAnnotations; `MiniValidation` |
| 23.2 | Custom `Optional<T>` | The `403`-not-drop requirement makes presence detection mandatory. `Optional<T>` keeps DTOs strongly typed and validators normal | `JsonElement`/`JsonObject` bodies (untyped, no OpenAPI schema); JSON Patch RFC 6902 (changes the public contract); nullable-means-absent (cannot express "set to null") |
| 23.3 | `UnmappedMemberHandling.Disallow` on request bodies | Turns typos and injected privilege fields into `400` instead of silence; complements the `403` policy | Default ignore |
| 23.4 | Writable-field policy in `Domain`, enforced in `Application` | It is a security rule, so it must be unit-testable without HTTP and impossible to bypass from a controller | Enforcing in an action filter (no access to target identity or permissions) |
| 23.5 | Suppress `[ApiController]` automatic 400 | One error shape from one place — [24](../24-error-handling/spec.md) | Two error shapes depending on where validation failed |
| 23.6 | Validators contain no I/O | Keeps uniqueness checks inside the write transaction, where they are actually race-free | DB-backed validators (TOCTOU between validation and write) |

## 7. Risks

- **`Optional<T>` ergonomics.** It is viral across PATCH DTOs, mappers and tests. Confine it to
  request DTOs; application services receive a resolved command object with plain values plus the
  present-field set.
- **OpenAPI fidelity.** The `Optional<T>` schema transformer must be covered by a test that
  inspects the generated document, or generated clients will silently acquire a wrapper type.
