# Tasks: Client Attachments
**Spec:** `specs/client-attachments.md`
**Status:** pending

---

## Overview

Implements file attachment management for cards: a disk-based file storage service, four Nitro API endpoints (list types, upload, update metadata, delete), an `attachments` DB table, a `transformAttachment` transformer, a `transformCard` attachment-split update, shared Zod schemas, the `FileTypeEnum`, and a frontend upload component with image lightbox preview. Every mutation writes a `CardLog` entry and broadcasts a Centrifugo event on the `card#{id}` channel. Uploads that include an attachment alongside a comment link the attachment via `comments.attachmentId`.

Execution order:
1. **Wave 1 (API Agent)** — shared enum + schemas + types + transformer → file storage service → four API endpoints → `transformCard` split.
2. **Wave 2 (Frontend Agent)** — `use-attachment-api.ts` + `card.ts` store updates + `File.vue` + `attachment-list.vue` + `CommentInput.vue` flag (depends on Wave 1).
3. **Wave 3 (Testing Agent)** — unit tests for service, endpoints, transformers + e2e attachment flow (depends on Waves 1 & 2).

---

## Wave 1 — Data Layer (API Agent)

### Task 019-A: FileTypeEnum, Zod Schemas, Shared Types & Attachment Transformer

- **Agent:** API
- **Spec:** `specs/client-attachments.md`
- **Section:** Data Model → File Type Enum; Data Model → TypeScript Interfaces; Acceptance Criteria → AC-2 (validation), AC-3 (validation)
- **Acceptance Criteria:**
  - [ ] AC-1 (partial): `FileTypeEnum` const object is defined in `shared/enums/FileTypeEnum.ts` with values `Other=0, Image=1, Video=2, Audio=3, PDF=4, Excel=5` — exported as `as const`.
  - [ ] AC-2 (partial): `createAttachmentSchema` validates `name` (min 1, max 50 chars, required) and `type` (integer 0–5, required) in `shared/schemas/attachment.schema.ts`.
  - [ ] AC-3 (partial): `updateAttachmentSchema` is a Zod `.partial()` of `createAttachmentSchema` with a `.refine()` that rejects an empty body `{}` — returns `422` when no fields are provided.
  - [ ] `TransformedAttachment` interface is defined in `shared/types/attachment.types.ts` with fields: `id: number`, `card_id: number`, `name: string`, `path: string`, `type: number`.
  - [ ] `AttachmentModel` (full DB row) interface is defined in `shared/types/attachment.types.ts` with fields: `id`, `user_id`, `card_id`, `name`, `type`, `path`, `createdAt`, `updatedAt`.
  - [ ] `CreateAttachmentRequest` and `UpdateAttachmentRequest` interfaces are defined in `shared/types/attachment.types.ts`.
  - [ ] `transformAttachment(row: AttachmentModel): TransformedAttachment` returns `{ id, card_id, name, path, type }` where `path` is resolved via `fileStorage.getPublicUrl(row.path)`.
- **Dependencies:** `tasks/001-infrastructure-setup.md`, `tasks/002-database-schema.md`, `tasks/005-api-transformers.md`
- **Output Files:**
  - `shared/enums/FileTypeEnum.ts`
  - `shared/schemas/attachment.schema.ts`
  - `shared/types/attachment.types.ts`
  - `server/transformers/attachment.transformer.ts`

---

### Task 019-B: File Storage Service

- **Agent:** API
- **Spec:** `specs/client-attachments.md`
- **Section:** Data Model → File Storage; Edge Cases (file size, deleteFile failure, env var absent)
- **Acceptance Criteria:**
  - [ ] AC-2 (partial): `fileStorage.uploadFile(file, MEDIA_DIR)` persists the binary to `storage/media/` and returns the relative file path.
  - [ ] AC-2 (partial): `MAX_FILE_SIZE` is read from the `MAX_FILE_SIZE` env var; defaults to `10485760` (10 MB) when the var is absent.
  - [ ] AC-2 (partial): Files exceeding `MAX_FILE_SIZE` are rejected before any disk write — no partial files are left under `storage/media/`.
  - [ ] AC-4 (partial): `fileStorage.deleteFile(path)` removes the file from disk; if the file does not exist, logs the error server-side and returns without throwing.
  - [ ] `fileStorage.getPublicUrl(path)` returns the publicly accessible URL for a given relative path.
  - [ ] `MEDIA_DIR` constant is exported as `'storage/media/'`.
  - [ ] Edge case: When `deleteFile` fails after a DB record is already removed, the error is logged server-side and treated as a soft-success — no `500` is surfaced to the client from the service layer.
  - [ ] Edge case: `uploadFile` + DB insert are wrapped in a try/catch — on any error the uploaded file is cleaned up and no orphaned DB records are created.
- **Dependencies:** `tasks/001-infrastructure-setup.md`
- **Output Files:**
  - `server/services/fileStorage.ts`

---

### Task 019-C: Attachment API Endpoints

- **Agent:** API
- **Spec:** `specs/client-attachments.md`
- **Section:** Acceptance Criteria → AC-1, AC-2, AC-3, AC-4
- **Acceptance Criteria:**
  - [ ] AC-1: `GET /api/client/attachment/file-types` returns `FileTypeEnum` as a key-value map with `200`.
  - [ ] AC-1: Requires `auth` and `active-user` middleware; returns `401` for unauthenticated requests.
  - [ ] AC-2: `POST /api/client/attachment/create/[cardId]` accepts `multipart/form-data` with `name`, `type`, and `file` fields; parsed via `readMultipartFormData()`.
  - [ ] AC-2: Validates `name` and `type` with `createAttachmentSchema`; returns `422` on validation failure.
  - [ ] AC-2: Rejects files exceeding `MAX_FILE_SIZE` with `422` — does not persist anything to disk or DB.
  - [ ] AC-2: Persists file via `fileStorage.uploadFile(file, MEDIA_DIR)` and inserts an `attachments` DB record with `userId`, `cardId`, `name`, `type`, `path`.
  - [ ] AC-2: Returns `transformAttachment` output with `201`.
  - [ ] AC-2: Calls `createCardLog({ type: 2, data: { message, attachment } })` after successful creation.
  - [ ] AC-2: Broadcasts `attachment_create` to the `card#{cardId}` Centrifugo channel after successful creation.
  - [ ] AC-2: Triggers `CardNewAttachmentNotification` to all card members after successful creation.
  - [ ] AC-2: Returns `404` if `cardId` does not exist or the authenticated user is not a card member (does not leak card existence).
  - [ ] AC-2: Returns `401` for unauthenticated requests.
  - [ ] AC-3: `PUT /api/client/attachment/[id]` accepts a JSON body with optional `name` and/or `type` fields.
  - [ ] AC-3: Validates with `updateAttachmentSchema`; returns `422` for empty body `{}` or invalid field values.
  - [ ] AC-3: Updates `name` and/or `type` on the `attachments` record; does not modify `path` or the physical file on disk.
  - [ ] AC-3: Returns updated `transformAttachment` result with `200`.
  - [ ] AC-3: Calls `createCardLog({ type: 2, data: { message, changes, old_data } })` after successful update.
  - [ ] AC-3: Broadcasts `attachment_update` to the `card#{cardId}` Centrifugo channel after successful update.
  - [ ] AC-3: Returns `404` if attachment `id` does not exist or belongs to another user.
  - [ ] AC-3: Returns `401` for unauthenticated requests.
  - [ ] AC-4: `DELETE /api/client/attachment/[id]` verifies ownership before proceeding.
  - [ ] AC-4: Deletes the physical file via `fileStorage.deleteFile(path)`.
  - [ ] AC-4: Deletes the `attachments` DB record.
  - [ ] AC-4: Returns `200` (or `204`) on success.
  - [ ] AC-4: Calls `createCardLog({ type: 2, data: { message, attachment } })` after successful deletion.
  - [ ] AC-4: Broadcasts `attachment_delete` to the `card#{cardId}` Centrifugo channel after successful deletion.
  - [ ] AC-4: Returns `404` if attachment does not exist or belongs to another user.
  - [ ] AC-4: Returns `401` for unauthenticated requests.
- **Dependencies:** Task 019-A, Task 019-B, `tasks/001-infrastructure-setup.md`, `tasks/004-authentication.md`, `tasks/005-api-transformers.md`, `tasks/007-audit-trail.md`, `tasks/008-notification-system.md`, `tasks/010-realtime-centrifugo.md`, `tasks/002-database-schema.md`, `tasks/018-client-comments.md`
- **Output Files:**
  - `server/api/client/attachment/file-types.get.ts`
  - `server/api/client/attachment/create/[cardId].post.ts`
  - `server/api/client/attachment/[id].put.ts`
  - `server/api/client/attachment/[id].delete.ts`

---

### Task 019-D: transformCard Attachment Split

- **Agent:** API
- **Spec:** `specs/client-attachments.md`
- **Section:** Acceptance Criteria → AC-5; Data Model → Comment → Attachment FK
- **Acceptance Criteria:**
  - [ ] AC-5: `transformCard` is updated to separate attachments into two arrays: `card_attachments` for attachments where `comments.attachmentId IS NULL` (standalone), and `comment_attachments` for attachments where `comments.attachmentId = attachments.id` (linked to a comment).
  - [ ] AC-5: Both arrays use `transformAttachment` output shape: `{ id, card_id, name, path, type }`.
  - [ ] AC-5: The `TransformedCard` interface is updated to include `card_attachments: TransformedAttachment[]` and `comment_attachments: TransformedAttachment[]`.
  - [ ] Split logic returns empty `[]` for both arrays when a card has no attachments.
  - [ ] Split correctly handles cards where all attachments are standalone (non-empty `card_attachments`, empty `comment_attachments`).
  - [ ] Split correctly handles cards where all attachments are comment-linked (empty `card_attachments`, non-empty `comment_attachments`).
- **Dependencies:** Task 019-A, `tasks/005-api-transformers.md`, `tasks/002-database-schema.md`, `tasks/018-client-comments.md`
- **Output Files:**
  - `server/transformers/card.transformer.ts` *(update)*

---

## Wave 2 — Frontend (Frontend Agent)

### Task 019-E: Attachment API Composable & Card Store Updates

- **Agent:** Frontend
- **Spec:** `specs/client-attachments.md`
- **Section:** State Management; API Endpoints Consumed
- **Acceptance Criteria:**
  - [ ] `use-attachment-api.ts` exposes `fetchFileTypes()`, `uploadAttachment(cardId, payload)`, `updateAttachment(id, payload)`, and `deleteAttachment(id)` with explicit return types (all mutations return `TransformedAttachment` or `void`).
  - [ ] `fetchFileTypes` uses `useFetch` or `useAsyncData`; mutation functions use `$fetch` per project convention.
  - [ ] Composable exposes per-call reactive `loading` and `error` state accessible to consuming components.
  - [ ] `card.ts` Pinia store is updated to hold `card_attachments: TransformedAttachment[]` and `comment_attachments: TransformedAttachment[]`, initialised as empty arrays.
  - [ ] On `attachment_create` Centrifugo event → push new attachment into `card_attachments` or `comment_attachments` based on whether an associated comment is present.
  - [ ] On `attachment_update` Centrifugo event → find and replace the matching attachment in place in whichever array it belongs to.
  - [ ] On `attachment_delete` Centrifugo event → remove the attachment from whichever array it belongs to.
- **Dependencies:** Task 019-A, Task 019-C, Task 019-D, `tasks/010-realtime-centrifugo.md`, `tasks/004-authentication.md`
- **Output Files:**
  - `app/composables/api/use-attachment-api.ts`
  - `app/stores/card.ts` *(update)*

---

### Task 019-F: File.vue, Attachment List & Comment Input Components

- **Agent:** Frontend
- **Spec:** `specs/client-attachments.md`
- **Section:** Acceptance Criteria → AC-6, AC-7, AC-8; Component Tree
- **Acceptance Criteria:**
  - [ ] AC-6: `attachment-list.vue` reads `card_attachments` from the `card.ts` store and renders each attachment; image attachments (`type === FileTypeEnum.Image`) render a clickable thumbnail or icon.
  - [ ] AC-6: Clicking an image attachment opens `vue-easy-lightbox` (or `nuxt-easy-lightbox`) in full-screen with the resolved image URL.
  - [ ] AC-6: Non-image attachments do not trigger the lightbox on click.
  - [ ] AC-6: When an image attachment is removed from the store reactively (via `attachment_delete` event), it is also removed from the lightbox image list without error.
  - [ ] AC-7: `CommentInput.vue` exposes an `isCreatingCommentWithAttachment` boolean flag (via reactive state or prop/emit pattern).
  - [ ] AC-7: When the flag is `true`, the upload composable links the resulting attachment to the in-progress comment via `comments.attachmentId`; the attachment lands in `comment_attachments`, not `card_attachments`.
  - [ ] AC-7: Cancellation or failure during a comment-with-attachment flow cleans up any orphaned attachment record.
  - [ ] AC-8: `components/ui/File.vue` renders a styled file input that accepts any file type.
  - [ ] AC-8: `File.vue` displays a loading or progress state while the multipart upload is in flight.
  - [ ] AC-8: `File.vue` shows an inline error message when the selected file exceeds 10 MB client-side or the server returns a `422` response.
  - [ ] AC-8: `File.vue` emits the selected `File` object to the parent component for upload via the composable.
- **Dependencies:** Task 019-E, `tasks/010-realtime-centrifugo.md`
- **Output Files:**
  - `app/components/ui/File.vue`
  - `app/components/attachment-list.vue`
  - `app/components/comment-input.vue` *(update — add `isCreatingCommentWithAttachment` flag)*

---

## Wave 3 — Tests (Testing Agent)

### Task 019-G: Attachment Unit & E2E Tests

- **Agent:** Testing
- **Spec:** `specs/client-attachments.md`
- **Section:** All ACs; Edge Cases
- **Acceptance Criteria:**
  - [ ] **Unit — `fileStorage.ts`:** `uploadFile` saves the file to `storage/media/` and returns a relative path. `uploadFile` rejects files over `MAX_FILE_SIZE` without writing to disk. `deleteFile` removes the file on success. `deleteFile` logs and does not throw when the file does not exist. `getPublicUrl` returns the correct public URL. `MAX_FILE_SIZE` defaults to `10485760` when the env var is absent.
  - [ ] **Unit — `file-types.get.ts`:** Returns `200` with the full `FileTypeEnum` key-value map. Returns `401` with no or invalid JWT.
  - [ ] **Unit — `create/[cardId].post.ts`:** Happy path returns `201` with a transformed attachment. Returns `422` when `name` is missing. Returns `422` when `name` exceeds 50 chars. Returns `422` when `type` is out of range (e.g. `6`). Returns `422` when file exceeds 10 MB (no disk write, no DB insert). Returns `404` for an unknown `cardId`. Returns `404` when the authenticated user is not a card member. Returns `401` with no JWT. Verifies `createCardLog` is called after successful upload. Verifies `attachment_create` is broadcast to Centrifugo. Verifies `CardNewAttachmentNotification` is triggered.
  - [ ] **Unit — `[id].put.ts`:** Happy path returns `200` with updated fields. Returns `422` on empty body `{}`. Returns `422` for a `type` value outside 0–5. Returns `404` for an unknown `id`. Returns `404` when the attachment belongs to another user. Returns `401` with no JWT. Verifies `createCardLog` is called after update. Verifies `attachment_update` is broadcast to Centrifugo.
  - [ ] **Unit — `[id].delete.ts`:** Happy path returns `200` or `204`. Calls `fileStorage.deleteFile` with the correct path. Removes the DB record. Returns `404` for an unknown `id`. Returns `404` when the attachment belongs to another user. Returns `401` with no JWT. Verifies `createCardLog` is called after deletion. Verifies `attachment_delete` is broadcast to Centrifugo.
  - [ ] **Unit — `attachment.transformer.ts`:** `transformAttachment` returns `{ id, card_id, name, path, type }`. `path` is the resolved public URL from `getPublicUrl`. Handles the minimum valid row shape without error.
  - [ ] **Unit — `card.transformer.ts` (split):** Standalone attachments (`comments.attachmentId IS NULL`) appear in `card_attachments`. Comment-linked attachments appear in `comment_attachments`. Both arrays are empty `[]` when the card has no attachments. Mixed attachment sets are split correctly.
  - [ ] **E2E — attachment flow:** Authenticated user uploads a valid file to a card → attachment appears in the card detail modal. User renames an attachment → updated name is visible immediately. User deletes an attachment → it is removed from the modal list. Uploading a file >10 MB shows an inline error in `File.vue` and does not create a record. Image attachment click opens the lightbox in full-screen. Non-image attachment click does not open the lightbox. Unauthenticated request to any attachment endpoint returns `401`.
- **Dependencies:** Task 019-A, Task 019-B, Task 019-C, Task 019-D, Task 019-E, Task 019-F
- **Output Files:**
  - `tests/unit/server/services/fileStorage.test.ts`
  - `tests/unit/server/api/client/attachment/file-types.get.test.ts`
  - `tests/unit/server/api/client/attachment/create-cardId.post.test.ts`
  - `tests/unit/server/api/client/attachment/id.put.test.ts`
  - `tests/unit/server/api/client/attachment/id.delete.test.ts`
  - `tests/unit/server/transformers/attachment.transformer.test.ts`
  - `tests/unit/server/transformers/card.transformer.split.test.ts`
  - `tests/e2e/client/attachments.spec.ts`
