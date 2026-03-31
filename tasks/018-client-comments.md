# Tasks: Client Comments
**Spec:** `specs/client-comments.md`
**Status:** pending

---

## Overview

Implements card-level commenting: 4 Nitro route handlers (`create/[cardId]`, `show/[id]`, `[id]` PUT, `[id]` DELETE), a `transformComment` transformer, and 3 Vue components (`CommentAndActivity`, `CommentInput`, `CommentItem`). Every mutation emits a `CardLog` entry and a Centrifugo broadcast; comment creation additionally dispatches `CardNewCommentNotification` to all card members.

---

## Wave 1 — Data Layer (API Agent)

### Task 018-A: Validation Schemas & Comment Transformer

- **Agent:** API
- **Spec:** `specs/client-comments.md`
- **Section:** Data Model → Zod Validation Schemas; Data Model → Transformer; Data Model → TypeScript Interfaces
- **Acceptance Criteria:**
  - [ ] AC-C05: `body` must be at least 1 character; empty or missing body returns `422`
  - [ ] AC-U01: `updateCommentSchema` accepts `body` (string, min 1), `pin` (boolean), or both; at least one field must be present (empty payload → `422`)
  - [ ] AC-U02: Empty `body` string in update schema returns `422`
  - [ ] AC-C09: `transformComment(commentRow, { user, attachment })` returns `{ id, user_id, card_id, body, user_name, user_avatar, attachment }`
  - [ ] AC-S03: Show endpoint response shape matches `transformComment`
- **Dependencies:** `tasks/002-database-schema.md`, `tasks/003-validation-schemas.md`, `tasks/005-api-transformers.md`
- **Output Files:**
  - `shared/schemas/comment.schema.ts` — exports `createCommentSchema`, `updateCommentSchema`
  - `shared/types/comment.types.ts` — exports `CommentModel`, `CreateCommentRequest`, `UpdateCommentRequest` interfaces
  - `server/transformers/comment.transformer.ts` — exports `transformComment(commentRow, { user, attachment })`

---

### Task 018-B: Comment API Route Handlers

- **Agent:** API
- **Spec:** `specs/client-comments.md`
- **Section:** Acceptance Criteria (Create Comment / Show Comment / Update Comment / Delete Comment); Side Effects per Operation; Edge Cases
- **Acceptance Criteria:**
  - [ ] AC-C01: POST `/api/client/comment/create/[cardId]` protected by `auth` and `active-user` middleware; unauthenticated → `401`; inactive → `403`
  - [ ] AC-C02: Text-only create: valid `body` creates a `comments` row and returns the transformed comment
  - [ ] AC-C03: Create with existing attachment: `body` + `attachment_id` (integer) links the existing attachment to the comment
  - [ ] AC-C04: Multipart create: `multipart/form-data` with `body` + file data creates the attachment first, then creates the comment referencing the new `attachment_id`; orphaned attachment cleaned up if comment insert fails
  - [ ] AC-C06: On create success — `CardLog` entry created: `{ type: 1, data: { message, ...commentData } }`
  - [ ] AC-C07: On create success — Centrifugo broadcasts `comment_create` event to `card#{cardId}` channel
  - [ ] AC-C08: On create success — `CardNewCommentNotification` dispatched to all card members (including Telegram code-block formatting)
  - [ ] AC-S01: GET `/api/client/comment/show/[id]` returns single comment with nested `user` and `attachment` data
  - [ ] AC-S02: Non-existent comment ID → `404`
  - [ ] AC-U01: PUT `/api/client/comment/[id]` accepts `body` (string, min 1), `pin` (boolean), or both; empty payload `{}` → `422`
  - [ ] AC-U03: Only the comment owner, board owner, or admin may update; others → `403`
  - [ ] AC-U04: On update success — `CardLog` entry created: `{ type: 1, data: { message, changes, old_data } }`
  - [ ] AC-U05: On update success — Centrifugo broadcasts `comment_update` event to `card#{cardId}` channel
  - [ ] AC-U06: `pin: true` marks the comment pinned; `pin: false` unpins it
  - [ ] AC-D01: DELETE `/api/client/comment/[id]` removes the comment row; returns `204 No Content`
  - [ ] AC-D02: Only the comment owner, board owner, or admin may delete; others → `403`
  - [ ] AC-D03: On delete success — `CardLog` entry created: `{ type: 1, data: { message, ...commentData } }`
  - [ ] AC-D04: On delete success — Centrifugo broadcasts `comment_delete` event to `card#{cardId}` channel
  - [ ] AC-L01: Every create, update, and delete operation produces exactly one `CardLog` entry with `type: 1`
- **Dependencies:** Task 018-A, `tasks/004-authentication.md`, `tasks/005-api-transformers.md`, `tasks/007-audit-trail.md`, `tasks/008-notification-system.md`, `tasks/010-realtime-centrifugo.md`, `tasks/017-client-card-management.md`
- **Output Files:**
  - `server/api/client/comment/create/[cardId].post.ts`
  - `server/api/client/comment/show/[id].get.ts`
  - `server/api/client/comment/[id].put.ts`
  - `server/api/client/comment/[id].delete.ts`

---

## Wave 2 — Frontend (Frontend Agent)

### Task 018-C: Comment API Composable & Modal State

- **Agent:** Frontend
- **Spec:** `specs/client-comments.md`
- **Section:** Data Model → API Endpoints Consumed; State Management (`useCardModal`); Real-Time Updates
- **Acceptance Criteria:**
  - [ ] AC-RT01: Receiving a `comment_create` Centrifugo event inserts the new comment into the feed without a full page reload
  - [ ] AC-RT02: Receiving a `comment_update` event updates the affected comment in-place
  - [ ] AC-RT03: Receiving a `comment_delete` event removes the affected comment from the feed
- **Dependencies:** Task 018-A (types + schemas), Task 018-B (routes available), `tasks/010-realtime-centrifugo.md`
- **Output Files:**
  - `app/composables/api/use-comment-api.ts` — wraps POST create, GET show, PUT update, DELETE; exposes reactive comment list; applies RT event handlers for `comment_create / comment_update / comment_delete`
  - `app/composables/useCardModal.ts` — **update** (do not replace): add `isCreatingCommentWithAttachment: Ref<boolean>` to existing composable exports; used by `CommentInput` to switch between JSON and `multipart/form-data`

---

### Task 018-D: Comment Components

- **Agent:** Frontend
- **Spec:** `specs/client-comments.md`
- **Section:** Component Tree / File Structure; Acceptance Criteria (Delete Comment, Pin Display, CardLog)
- **Acceptance Criteria:**
  - [ ] AC-D05: `CommentItem` presents a confirmation modal before sending the delete request; request is not sent if user cancels
  - [ ] AC-P01: Pinned comments rendered in `CommentItem` with a visible pin indicator (icon + highlighted background)
  - [ ] AC-P02: `CommentAndActivity` sorts pinned comments to the top of the thread above unpinned ones
  - [ ] AC-P03: Pin/unpin control in `CommentItem` is visible only to the comment owner, board owner, or admin
  - [ ] AC-L02: `CommentAndActivity` displays `CardLog` entries interleaved with comments, sorted chronologically ascending
- **Dependencies:** Task 018-C
- **Output Files:**
  - `app/components/card-board/CommentAndActivity.vue` — chronological feed of comments + `CardLog` items interleaved; delegates RT mutations to `use-comment-api`; shows pinned comments first
  - `app/components/card-board/CommentInput.vue` — comment form with optional file-attachment toggle; sends JSON when `isCreatingCommentWithAttachment` is `false`, `multipart/form-data` otherwise; surfaces attachment-layer errors via `vue-sonner` toast
  - `app/components/card-board/CommentItem.vue` — single comment row: avatar, user name, relative timestamp, body, attachment preview, pin indicator, edit / delete / pin-toggle action buttons

---

## Wave 3 — Tests (Testing Agent)

### Task 018-E: Server Route Tests

- **Agent:** Testing
- **Spec:** `specs/client-comments.md`
- **Section:** Acceptance Criteria (Create / Show / Update / Delete / CardLog); Edge Cases
- **Acceptance Criteria:**
  - [ ] AC-C01: Unauthenticated create → `401`; inactive user → `403`
  - [ ] AC-C02: Valid text-only create → `200` with transformed comment body
  - [ ] AC-C03: Create with `attachment_id` → comment row has `attachment_id` set
  - [ ] AC-C04: Multipart create → attachment created first; comment references new `attachment_id`
  - [ ] AC-C05: Empty / missing `body` on create → `422`
  - [ ] AC-C06: Create produces one `CardLog` with `type: 1` and `comment_create` data shape
  - [ ] AC-C07: Create calls `broadcastToCentrifugo` with `comment_create` on `card#{cardId}`
  - [ ] AC-C08: Create dispatches `CardNewCommentNotification`
  - [ ] AC-C09: Create response shape matches `transformComment` contract
  - [ ] AC-S01: Show returns comment with nested `user` + `attachment`
  - [ ] AC-S02: Show with non-existent ID → `404`
  - [ ] AC-S03: Show response shape matches `transformComment`
  - [ ] AC-U01: Update with valid `body`, valid `pin`, or both → `200`
  - [ ] AC-U02: Update with empty string `body` → `422`
  - [ ] AC-U01 (empty payload): Update with `{}` → `422`
  - [ ] AC-U03: Update by non-owner / non-admin → `403`
  - [ ] AC-U04: Update produces one `CardLog` with `changes` + `old_data`
  - [ ] AC-U05: Update calls `broadcastToCentrifugo` with `comment_update`
  - [ ] AC-U06: `pin: true` sets `pin` to `true`; `pin: false` sets `pin` to `false`
  - [ ] AC-D01: Delete removes comment row from DB
  - [ ] AC-D02: Delete by non-owner / non-admin → `403`
  - [ ] AC-D03: Delete produces one `CardLog` entry
  - [ ] AC-D04: Delete calls `broadcastToCentrifugo` with `comment_delete`
  - [ ] AC-L01: Each of create / update / delete produces exactly one `CardLog` entry with `type: 1`
- **Dependencies:** Task 018-A, Task 018-B
- **Output Files:**
  - `tests/unit/server/api/client/comment/create.test.ts`
  - `tests/unit/server/api/client/comment/show.test.ts`
  - `tests/unit/server/api/client/comment/update.test.ts`
  - `tests/unit/server/api/client/comment/delete.test.ts`

---

### Task 018-F: Component & Composable Tests

- **Agent:** Testing
- **Spec:** `specs/client-comments.md`
- **Section:** Acceptance Criteria (Delete Comment, Pin Display, Real-Time Updates, CardLog)
- **Acceptance Criteria:**
  - [ ] AC-D05: Confirmation modal renders before delete; cancelling does not invoke the delete API call
  - [ ] AC-P01: `CommentItem` with `pin: true` renders a pin indicator element
  - [ ] AC-P02: Feed in `CommentAndActivity` places pinned items before unpinned items regardless of `createdAt` order
  - [ ] AC-P03: Pin/unpin button is absent in the DOM when current user is not the comment owner, board owner, or admin
  - [ ] AC-RT01: Calling the `comment_create` handler on the composable appends the comment to the reactive list
  - [ ] AC-RT02: Calling the `comment_update` handler mutates the correct item in the reactive list
  - [ ] AC-RT03: Calling the `comment_delete` handler removes the correct item from the reactive list
  - [ ] AC-L02: `CommentAndActivity` renders `CardLog` entries interleaved with comments in ascending chronological order
- **Dependencies:** Task 018-C, Task 018-D
- **Output Files:**
  - `tests/unit/app/components/card-board/CommentAndActivity.test.ts`
  - `tests/unit/app/components/card-board/CommentInput.test.ts`
  - `tests/unit/app/components/card-board/CommentItem.test.ts`
  - `tests/unit/app/composables/api/use-comment-api.test.ts`
