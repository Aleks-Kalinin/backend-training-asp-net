# Feature: Transformation History

## 1. Overview

- **Purpose:** Provide users and administrators access to historical execution logs for data file and image transformations.
- **Target Roles:**
  - **Self:** Can view only their own transformation history logs.
  - **Admin:** Can view the transformation history for any specified user.
- **Dependencies:** Access JWT (Cookie), Unified Transformation Audit Store (Database), Cursor-based Pagination Engine.

## 2. Technical Contract

### 2.1 Own Transformation History Endpoint

- **Endpoint:** `GET /api/transformations/history`
- **Access:** Authenticated users (Self).
- **Query Parameters:**
  - `cursor` (string, optional): Cursor for pagination.
  - `limit` (number, optional, default: 20, max: 100): Page size limit.
  - `type` (enum, optional): Filter by transformation type (`"file"` | `"image"`).
  - `sourceFormat` (string, optional): Source format (`csv`, `json`, `xml`, `yaml`, `png`, `jpeg`, `svg`).
  - `targetFormat` (string, optional): Target format.
  - `status` (enum, optional): Result status (`"success"` | `"error"`).
  - `createdAtFrom` (ISO-8601 string, optional): Filter start timestamp.
  - `createdAtTo` (ISO-8601 string, optional): Filter end timestamp.

- **Execution Logic:**
  1. Authenticate caller via access JWT.
  2. Query transformation history store filtering strictly by `userId == req.user.id`.
  3. Apply optional query parameters (`type`, `sourceFormat`, `targetFormat`, `status`, date ranges).
  4. Return paginated dataset ordered by `createdAt` descending.

- **Response Payload (`200 OK`):**
  ```json
  {
    "items": [
      {
        "id": "string",
        "type": "file | image",
        "sourceFormat": "string",
        "targetFormat": "string",
        "status": "success | error",
        "fileSize": 1024,
        "durationMs": 145,
        "errorCode": "INVALID_SYNTAX",
        "createdAt": "2026-09-01T16:00:00.000Z"
      }
    ],
    "nextCursor": "string | null"
  }
  ```

### 2.2 Admin User Transformation History Endpoint

- **Endpoint:** `GET /admin/users/{userId}/transformations/history`
- **Access:** Admin only (requires `transformations.history.admin` RBAC permission).
- **Query Parameters:** Same as endpoint 2.1.
- **Response Payload:** Same as endpoint 2.1.
- **Execution Logic:**
  1. Authenticate caller and evaluate `transformations.history.admin` permission.
  2. Verify target `userId` existence (`404 Not Found` if missing).
  3. Query transformation history records where `userId == path.userId`.
  4. Return paginated dataset.

## 3. Error Handling & Edge Cases

- `400 Bad Request`: Invalid cursor format, out-of-range `limit`, or invalid filter values.
- `401 Unauthorized`: Missing or invalid JWT cookie.
- `403 Forbidden`: Non-admin caller attempting to access `/admin/users/{userId}/...` endpoints.
- `404 Not Found`: Target `userId` does not exist.
- `429 Too Many Requests`: Rate limit exceeded on history retrieval endpoints.

## 4. Audit & Logging

- Log history queries: `actorUserId`, `targetUserId` (for admin queries), applied search filter keys, response status code, and total returned item count.
- **Forbidden:** Never store or log raw file binary contents in history audit tables.

## 5. Non-Functional & Security Requirements

- **IDOR Protection:** Strict query parameter scoping ensuring non-admin users cannot access records belonging to other `userId`s.
- **Database Performance:** Mandatory composite database indices on (`userId`, `createdAt`), `type`, and `status` to maintain sub-millisecond cursor pagination over growing audit tables.
- **Data Retention:** Retain transformation history records for an admin-configurable TTL (e.g., 90 days default) before triggering automated database cleanup jobs.
