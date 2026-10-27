# 13 — Supabase-Hosted User Avatars

## 1. Overview

- **Purpose:** Let a user upload, replace or remove their profile avatar. Supabase Storage is the
  only supported avatar source; arbitrary external photo URLs are not supported.
- **Upload path:** the frontend sends the image to this backend's authenticated avatar endpoint.
  The backend validates and processes it, then writes it to Supabase Storage using a server-side
  service-role key. **The frontend never receives Supabase credentials and never uploads directly
  to Supabase.**
- **Target roles:**
  - **Self** — may manage their own avatar.
  - **Admin** — may manage another user's avatar when authorised to update that user's profile.
- **Dependencies:** [04](../04-authorization-session/spec.md), [02](../02-rbac/spec.md),
  SkiaSharp for image processing, Supabase Storage,
  [25](../25-background-jobs/spec.md) (retryable cleanup),
  [26](../26-configuration-and-settings/spec.md) (configuration and startup validation).

## 2. Data and API contract

- The nullable `photo` column on the user record is the **only** stored avatar reference. It holds
  the bucket-relative object key, for example `{userId}/{uuid}.webp`, and is `null` until an avatar
  is uploaded. There is no separate storage-path column.
- No request body anywhere accepts a `photo` value. A user cannot set or replace an avatar by
  supplying a URL, through user creation or profile update.
- The API response field `photo` is the **public URL resolved** from the stored path, or `null`
  when no avatar exists. This URL is not persisted.
- The raw stored path is never exposed in a response DTO.
- Do not persist image bytes or absolute local paths. Do not log object contents, credentials, raw
  client filenames or signed URLs.

## 3. Endpoints

| Endpoint | Method | Auth | Rate limit policy |
| --- | --- | --- | --- |
| `/api/v1/users/{userId}/photo` | PUT, `multipart/form-data` | `[AllowSelf("userId")] [RequirePermission("users","update")]` | `upload` |
| `/api/v1/users/{userId}/photo` | DELETE | same | `upload` |

Both return the standard profile response ([05](../05-user-profile-view/spec.md) §3) with `photo`
set to the computed public URL, or `null` after removal.

### 3.1 Upload

**Input:** `file` — exactly one JPEG, PNG or WebP image.

**Validation rules:**

1. Require exactly one non-empty image file.
2. Limit the upload to 5 MiB; reject unsupported media types with `415`.
3. Verify the file's **actual** image format from its contents. Do not trust the client-provided
   MIME type or filename.
4. Decode and re-encode to WebP, strip metadata, and enforce a maximum output dimension of
   1024 by 1024 while preserving aspect ratio.
5. Reject corrupt or invalid image data. Apply a pixel limit to protect against decompression
   bombs ([10](../10-image-transformation/spec.md) §5).

**Execution:**

1. Authenticate and authorise the caller, then verify the target user exists.
2. Read the multipart section with `MultipartReader`, enforcing the 5 MiB limit mid-stream.
3. Validate and process the image before any storage call.
4. Upload to the configured Supabase bucket under a **server-generated key** with content type
   `image/webp`, using the backend-only service-role credential.
5. Update the stored `photo` path **only after** the upload succeeds.
6. If updating the user record fails, attempt to delete the newly uploaded object and propagate the
   original failure. The user keeps their previous avatar.
7. After the database update succeeds, remove the previous avatar object, if any, through the
   retryable cleanup job. **A cleanup failure must not lose the new avatar reference**, and must
   remain observable and retryable.
8. Return the profile with `photo` set to the computed public URL.

Each replacement uses a **unique object key** rather than overwriting a shared one, so caches
cannot serve stale content and a failed replacement cannot destroy the current avatar.

### 3.2 Removal

1. Authenticate and authorise, then verify the target user exists.
2. Clear the stored `photo` path.
3. Remove the previous object through the cleanup job. A storage deletion failure is recorded and
   retried, never silently dropped.
4. Return the updated profile with `photo: null`.

Removal is **idempotent**: deleting when no avatar is set succeeds and returns `photo: null`.

### 3.3 User creation and profile update

- User creation and profile update keep their other fields unchanged.
- `photo` is absent from their request DTOs and validation.
- A newly created user has no avatar; profile responses expose `photo: null`.
- Avatar changes happen **only** through the endpoints above. Updating unrelated profile fields
  must not change or delete the current avatar.

## 4. Storage adapter

```csharp
public interface IAvatarStorage
{
    Task UploadAsync(string key, Stream webp, CancellationToken ct);
    Task DeleteAsync(string key, CancellationToken ct);
    string PublicUrl(string key);
}
```

| Adapter | Use |
| --- | --- |
| `SupabaseAvatarStorage` | A typed `HttpClient` against the Storage REST API (`POST /storage/v1/object/{bucket}/{key}`, service-role key as a bearer token), registered through `IHttpClientFactory` with a retry on transient failures |
| `LocalAvatarStorage` | Writes under a configured directory and serves from a static route; permitted **only** in the `Test` and `E2E` environments ([19](../19-test-infrastructure/spec.md) §3) |

The selected driver is logged once at startup.

## 5. Configuration and operations

- The Supabase project URL, service-role key and bucket name come from server-side configuration
  ([26](../26-configuration-and-settings/spec.md) §2). Committed files carry variable names and
  placeholders, never real credentials.
- **Startup fails clearly** when Supabase configuration is required but missing or invalid. There
  is never a silent fallback to local disk, and the service-role key is never exposed to clients.
- The bucket allows public object reads (profile responses use public URLs), accepts only the
  stored WebP content type, and enforces the upload size limit. It grants **no** anonymous upload
  or delete access.
- Avatar upload limits are kept separate from the global multipart limits, so changing this
  endpoint does not alter conversion upload behaviour.
- Document the configured bucket and project in deployment configuration, and monitor quota and
  storage errors. Free-plan quotas and inactivity behaviour are provider constraints, not
  application guarantees.

## 6. Error handling

| Status | Code | Condition |
| --- | --- | --- |
| `200` | — | Avatar uploaded, replaced or removed successfully |
| `400` | `invalid_image` | Missing file, multiple files, empty file, or invalid image data |
| `401` | `not_authenticated` | Missing or invalid authentication |
| `403` | `permission_denied` | Caller may not update the target user's profile |
| `404` | `user_not_found` | Target user does not exist |
| `413` | `file_too_large` | Upload exceeds the 5 MiB avatar limit |
| `415` | `unsupported_media_type` | Unsupported or mismatched image format |
| `500` | `storage_unavailable` | Image processing, upload or database operation failed |

A `500` body never returns storage credentials or provider detail.

## 7. User deletion and lifecycle

- Account deletion removes the avatar object from storage in addition to clearing the database
  reference ([07](../07-user-deletion/spec.md) §3).
- Object cleanup is observable and retryable through the same job used by replacement and removal.

## 8. Audit, security and performance

- `avatar.operation` with actor id, target user id, operation, status and processed byte size, for
  upload, replacement, removal and cleanup outcomes.
- The same ownership and RBAC protections as profile updates, preventing IDOR.
- The service-role key is used **only** in the backend. The frontend sends file bytes only to this
  endpoint.
- Public read access is intentional for profile photos. If avatars later need to be private, use an
  authorised streaming endpoint — the shape in [12](../12-transformation-storage/spec.md) §4 —
  rather than exposing the service-role key.
- Throttle uploads and removals.
- Avoid buffering files beyond the configured size limit; reject oversized requests while
  streaming.

## 9. Decisions

| # | Decision | Rationale | Rejected |
| --- | --- | --- | --- |
| 13.1 | `HttpClient` against the Storage REST API | The entire surface is upload, delete and a URL template; a typed client keeps retry, timeout and lifetime inside `IHttpClientFactory` | A community Supabase SDK — an extra dependency and its own lifecycle for three operations |
| 13.2 | SkiaSharp for decode and WebP encode | The same engine as [10](../10-image-transformation/spec.md): one native dependency, one licence | ImageSharp (licence); Magick.NET (a second heavy native dependency) |
| 13.3 | A new key per replacement | Makes the upload, update and cleanup sequence recoverable, and defeats CDN staleness | Overwriting a stable key, where a failed replacement destroys the current avatar |
| 13.4 | Compensating delete on database failure | Upload and database write cannot share a transaction; this is the recoverable ordering | Leaving the orphaned object |
| 13.5 | Previous-object cleanup through a retryable job | Cleanup must be observable and retryable, and must never block the response | Inline best-effort delete |
| 13.6 | `photo` stores the key; responses show the URL | Persisting the URL breaks if the bucket or project moves | Persisting the public URL |
| 13.7 | Driver switch guarded by environment | Tests need local storage; production must never fall back silently. An explicit, guarded switch satisfies both | Always Supabase (untestable); an unguarded local driver |

## 10. Risks and open points

- **Public bucket reads are intentional**, which makes avatar URLs unauthenticated and permanent.
  Privacy would require an authorised streaming endpoint, not signed URLs with a shared key.
- **Provider quotas and inactivity pauses** are outside application control; monitor storage
  errors.
- **Admins can change another user's avatar** through `users:update`. Reasonable for moderation —
  make sure the audit event records both actor and target.
- **The compensating delete can itself fail**, leaving an orphan. It is logged and retried; after
  that the object leaks. A periodic sweep for keys not referenced by any user would close it.

## 11. Acceptance criteria

1. An authorised user uploads a valid image and receives a profile whose `photo` is the new public
   URL.
2. Unsupported, corrupt, mismatched and oversized uploads are rejected with `415`, `400`, `415` and
   `413` respectively.
3. Unauthenticated callers, and callers lacking self or admin authorisation, cannot upload or
   remove an avatar.
4. Replacing an avatar stores a new object and cleans up the prior one, and a database failure
   leaves the previous avatar intact.
5. Avatar removal and account deletion clear `photo` and delete the storage object.
6. User creation and profile update cannot set `photo`; unrelated profile updates preserve the
   avatar.
7. Responses expose `photo` as a resolved public URL or `null`; neither the raw path nor the
   service-role key appears in any response or log.
8. Starting with the local driver outside `Test`/`E2E` fails at startup.
9. Tests cover validation, authorisation, storage and database failures, replacement cleanup,
   URL-field rejection, deletion cleanup and profile response output.
