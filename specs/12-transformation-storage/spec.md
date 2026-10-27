# Feature: Transformation Result Storage & File Download

## 1. Overview
* **Purpose:** Allow users to opt-in to saving output transformation files to backend storage during file/image conversion. Saved files remain downloadable from transformation history until the retention period expires.
* **Target Roles:**
  * **Self:** Saves and downloads their own transformed files.
  * **Admin:** Can download saved transformation files for any specified user.
* **Dependencies:** Storage Abstraction Layer (Local FS, S3-compatible, or Firebase), Transformation History Database Engine, Automated Background Pruning Worker.

## 2. Technical Contract

### 2.1 Transformation Request Extension
* **Endpoints Modified:** `POST /api/convert` and `POST /api/images/convert`
* **Additional Input Parameter (`multipart/form-data`):**
  * `save` (boolean, optional, default: `false`): Flag requesting server-side storage of output file.

* **Execution Logic:**
  1. Execute standard file/image transformation pipeline.
  2. **If `save == true` and transformation succeeds:**
     * Persist output file to storage abstraction layer.
     * Generate unique `fileId`.
     * Attach `fileId` to the corresponding `TransformationHistoryItem` database entry.
     * Set file expiration timestamp synchronized with history retention TTL (e.g., 90 days).
  3. Stream converted file back to client immediately (*storage operation must not block or delay HTTP response stream*).
  4. **If `save == false`:** Record history entry without attaching a `fileId`.

### 2.2 Download Saved File Endpoint (Self)
* **Endpoint:** `GET /api/transformations/history/{itemId}/download`
* **Access:** Authenticated user (Self).
* **Validation & Execution Steps:**
  1. Authenticate caller via access JWT.
  2. Fetch `TransformationHistoryItem` by `itemId`.
  3. Enforce ownership validation (`req.user.id == history.userId`).
  4. Verify presence of `fileId` and check file TTL status.
  5. Fetch binary stream from storage abstraction layer by `fileId`.
  6. Return binary stream response.

* **Response Headers:**
  * `200 OK`: Streamed binary payload.
  * `Content-Type`: Matching file format MIME type.
  * `Content-Disposition`: `attachment; filename="<original_name>"`

### 2.3 Download Saved File Endpoint (Admin)
* **Endpoint:** `GET /admin/users/{userId}/transformations/history/{itemId}/download`
* **Access:** Admin only (requires `transformations.history.admin` RBAC permission).
* **Validation & Execution Steps:**
  1. Authenticate caller and verify `transformations.history.admin` permission.
  2. Verify target `userId` exists.
  3. Retrieve `TransformationHistoryItem` matching `itemId` and target `userId`.
  4. Verify non-expired `fileId` exists on the record.
  5. Stream stored file payload to response.

## 3. Error Handling & HTTP Statuses
* `401 Unauthorized`: Missing or invalid JWT cookie.
* `403 Forbidden`: IDOR attempt (requesting another user's file without admin privileges).
* `404 Not Found`: History item, target user, or stored file does not exist or has expired.
* `410 Gone`: (Optional) Explicit response indicating target file was purged by retention TTL.
* `500 Internal Server Error`: Storage provider failure (write/read error or storage service unavailable).

## 4. Audit & Logging
* Log storage operations: `userId`, `transformationId`, `fileId`, action (`save` | `download`), result status, file size, and execution duration.
* **Forbidden:** Never expose raw storage paths, internal bucket keys, or file binary contents in application logs.

## 5. Non-Functional & Security Requirements
* **IDOR & Direct Link Prevention:** Raw internal storage paths or bucket keys must never be exposed to clients; all access occurs via authorized application streaming proxies or short-lived signed URLs.
* **Asynchronous Persistence:** File storage writes should be decoupled or buffered so as not to increase transformation HTTP output latency.
* **Lifecycle & Retention Synchronization:** Background Cron/Worker processes must automatically delete physical storage assets simultaneously when corresponding history database records expire.
* **Storage Provider Abstraction:** File operations must be encapsulated behind a repository pattern interface to allow switching backends (Local, S3, MinIO, Firebase) without altering core domain logic.