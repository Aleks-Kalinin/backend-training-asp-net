# Claude Execution Prompt: Supabase-Only User Avatars

Implement the user-avatar changes defined in [`spec.md`](./spec.md). Treat that spec as the source of truth and read the repository's `Agents.md`, `CLAUDE.md`, and relevant existing code before editing.

## Required outcome

- Supabase-hosted uploads through the existing authenticated backend `PUT /api/v1/users/:id/photo` endpoint are the only way to set an avatar. The frontend sends the image to this backend; do not implement direct frontend-to-Supabase uploads.
- Use the `users.photo` column as the only persisted avatar reference: it stores the bucket-relative storage path (`null` on registration). Remove the `avatarStoragePath` column.
- Remove all request DTO support for setting a photo URL. Profile responses continue to include `photo`, resolved from the stored path to the avatar's public Supabase URL or `null`.
- Preserve authorization, image validation/processing, replacement and removal semantics, retryable cleanup, user-deletion cleanup, and existing behavior for unrelated profile fields.
- Do not expose the Supabase service-role key or the raw stored avatar path in public responses.

## Implementation guidance

1. Trace all uses of `photo` and `avatarStoragePath` across entities, domain types, repository ports/adapters, services, DTOs, mappers, field-visibility policies, controllers, tests, migrations, and documentation. Update every relevant surface consistently.
2. Add a new TypeORM migration that copies `users.avatarStoragePath` into `users.photo` and drops `avatarStoragePath`; do not rewrite an existing migration. Legacy URL values remaining in `photo` are intentionally cleared. Do not fetch or import arbitrary external URLs. Make the rollback restore `avatarStoragePath` from `photo` if that follows repository migration conventions.
3. Remove `photo` from user creation and profile update inputs and update Swagger/API documentation. Responses expose `photo` as the public URL resolved from the stored path, including profile and admin-list responses.
4. Ensure avatar upload/removal update only the stored `photo` path and unrelated profile updates leave it unchanged; remove obsolete URL-replacement cleanup logic and tests, but retain replacement, rollback, retry, and account-deletion cleanup behavior.
5. Update the README and any directly related specs so they do not describe URL-based photo PATCH behavior. Keep configuration examples free of real credentials.
6. Add or adjust tests for the new request/schema contract, response URL resolution, no-avatar `null` behavior, upload and removal, replacement cleanup, unrelated profile updates, user deletion, and failures.

## Constraints

- Keep changes focused on this feature and follow the existing NestJS, Fastify, TypeORM, and test patterns.
- Do not add a new storage provider, signed-upload flow, frontend implementation, or unrelated API redesign.
- Do not change the avatar endpoint path, multipart field name, image limits, processing rules, storage key format, or public-read behavior unless required to satisfy the spec.
- Do not modify `.env` or expose/copy secrets into output, docs, tests, or committed files.
- Do not commit changes.

## Verification and handoff

Run the focused user/avatar tests, the repository's formatter, and a build or type-check. Fix failures caused by the changes. In your final response, summarize the schema/API behavior changes, list the validation commands and their outcomes, and call out clearly that the migration discards any legacy URL values.
