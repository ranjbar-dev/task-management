# Spec: Client Comments

## Status: Approved

---

## Purpose

Provide card-level commenting for authenticated, active users. Comments support plain text, linked attachments, and new file uploads. A pin feature highlights important comments. Every comment operation emits a CardLog entry, a Centrifugo real-time broadcast, and (on create) a `CardNewCommentNotification` to all card members.

---

## User Stories

- **As a card member**, I can write a comment on a card so that I can communicate progress or questions.
- **As a card member**, I can attach a file to my comment so that I can share relevant assets inline.
- **As a card member**, I can edit or delete my own comments so that I can correct mistakes.
- **As a card member**, I can see all comments and activity interleaved chronologically so that I have full context.
- **As a board owner or admin**, I can pin an important comment so that it is visually prominent and surfaced at the top.
- **As a card member**, I receive a notification when a new comment is posted on a card I'm following.

---

## Acceptance Criteria

### Create Comment

- **AC-C01** — POST `/api/client/comment/create/[cardId]` is protected by `auth` and `active-user` middleware; unauthenticated or inactive requests receive `401`/`403`.
- **AC-C02** — Text-only: a request with a valid `body` string creates a `comments` row and returns the transformed comment.
- **AC-C03** — With existing attachment: a request with `body` + `attachment_id` (integer) links the existing attachment to the comment.
- **AC-C04** — With new file upload: a `multipart/form-data` request containing `body` + file data creates the attachment first, then creates the comment referencing the new `attachment_id`.
- **AC-C05** — `body` must be at least 1 character; an empty or missing body returns `422`.
- **AC-C06** — On success, a `CardLog` entry is created: `{ type: 1, data: { message, ...commentData } }`.
- **AC-C07** — On success, Centrifugo broadcasts a `comment_create` event to the `card#{cardId}` channel.
- **AC-C08** — On success, `CardNewCommentNotification` is dispatched to all card members (includes code-block formatting in the Telegram message).
- **AC-C09** — Response shape matches `transformComment`: `{ id, user_id, card_id, body, user_name, user_avatar, attachment }`.

### Show Comment

- **AC-S01** — GET `/api/client/comment/show/[id]` returns a single comment with nested `user` and `attachment` data.
- **AC-S02** — Requesting a non-existent comment ID returns `404`.
- **AC-S03** — Response shape matches `transformComment`.

### Update Comment

- **AC-U01** — PUT `/api/client/comment/[id]` accepts `body` (string, min 1), `pin` (boolean), or both; at least one field must be present.
- **AC-U02** — An empty `body` string returns `422`.
- **AC-U03** — Only the comment owner, board owner, or admin may update; others receive `403`.
- **AC-U04** — On success, a `CardLog` entry is created: `{ type: 1, data: { message, changes, old_data } }`.
- **AC-U05** — On success, Centrifugo broadcasts a `comment_update` event to the `card#{cardId}` channel.
- **AC-U06** — Setting `pin: true` marks the comment as pinned; setting `pin: false` unpins it.

### Delete Comment

- **AC-D01** — DELETE `/api/client/comment/[id]` removes the comment row.
- **AC-D02** — Only the comment owner, board owner, or admin may delete; others receive `403`.
- **AC-D03** — On success, a `CardLog` entry is created: `{ type: 1, data: { message, ...commentData } }`.
- **AC-D04** — On success, Centrifugo broadcasts a `comment_delete` event to the `card#{cardId}` channel.
- **AC-D05** — The UI presents a confirmation modal before sending the delete request; the request is not sent if the user cancels.

### Pin Display

- **AC-P01** — Pinned comments (`pin: true`) are rendered with a visible pin indicator (e.g., icon, highlighted background).
- **AC-P02** — Pinned comments appear at the top of the comment thread above unpinned ones.
- **AC-P03** — The pin/unpin control is only visible to the comment owner, board owner, or admin.

### Real-Time Updates

- **AC-RT01** — Receiving a `comment_create` Centrifugo event inserts the new comment into the feed without a full page reload.
- **AC-RT02** — Receiving a `comment_update` event updates the affected comment in-place.
- **AC-RT03** — Receiving a `comment_delete` event removes the affected comment from the feed.

### CardLog

- **AC-L01** — Every create, update, and delete operation produces exactly one `CardLog` entry with `type: 1`.
- **AC-L02** — CardLog entries are displayed interleaved with comments in `CommentAndActivity.vue`, sorted chronologically ascending.

---

## Component Tree / File Structure

```
server/api/client/comment/
├── create/[cardId].post.ts       # POST — create comment (text / attachment_id / multipart)
├── show/[id].get.ts              # GET  — fetch single comment with user + attachment
├── [id].put.ts                   # PUT  — update body and/or pin status
└── [id].delete.ts                # DELETE — remove comment

app/components/card-board/
├── CommentAndActivity.vue        # Combined chronological feed: comments + CardLog items
├── CommentInput.vue              # Comment form with optional file attachment toggle
└── CommentItem.vue               # Single comment row: avatar, name, timestamp, body, attachment, pin, actions

app/composables/
└── useCardModal.ts               # Shared modal state; exposes isCreatingCommentWithAttachment

shared/schemas/
└── comment.schema.ts             # createCommentSchema, updateCommentSchema (Zod)
```

---

## Data Model

### Database Schema

```typescript
export const comments = pgTable('comments', {
  id:           serial('id').primaryKey(),
  userId:       integer('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  cardId:       integer('card_id').notNull().references(() => cards.id, { onDelete: 'cascade' }),
  attachmentId: integer('attachment_id').references(() => attachments.id),
  body:         text('body').notNull(),
  pin:          boolean('pin').notNull().default(false),
  createdAt:    timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:    timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

### Zod Validation Schemas

```typescript
// shared/schemas/comment.schema.ts
export const createCommentSchema = z.object({
  body:          z.string().min(1),
  attachment_id: z.number().int().optional(),
})

export const updateCommentSchema = z.object({
  body: z.string().min(1).optional(),
  pin:  z.boolean().optional(),
})
```

### API Endpoints Consumed

| Method | Endpoint | Request | Response |
|--------|----------|---------|----------|
| `POST` | `/api/client/comment/create/[cardId]` | `{ body: string, attachment_id?: number }` or `multipart/form-data` with `body` + file | `CommentModel` (transformed) |
| `GET` | `/api/client/comment/show/[id]` | — | `CommentModel` with nested `user` + `attachment` |
| `PUT` | `/api/client/comment/[id]` | `{ body?: string, pin?: boolean }` | Updated `CommentModel` |
| `DELETE` | `/api/client/comment/[id]` | — | `204 No Content` |

### TypeScript Interfaces

```typescript
interface CommentModel {
  id: number
  user_id: number
  user_name: string
  user_avatar: string | null
  user?: UserModel
  card_id: number
  attachment_id?: number
  attachment?: AttachmentModel
  body: string
  pin: boolean
  createdAt: string
  updatedAt: string
}

interface CreateCommentRequest {
  body: string
  attachment_id?: number  // mutually exclusive with file upload
}

interface UpdateCommentRequest {
  body?: string
  pin?: boolean
}
```

### Transformer

```typescript
// transformComment(commentRow, { user, attachment })
// Returns: { id, user_id, card_id, body, user_name, user_avatar, attachment }
```

---

## State Management

### `useCardModal` Composable (`app/composables/useCardModal.ts`)

| Ref | Type | Purpose |
|-----|------|---------|
| `openCardModal` | `Ref<{ cardId: number; refreshCardData: () => void; cardData: CardModel } \| null>` | Active card context for the modal |
| `isCreatingCommentWithAttachment` | `Ref<boolean>` | Signals the UI that the comment being composed has an attachment; switches `CommentInput` to multipart mode |

- `CommentInput.vue` reads `isCreatingCommentWithAttachment` to decide whether to send JSON vs. `multipart/form-data`.
- On successful comment creation, the composable calls `refreshCardData()` to sync card-level state.
- The comment list in `CommentAndActivity.vue` is updated optimistically or via Centrifugo events; a full refresh is the fallback.

---

## Side Effects per Operation

| Operation | CardLog | Centrifugo Event | Notification |
|-----------|---------|-----------------|--------------|
| Create | `type: 1, data: { message, ...commentData }` | `comment_create` → `card#{cardId}` | `CardNewCommentNotification` → card members |
| Update | `type: 1, data: { message, changes, old_data }` | `comment_update` → `card#{cardId}` | — |
| Delete | `type: 1, data: { message, ...commentData }` | `comment_delete` → `card#{cardId}` | — |

---

## Edge Cases

- **Attachment race condition** — If the file upload for a new attachment succeeds but the comment insert fails, the orphaned attachment must be cleaned up (or the create handler wraps both in a transaction).
- **Deleted user** — If the comment author's account is deleted (cascade), all their comments are also deleted; the UI should handle empty `user_name` gracefully via transformer defaults.
- **Concurrent edit** — If two users edit the same comment simultaneously, last-write-wins; no optimistic conflict resolution is required in v1.
- **Pin permission enforcement** — A regular member who is not the comment owner must receive `403` when attempting to set `pin: true`; pin is reserved for owners and admins.
- **Empty update payload** — A PUT request with no fields (`{}`) should return `422` — do not silently no-op.
- **Large file upload** — File size limits are enforced at the attachment layer; `CommentInput.vue` should surface the error returned by the attachment endpoint.
- **Comment deleted while editing** — If a `comment_delete` Centrifugo event arrives while the edit form is open, dismiss the form and remove the item from the feed.

---

## Non-Goals

- **Threaded / nested replies** — Comments are flat in v1; no reply chains.
- **Reactions / emoji** — No emoji reactions on comments in this spec.
- **Markdown rendering** — Comment body is plain text; no rich-text editor.
- **Pagination of comments** — All comments for a card are loaded in a single request; pagination is a future enhancement.
- **Comment search** — Full-text search across comments is out of scope.
- **Read receipts** — No per-user read tracking for comments.

---

## Dependencies

| Dependency | Role |
|------------|------|
| `auth` server middleware | Validates JWT and populates `event.context.user` |
| `active-user` server middleware | Ensures the authenticated user's account is active |
| `transformComment` transformer | Normalises comment DB rows into `CommentModel` |
| `createCardLog` utility | Writes audit trail entries for every comment mutation |
| `broadcast` utility (`server/utils/broadcast.ts`) | Publishes Centrifugo events to `card#{cardId}` |
| `CardNewCommentNotification` | Dispatches notification (including Telegram message) to card members on comment create |
| `AttachmentModel` / attachment upload endpoint | Required for the "with new file" comment-create flow |
| `useCardModal` composable | Provides `openCardModal` context and `isCreatingCommentWithAttachment` flag to `CommentInput` |
| `CommentAndActivity.vue` | Depends on both comment list data and CardLog entries from the card data fetch |
| Centrifugo WebSocket | Real-time delivery of `comment_create/update/delete` events to connected clients |
| `vue-sonner` | Toast feedback on comment create/update/delete success and error |
