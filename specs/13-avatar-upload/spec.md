# Feature: Supabase-Hosted User Avatars

## 1. Overview

- **Purpose:** Let a user upload, replace, or remove their profile avatar. Supabase Storage is the only supported avatar source; arbitrary external photo URLs are not supported.
- **Upload path:** The frontend sends the image to this backend's authenticated avatar endpoint. The backend validates and processes it, then writes it to Supabase Storage with a server-side service-role key. The frontend must never receive Supabase privileged credentials or upload directly to Supabase.
- **Target Roles:**
  - **Self:** May manage their own avatar.
  - **Admin:** May manage an avatar for another user when authorized to update that user's profile.
- **Dependencies:** Authentication and RBAC modules, `@fastify/multipart`, Sharp image processing, Supabase Storage, and the user repository.

## 2. Data & API Contract

- The nullable `photo` column on the user record is the only stored avatar reference. It holds the bucket-relative object key of the uploaded avatar, for example `{userId}/{uuid}.webp`, and is `null` until an avatar is uploaded. There is no separate `avatarStoragePath` column.
- Remove all request-body support for setting `photo`. The user cannot set or replace an avatar using a URL through profile creation or update.
- The API response field `photo` is the **public URL resolved** from the stored path, or `null` when no uploaded avatar exists. Do not persist this URL.
- Never expose the raw stored path in public response DTOs; responses expose only the resolved URL.
- Existing legacy `photo` URL values are intentionally discarded by the migration. Do not attempt to fetch or import remote URLs as part of the migration.
- Do not persist image bytes or absolute local paths. Do not log object contents, credentials, raw client filenames, or signed URLs.

## 3. Technical Contract

### 3.1 Upload Avatar

- **Endpoint:** `PUT /users/{userId}/photo`
- **Content-Type:** `multipart/form-data`
- **Input:** `file` (binary, required): One JPEG, PNG, or WebP image.
- **Access:** Authenticated user. The caller may target their own account; an administrator may target another account only with the existing user-update permission.
- **Validation Rules:**
  1. Require exactly one non-empty image file.
  2. Limit the uploaded file to 5 MiB and reject unsupported media types with `415 Unsupported Media Type`.
  3. Verify the file's actual image format from its contents; do not trust the client-provided MIME type or filename.
  4. Decode and re-encode with Sharp to WebP, strip metadata, and enforce a maximum output dimension of 1024 by 1024 pixels while preserving aspect ratio.
  5. Reject corrupt or invalid image data. Apply an appropriate pixel limit to protect against decompression bombs.
- **Execution Logic:**
  1. Authenticate and authorize the caller, then verify the target user exists.
  2. Validate and process the file before storage.
  3. Upload to the configured Supabase bucket using a server-generated key and `image/webp` content type. Use the backend-only service-role credential.
  4. Update the stored `photo` path only after the upload succeeds.
  5. If updating the user record fails, attempt to delete the newly uploaded object and propagate the original failure.
  6. After the database update succeeds, remove the previous avatar object, if any. Cleanup failure must not lose the new avatar reference and must be observable and retryable through the existing cleanup mechanism.
  7. Return the standard profile response with `photo` set to the computed public URL.

### 3.2 Remove Avatar

- **Endpoint:** `DELETE /users/{userId}/photo`
- **Access:** Same self/admin authorization as the upload endpoint.
- **Execution Logic:**
  1. Authenticate and authorize the caller, then verify the target user exists.
  2. Clear the stored `photo` path.
  3. Remove the previous uploaded object, if any. If storage deletion fails, record or log it through the existing observable, retryable cleanup mechanism.
  4. Return the updated profile with `photo: null`.
- Removing an avatar is idempotent: deleting when no avatar is set succeeds with `photo: null`.

### 3.3 User Creation and Profile Update

- Preserve the existing user creation and profile update endpoints for their other supported fields.
- Remove `photo` from user creation and profile update request DTOs and validation. Clients cannot set an avatar by supplying a URL.
- A newly created user has no avatar (the `photo` column is `null`); profile responses expose `photo: null`.
- Avatar changes happen only through the upload and remove endpoints. Updating unrelated profile fields must not change or delete the current avatar.

## 4. Schema Migration

- Add a new TypeORM migration; do not edit or rewrite migrations that may already have run in deployed environments.
- The migration copies existing stored avatar paths from `users.avatarStoragePath` into `users.photo`, then drops `avatarStoragePath`, preserving the existing avatar cleanup task schema.
- Any `photo` value that is not an uploaded-avatar path owned by that user (`{userId}/{uuid}.webp`) is a legacy external URL and is set to `null`. Do not fetch external URLs or silently convert them to stored avatars.
- Make the migration and its rollback consistent with repository migration conventions. The rollback may restore `avatarStoragePath` from `photo`, but cannot reconstruct discarded URL values.
- Keep TypeORM entity definitions, domain/repository types, and schema metadata in agreement with the migrated schema.

## 5. Configuration & Operations

- Configure the Supabase project URL, service-role key, and bucket name through server-side environment variables. Add variable names (not secrets) to `.env.example`; do not commit real credentials.
- Fail application startup clearly when Supabase configuration is required but missing or invalid. Never fall back silently to local disk or expose the service-role key to clients.
- Configure the bucket to allow public object reads if profile responses use public URLs, accept only the stored WebP content type, and enforce the upload size limit. Do not grant public anonymous upload or delete access.
- Keep avatar upload limits separate from global multipart limits where possible, so changing this endpoint does not unintentionally change conversion upload behavior.
- Document the configured bucket/project in deployment configuration and monitor quota or storage errors. Free-plan quotas and inactivity behavior are provider constraints, not application guarantees.

## 6. Error Handling & HTTP Statuses

- `200 OK`: Avatar uploaded, replaced, or removed successfully.
- `400 Bad Request`: Missing file, multiple files, empty file, or invalid image data.
- `401 Unauthorized`: Missing or invalid authentication.
- `403 Forbidden`: Caller lacks permission to update the target user's profile.
- `404 Not Found`: Target user does not exist.
- `413 Payload Too Large`: Upload exceeds the avatar size limit.
- `415 Unsupported Media Type`: Unsupported or mismatched image format.
- `500 Internal Server Error`: Image processing, Supabase upload, or database operation failed. Do not return internal storage credentials or private details.

## 7. User Deletion & Lifecycle

- User account deletion must remove the avatar object from Supabase Storage in addition to deleting the database reference.
- Object cleanup must be observable and retryable. Follow the established deletion flow and cleanup task behavior.
- Use unique object keys for each replacement rather than overwriting a shared key, so caches cannot serve stale content and failed replacements do not destroy the current avatar.

## 8. Audit, Security & Performance

- Log avatar upload, replacement, removal, and cleanup outcomes with actor ID, target user ID, operation, status, and processed byte size.
- Enforce the same ownership and RBAC protections used by profile updates to prevent IDOR.
- Use the service-role key only in the backend. The frontend uploads file bytes only to this backend endpoint.
- Public read access is intentional for profile photos. If avatars later need to be private, use an authorized streaming endpoint or short-lived signed URLs without exposing the service-role key.
- Apply request throttling to uploads and removals.
- Avoid buffering files beyond the configured size limit; reject oversized requests while streaming where supported.

## 9. Acceptance Criteria

1. An authorized user can upload a valid image through the backend endpoint and receives a profile whose computed `photo` is the new public avatar URL.
2. Unsupported, corrupt, mismatched, and oversized uploads are rejected with the specified HTTP status.
3. Unauthenticated callers and callers lacking self/admin authorization cannot upload or remove a user's avatar.
4. Replacing an avatar stores a new object and cleans up the prior object without losing the active avatar on database failure.
5. Avatar removal and account deletion clear the stored `photo` path and clean up the corresponding storage object.
6. User creation and profile update requests cannot set `photo` to an external URL; unrelated profile updates preserve the uploaded avatar.
7. The users table stores the avatar path in `photo` and no longer has an `avatarStoragePath` column, and the user entity, repository, DTOs, mappings, migrations, and tests match that schema.
8. Profile and admin-list response contracts expose `photo` as the resolved public URL or `null`; the raw storage path and service-role key are not exposed.
9. Automated tests cover validation, authorization, storage/database failures, replacement cleanup, URL-field rejection/removal, user deletion cleanup, and profile response output.
