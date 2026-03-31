# Spec: Client Attachments

## Status: Approved

---

## Purpose

Enable users to upload, view, update, and delete file attachments on cards. Attachments are stored on disk under `storage/media/`, served publicly, and displayed in the card detail modal. Each operation produces a CardLog entry and broadcasts a Centrifugo event to all subscribers of the card channel. Image attachments support full-screen lightbox preview. Attachments can be standalone (card-level) or linked to a comment.

---

## User Stories

- As a user, I can upload a file attachment to a card so that teammates can access supporting documents.
- As a user, I can rename an attachment or change its file type classification so that the listing stays accurate.
- As a user, I can delete an attachment so that outdated or mistaken files are removed from the card and from disk.
- As a user, I can preview image attachments in a full-screen lightbox without leaving the card detail modal.
- As a user, I can see which attachments are standalone vs. linked to a specific comment so that context is preserved.
- As a user, I receive a list of supported file type enum values so that the UI can render correct type icons and options.

---

## Acceptance Criteria

### AC-1 — List File Types (`GET /api/client/attachment/file-types`)
- [ ] Returns the full `FileTypeEnum` as a key-value map or array: `Other=0, Image=1, Video=2, Audio=3, PDF=4, Excel=5`.
- [ ] Requires authentication (`auth` middleware) and active-user status (`active-user` middleware).
- [ ] Returns `200` with enum values; unauthenticated requests return `401`.

### AC-2 — Upload Attachment (`POST /api/client/attachment/create/[cardId]`)
- [ ] Accepts `multipart/form-data` with fields: `name` (string), `type` (FileTypeEnum int 0–5), `file` (binary).
- [ ] Validates `name` (min 1, max 50 chars) and `type` (integer 0–5) using `createAttachmentSchema` from `shared/schemas/attachment.schema.ts`.
- [ ] Rejects files exceeding `MAX_FILE_SIZE` (10 MB / 10,485,760 bytes) with `422`.
- [ ] Persists the file to `storage/media/` via `fileStorage.uploadFile(file, MEDIA_DIR)`.
- [ ] Creates an `attachments` DB record with `userId`, `cardId`, `name`, `type`, `path`.
- [ ] Returns the transformed attachment (`transformAttachment`) as `{ id, card_id, name, path, type }` with `201`.
- [ ] After successful creation, calls `createCardLog({ type: 2, data: { message, attachment } })` for the card.
- [ ] After successful creation, broadcasts `attachment_create` to the `card#{cardId}` Centrifugo channel.
- [ ] After successful creation, triggers `CardNewAttachmentNotification` to all card members.
- [ ] Returns `404` if `cardId` does not exist or the user does not have access to the card.
- [ ] Returns `401` for unauthenticated requests.

### AC-3 — Update Attachment Metadata (`PUT /api/client/attachment/[id]`)
- [ ] Accepts JSON body with optional fields: `name` (string, min 1, max 50), `type` (integer 0–5).
- [ ] Validates with `updateAttachmentSchema` (partial of `createAttachmentSchema`).
- [ ] Rejects an empty body (no fields provided) with `422`.
- [ ] Updates `name` and/or `type` columns on the `attachments` record; does not touch `path` or the file on disk.
- [ ] Returns the updated transformed attachment with `200`.
- [ ] After successful update, calls `createCardLog({ type: 2, data: { message, changes, old_data } })`.
- [ ] After successful update, broadcasts `attachment_update` to the `card#{cardId}` Centrifugo channel.
- [ ] Returns `404` if attachment `id` does not exist or belongs to another user.
- [ ] Returns `401` for unauthenticated requests.

### AC-4 — Delete Attachment (`DELETE /api/client/attachment/[id]`)
- [ ] Looks up the attachment by `id`, verifying ownership.
- [ ] Deletes the physical file from disk via `fileStorage.deleteFile(path)`.
- [ ] Deletes the `attachments` DB record.
- [ ] Returns `200` (or `204`) on success.
- [ ] After successful deletion, calls `createCardLog({ type: 2, data: { message, attachment } })`.
- [ ] After successful deletion, broadcasts `attachment_delete` to the `card#{cardId}` Centrifugo channel.
- [ ] Returns `404` if attachment does not exist or belongs to another user.
- [ ] Returns `401` for unauthenticated requests.

### AC-5 — Card Response Attachment Split
- [ ] `transformCard` separates attachments into two arrays:
  - `card_attachments` — attachments where `comments.attachmentId` is `NULL` (standalone).
  - `comment_attachments` — attachments where `comments.attachmentId = attachments.id` (linked to a comment).
- [ ] Both arrays use `transformAttachment` output shape: `{ id, card_id, name, path, type }`.

### AC-6 — Image Lightbox
- [ ] Attachments with `type === FileTypeEnum.Image` (1) render a clickable thumbnail or icon in the attachment list.
- [ ] Clicking an image attachment opens `vue-easy-lightbox` (or `nuxt-easy-lightbox`) in full-screen.
- [ ] Non-image attachments do not trigger the lightbox.

### AC-7 — Comment-Attachment Integration
- [ ] `CommentInput.vue` exposes an `isCreatingCommentWithAttachment` flag to signal that the upload flow is linked to a comment creation.
- [ ] When a comment with an attachment is created, the resulting attachment record is linked via `comments.attachmentId FK → attachments.id`.
- [ ] The linked attachment appears in `comment_attachments`, not `card_attachments`.

### AC-8 — File Upload UI
- [ ] `components/ui/File.vue` renders a styled file input that accepts any file type.
- [ ] Displays upload progress or loading state while the multipart request is in flight.
- [ ] Shows an error message when the file exceeds the 10 MB limit or the server returns a validation error.

---

## Component Tree / File Structure

```
server/api/client/attachment/
├── file-types.get.ts          # GET  /api/client/attachment/file-types
├── create/
│   └── [cardId].post.ts       # POST /api/client/attachment/create/[cardId]
├── [id].put.ts                # PUT  /api/client/attachment/[id]
└── [id].delete.ts             # DELETE /api/client/attachment/[id]

server/services/
└── fileStorage.ts             # uploadFile, deleteFile, getPublicUrl

shared/schemas/
└── attachment.schema.ts       # createAttachmentSchema, updateAttachmentSchema

shared/enums/
└── FileTypeEnum.ts            # FileTypeEnum const object

app/components/ui/
└── File.vue                   # Reusable file input component

app/composables/api/
└── use-attachment-api.ts      # wraps all 4 attachment endpoints

app/stores/
└── card.ts                    # holds card_attachments + comment_attachments arrays

app/components/
└── (card detail modal)
    ├── attachment-list.vue    # Renders standalone attachments, lightbox trigger
    └── comment-input.vue      # Comment + optional attachment upload (isCreatingCommentWithAttachment)
```

---

## Data Model

### Database Table — `attachments`

| Column | Type | Constraints |
|--------|------|-------------|
| `id` | `serial` | PK |
| `userId` | `integer` | NOT NULL, FK → `users.id` (cascade delete) |
| `cardId` | `integer` | NOT NULL, FK → `cards.id` (cascade delete) |
| `name` | `varchar(50)` | NOT NULL |
| `type` | `smallint` | NOT NULL — maps to `FileTypeEnum` |
| `path` | `varchar(255)` | NOT NULL — relative path under `storage/media/` |
| `createdAt` | `timestamp with tz` | NOT NULL, default `now()` |
| `updatedAt` | `timestamp with tz` | NOT NULL, default `now()` |

### Comment → Attachment FK

`comments.attachmentId` is a nullable FK → `attachments.id`. When set, the attachment is considered a comment attachment and appears in `comment_attachments`.

### File Type Enum

| Name | Value |
|------|-------|
| Other | 0 |
| Image | 1 |
| Video | 2 |
| Audio | 3 |
| PDF | 4 |
| Excel | 5 |

### File Storage

| Constant | Value |
|----------|-------|
| `MEDIA_DIR` | `storage/media/` |
| `MAX_FILE_SIZE` | `10485760` (10 MB) — read from `MAX_FILE_SIZE` env var |
| Public URL | Resolved via `fileStorage.getPublicUrl(path)` |

---

### API Endpoints Consumed

| Method | Endpoint | Request | Response |
|--------|----------|---------|----------|
| `GET` | `/api/client/attachment/file-types` | — | `{ [key: string]: number }` — FileTypeEnum map |
| `POST` | `/api/client/attachment/create/[cardId]` | `multipart/form-data`: `name: string`, `type: number`, `file: File` | `201` — `TransformedAttachment` |
| `PUT` | `/api/client/attachment/[id]` | `{ name?: string, type?: number }` | `200` — `TransformedAttachment` |
| `DELETE` | `/api/client/attachment/[id]` | — | `200` or `204` |

---

### TypeScript Interfaces

```typescript
// Transformer output shape (server/transformers/attachment.transformer.ts)
interface TransformedAttachment {
  id: number
  card_id: number
  name: string
  path: string      // public URL resolved by getPublicUrl()
  type: number      // FileTypeEnum value
}

// Full DB row shape (used internally in services)
interface AttachmentModel {
  id: number
  user_id: number
  card_id: number
  name: string
  type: number      // FileTypeEnum
  path: string      // relative storage path
  createdAt: string
  updatedAt: string
}

interface CreateAttachmentRequest {
  name: string
  type: number      // FileTypeEnum (0–5)
  file: File        // multipart file field
}

interface UpdateAttachmentRequest {
  name?: string
  type?: number     // FileTypeEnum (0–5)
}

// Extended card response (from transformCard)
interface TransformedCard {
  // ...other card fields...
  card_attachments: TransformedAttachment[]     // standalone attachments
  comment_attachments: TransformedAttachment[]  // attachments linked to comments
}
```

---

## State Management

- **`card.ts` Pinia store** manages the currently open card's attachment arrays:
  - `card_attachments: TransformedAttachment[]`
  - `comment_attachments: TransformedAttachment[]`
- On `attachment_create` Centrifugo event → push new attachment into the appropriate array based on whether it has an associated comment.
- On `attachment_update` Centrifugo event → find and replace the matching attachment in place.
- On `attachment_delete` Centrifugo event → remove the attachment from whichever array it belongs to.
- `use-attachment-api.ts` composable encapsulates all `$fetch` calls and exposes loading/error state to components.
- Components read attachment arrays from the store; mutations go through the composable → store.

---

## Edge Cases

| Scenario | Expected Behaviour |
|----------|--------------------|
| File exceeds 10 MB | Return `422` with a clear `message`; do not persist anything to disk or DB |
| `cardId` does not exist | Return `404` |
| User uploads to a card they don't own / aren't a member of | Return `404` (do not leak card existence) |
| `fileStorage.deleteFile` fails after DB record deletion | Log the error server-side; the DB record is already gone — treat as soft-success or surface a `500` |
| Attachment `id` belongs to a different user | Return `404` |
| `updateAttachmentSchema` receives empty body `{}` | Return `422` — at least one field required |
| Lightbox opened while another image is being deleted | Remove deleted attachment from lightbox list reactively via store update |
| Comment deleted while `isCreatingCommentWithAttachment` flow is in progress | Cancel or clean up orphaned attachment record |
| `MAX_FILE_SIZE` env var not set | Default to 10 MB (10,485,760 bytes) |
| Network error during multipart upload | Show error in `File.vue`; do not leave partial DB records (wrap upload + insert in try/catch with rollback) |

---

## Non-Goals

- Virus/malware scanning of uploaded files.
- Generating thumbnail previews server-side (client-side `<img>` tag only).
- Versioning / revision history of uploaded files.
- Folder or nested organisation of attachments within a card.
- Support for external storage providers (S3, GCS) — local disk only for this spec.
- Drag-and-drop file upload (future enhancement).
- Bulk upload of multiple files in a single request.

---

## Dependencies

| Dependency | Role |
|------------|------|
| `formidable` | Parses `multipart/form-data` on the Nitro server |
| `server/services/fileStorage.ts` | `uploadFile`, `deleteFile`, `getPublicUrl` |
| `server/utils/broadcast.ts` | Centrifugo `attachment_create`, `attachment_update`, `attachment_delete` events |
| `server/services/cardLog.ts` (or equivalent) | `createCardLog({ type: 2, ... })` after each mutation |
| `CardNewAttachmentNotification` (notification system) | Notifies card members on new attachment |
| `transformAttachment` | Shapes DB row into public API response |
| `transformCard` | Splits `card_attachments` vs `comment_attachments` |
| `shared/schemas/attachment.schema.ts` | `createAttachmentSchema`, `updateAttachmentSchema` (Zod) |
| `shared/enums/FileTypeEnum.ts` | `FileTypeEnum` const object |
| `vue-easy-lightbox` / `nuxt-easy-lightbox` | Full-screen image preview in card detail modal |
| `components/ui/File.vue` | Reusable file input UI component |
| `server/middleware/auth.ts` | JWT authentication guard on all attachment routes |
| `server/middleware/active-user.ts` | Active-user status guard on all attachment routes |
| `MAX_FILE_SIZE` env var | Configurable upload size limit (default 10 MB) |
