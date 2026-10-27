# Feature: Admin User List & Search

## 1. Overview

- **Purpose:** Provide administrative interfaces with a paginated, filterable, and searchable list of registered application users.
- **Target Roles:**
  - **Admin:** Granted access via explicit RBAC permission (`users.list.admin`).
  - **Other Roles:** Access strictly prohibited (`403 Forbidden`).
- **Dependencies:** Access JWT (Cookie), RBAC Module, Database Search Indexes, Supabase avatar URL resolution.

## 2. Technical Contract

### 2.1 User List Endpoint

- **Endpoint:** `GET /admin/users`
- **Query Parameters:**
  - `cursor` (string, optional): Pagination cursor for deterministic page loading.
  - `limit` (number, optional, default: 20, max: 100): Number of items per page.
  - `q` (string, optional): Search query matching `email`, `id`, or name fields.
  - `status` (enum, optional): Filter by user status (`"active"` | `"blocked"` | `"deleted"`).
  - `sort` (enum, optional, default: `"created_at"`): Sort field (`"created_at"` | `"last_login"` | `"email"`).
  - `order` (enum, optional, default: `"desc"`): Sort direction (`"asc"` | `"desc"`).

- **Access Control:**
  - Validate `access_token` JWT.
  - Enforce `users.list.admin` RBAC permission check.

- **Response Payload Structure:**
  ```json
  {
    "items": [
      {
        "id": "string",
        "email": "string",
        "photo": "string | null",
        "createdAt": "ISO-8601 string",
        "status": "string",
        "lastLoginAt": "ISO-8601 string | null"
      }
    ],
    "nextCursor": "string | null"
  }
  ```
  `photo` is the public Supabase URL resolved from the avatar storage path stored in the user's `photo` column, or `null` when no avatar is set. It is never an externally supplied URL, and the raw storage path is not exposed.

## 3. Error Handling & Limits

- `400 Bad Request`: Invalid pagination cursor, out-of-range `limit`, or invalid sort/filter values.
- `401 Unauthorized`: Missing or invalid JWT.
- `403 Forbidden`: Non-admin caller attempting access.
- `429 Too Many Requests`: Rate limit exceeded for administrative search endpoints.

## 4. Audit & Logging

- Log list queries: `actorUserId`, applied search/filter parameter names, response status code, and total returned record count.
- Exclude raw full-text search parameters from logs if they risk leaking PII.

## 5. Non-Functional & Security Requirements

- **Sensitive Data Exposure Prevention:** Response objects must strictly omit sensitive internal attributes (password hashes, secrets, internal system flags, or security tokens).
- **Performance:** Mandatory cursor-based pagination and DB index optimization on sort/search fields (`created_at`, `status`, `email`).
