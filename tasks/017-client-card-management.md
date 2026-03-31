# Tasks: Client Card Management
**Spec:** `specs/client-card-management.md`
**Status:** pending

---

## Overview

Full card lifecycle management for authenticated, active board members. Covers 11 server endpoints grouped across CRUD core (create, quick-create, duplicate, show, update, delete) and sync/state operations (position reorder, member sync, single member remove, tag sync, per-user status update). All mutations broadcast dual-channel Centrifugo events (`board#{id}` and/or `card#{id}`) and dispatch 8 distinct notification types via BullMQ. The frontend layer consists of a client API composable (`use-card-api.ts`), a board store card slice (Centrifugo event handlers), a card item component (`Item.vue`), a card detail modal (`task-detail.vue`), and the `useCardModal` composable.

**Agent execution order:**
1. Wave 1 (API Agent): shared foundations → transformer → CRUD endpoints → sync/state endpoints
2. Wave 2 (Frontend Agent): API composable + board store → card item + modal + useCardModal
3. Wave 3 (Testing Agent): endpoint unit tests → component tests → e2e workflow

---

## Wave 1 — Data Layer (API Agent)

### Task 017-A: Shared Schemas, Types, Enum & Soft-Delete Utility

- **Agent:** API
- **Spec:** `specs/client-card-management.md`
- **Section:** Data Model — TypeScript Interfaces, Data Model — Enums, Acceptance Criteria AC-12
- **Acceptance Criteria:**
  - [ ] AC-12a: `softDeleteCard(cardId)` sets `cards.deleted_at = now()` via Drizzle update.
  - [ ] AC-12b: `restoreCard(cardId)` sets `cards.deleted_at = null` via Drizzle update.
  - [ ] Zod schema `createCardSchema` validates: `name` (string, 1–255, required), `description` (string, optional), `position` (int, required), `user_ids` (int array, optional), `tag_ids` (int array, optional).
  - [ ] Zod schema `quickCreateCardSchema` validates: `name` (string, 1–255, required).
  - [ ] Zod schema `updateCardSchema` validates: `name` (string, 1–255, optional), `description` (string, optional), `board_list_id` (int, optional), `position` (int, optional).
  - [ ] Zod schema `updateCardPositionSchema` validates: `board_list_id` (int, required), `items` (array of `{ id: int, position: int }`, required).
  - [ ] Zod schema `syncCardMembersSchema` validates: `user_ids` (int array, required).
  - [ ] Zod schema `syncCardTagsSchema` validates: `tag_ids` (int array, required).
  - [ ] Zod schema `updateCardStatusSchema` validates: `user_id` (int, required), `status` (coerced int 0–4, required).
  - [ ] `CardUserStatusEnum` const object exported with `None=0`, `Pending=1`, `Doing=2`, `OnHold=3`, `Done=4` and `as const`.
  - [ ] `CardUserStatusEnumValue` type derived via `typeof CardUserStatusEnum[keyof typeof CardUserStatusEnum]`.
  - [ ] `CardUserStatusColorMap` maps each status value (0–4) to its hex color: `0='#B6B1BE'`, `1='#EAB035'`, `2='#A777E1'`, `3='#E55151'`, `4='#48CC4C'`.
  - [ ] TypeScript interfaces exported from `shared/types/card.types.ts`: `CardAttachment`, `CardComment`, `CardCommentUser`, `CardTag`, `CardUser`, `CardUserStatus`, `Card`, `CreateCardRequest`, `QuickCreateCardRequest`, `UpdateCardRequest`, `SyncCardMembersRequest`, `SyncCardTagsRequest`, `UpdateCardStatusRequest`, `UpdateCardPositionRequest`.
  - [ ] `CardUser` interface includes `status: CardUserStatus` field (pivot from `card_user`).
  - [ ] `CardTag` interface includes `color: { name: string; value: string }` expanded color object.
  - [ ] `Card` interface includes `card_attachments: CardAttachment[]` (comment_id IS NULL) and `comment_attachments: CardAttachment[]` (comment_id IS NOT NULL) as separate fields.
- **Dependencies:** `tasks/002-database-schema.md`, `tasks/003-validation-schemas.md`
- **Output Files:**
  - `shared/schemas/card.schema.ts`
  - `shared/enums/CardUserStatusEnum.ts`
  - `shared/types/card.types.ts`
  - `server/utils/soft-delete.ts`

---

### Task 017-B: Card Transformer

- **Agent:** API
- **Spec:** `specs/client-card-management.md`
- **Section:** Data Model — Transformer, Acceptance Criteria AC-4
- **Acceptance Criteria:**
  - [ ] `transformCard(cardRow, { users, comments, attachments })` returns a fully shaped `Card` object matching the `Card` interface.
  - [ ] `users` array on the output includes the pivot `status` field pulled from `card_user` for each user.
  - [ ] `tags` array on the output expands each tag's color enum into `{ name: string, value: string }` (e.g., `{ name: "Violet", value: "#7C3AED" }`).
  - [ ] `card_attachments` contains only attachment records where `comment_id IS NULL`.
  - [ ] `comment_attachments` contains only attachment records where `comment_id IS NOT NULL`.
  - [ ] `comments` array is ordered (e.g., by `created_at` ascending) and each comment includes its nested `user` and `attachments`.
  - [ ] Transformer is the single, canonical function used by all 8 endpoints that return a card response body.
- **Dependencies:** `tasks/005-api-transformers.md`, Task 017-A
- **Output Files:**
  - `server/transformers/card.transformer.ts`

---

### Task 017-C: CRUD Core Endpoints (Create, Quick-Create, Duplicate, Show, Update, Delete)

- **Agent:** API
- **Spec:** `specs/client-card-management.md`
- **Section:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6, AC-12, AC-13, AC-14
- **Acceptance Criteria:**
  - [ ] **AC-1a:** `POST /api/client/card/create/[boardListId]` applies `auth` + `active-user` server middleware.
  - [ ] **AC-1b:** Validates request body against `createCardSchema`; returns `400` with Zod field errors on invalid input.
  - [ ] **AC-1c:** Inserts a new row into `cards` with the resolved `board_list_id` (from route param) and supplied `position`.
  - [ ] **AC-1d:** If `user_ids` is provided and non-empty, inserts `card_user` rows for each user ID and dispatches `CreateNewCardNotification` to each assigned user via `NotificationService`.
  - [ ] **AC-1e:** If `tag_ids` is provided and non-empty, inserts `card_tag` rows for each tag ID.
  - [ ] **AC-1f:** Broadcasts `card_create` event to `board#{boardId}` via `broadcastToCentrifugo` (fire-and-forget).
  - [ ] **AC-1g:** Returns `201` with card payload transformed by `transformCard`.
  - [ ] **AC-2a:** `POST /api/client/card/quick-create/[boardListId]` applies `auth` + `active-user` server middleware.
  - [ ] **AC-2b:** Validates request body against `quickCreateCardSchema`; returns `400` on invalid input.
  - [ ] **AC-2c:** Inserts card with `name` only; auto-assigns `position` as last in the list (queries current max position + 1).
  - [ ] **AC-2d:** Automatically inserts a `card_user` row for the currently authenticated user (self-assignment).
  - [ ] **AC-2e:** Dispatches `CreateNewCardNotification` to the creating user via `NotificationService`.
  - [ ] **AC-2f:** Broadcasts `card_create` event to `board#{boardId}` (fire-and-forget).
  - [ ] **AC-2g:** Returns `201` with card payload transformed by `transformCard`.
  - [ ] **AC-3a:** `POST /api/client/card/duplicate/[id]` applies `auth` + `active-user` server middleware.
  - [ ] **AC-3b:** Returns `404` if the source card does not exist or has `deleted_at IS NOT NULL`.
  - [ ] **AC-3c:** Inserts a new card row on the same `board_list_id` with `name` suffixed with ` (copy)`.
  - [ ] **AC-3d:** Sets new card's `position` as last in the list (max position + 1), not a copy of the original.
  - [ ] **AC-3e:** Does NOT copy `card_user`, `card_tag`, comments, or attachments to the duplicated card.
  - [ ] **AC-3f:** Broadcasts `card_create` event to `board#{boardId}` (fire-and-forget).
  - [ ] **AC-3g:** Returns `201` with duplicated card payload transformed by `transformCard`.
  - [ ] **AC-4a:** `GET /api/client/card/show/[id]` applies `auth` + `active-user` server middleware.
  - [ ] **AC-4b:** Returns `404` if the card does not exist or is soft-deleted (`deleted_at IS NOT NULL`).
  - [ ] **AC-4c:** Returns `200` with the full card payload via `transformCard`, including `tags[]`, `users[]` (each with pivot `status`), `comments[]`, `card_attachments[]`, and `comment_attachments[]`.
  - [ ] **AC-4d:** Tags in the response include the expanded `color: { name, value }` object — not a raw enum int.
  - [ ] **AC-5a:** `PUT /api/client/card/[id]` applies `auth` + `active-user` server middleware.
  - [ ] **AC-5b:** Validates request body against `updateCardSchema`; returns `400` on invalid input.
  - [ ] **AC-5c:** Returns `404` if card is not found or is soft-deleted.
  - [ ] **AC-5d:** Performs a partial update — only fields present in the request body are written to the DB row.
  - [ ] **AC-5e:** If `name` changed → dispatches `CardInfoChangeNotification` to all current card members; broadcasts `card_update` to **both** `board#{boardId}` AND `card#{cardId}`.
  - [ ] **AC-5f:** If `description` changed → dispatches `CardDescriptionChangeNotification` to all current card members; broadcasts `card_update` to **both** `board#{boardId}` AND `card#{cardId}`.
  - [ ] **AC-5g:** If `board_list_id` or `position` changed → broadcasts `card_update` to **both** `board#{boardId}` AND `card#{cardId}` (no notification required for positional moves).
  - [ ] **AC-5h:** Returns `200` with updated card payload via `transformCard`.
  - [ ] **AC-6a:** `DELETE /api/client/card/[id]` applies `auth` + `active-user` server middleware.
  - [ ] **AC-6b:** Returns `404` if card is not found or is already soft-deleted.
  - [ ] **AC-6c:** Calls `softDeleteCard(cardId)` to set `cards.deleted_at = now()` — does NOT remove the DB row.
  - [ ] **AC-6d:** Dispatches `DeleteCardNotification` to all current card members via `NotificationService`.
  - [ ] **AC-6e:** Broadcasts `card_delete` event to `board#{boardId}` (fire-and-forget).
  - [ ] **AC-6f:** Returns `200` with a success message.
  - [ ] **AC-12c:** All six endpoints filter queries with Drizzle `isNull(cards.deletedAt)` — soft-deleted cards never appear in show responses.
  - [ ] **AC-13 (partial):** All `broadcastToCentrifugo` calls are fire-and-forget and do not block the API response.
  - [ ] **AC-14 (partial):** All `NotificationService` calls enqueue BullMQ jobs — no inline notification delivery.
  - [ ] **Edge case:** Board access check — each endpoint verifies the requesting user is a member of the card's parent board (not just that the card exists).
- **Dependencies:** `tasks/002-database-schema.md`, `tasks/004-authentication.md`, `tasks/005-api-transformers.md`, `tasks/007-audit-trail.md`, `tasks/008-notification-system.md`, `tasks/010-realtime-centrifugo.md`, Task 017-A, Task 017-B
- **Output Files:**
  - `server/api/client/card/create/[boardListId].post.ts`
  - `server/api/client/card/quick-create/[boardListId].post.ts`
  - `server/api/client/card/duplicate/[id].post.ts`
  - `server/api/client/card/show/[id].get.ts`
  - `server/api/client/card/[id].put.ts`
  - `server/api/client/card/[id].delete.ts`

---

### Task 017-D: Position, Member Sync, Tag Sync & Status Endpoints

- **Agent:** API
- **Spec:** `specs/client-card-management.md`
- **Section:** AC-7, AC-8, AC-9, AC-10, AC-11, AC-13, AC-14
- **Acceptance Criteria:**
  - [ ] **AC-7a:** `POST /api/client/card/update-position/[id]` applies `auth` + `active-user` server middleware.
  - [ ] **AC-7b:** Validates request body against `updateCardPositionSchema`; returns `400` on invalid input.
  - [ ] **AC-7c:** Batch-updates `position` for all card IDs in `items` inside a single Drizzle **transaction** to prevent partial updates if one fails.
  - [ ] **AC-7d:** Sets `board_list_id` on the moved card (identified by the route `[id]`) to the provided `board_list_id`.
  - [ ] **AC-7e:** Broadcasts `card_position` event to `board#{boardId}` (fire-and-forget).
  - [ ] **AC-7f:** Returns `200` with a success message.
  - [ ] **AC-8a:** `POST /api/client/card/members/[id]` applies `auth` + `active-user` server middleware.
  - [ ] **AC-8b:** Validates request body against `syncCardMembersSchema`; returns `400` on invalid input.
  - [ ] **AC-8c:** Returns `404` if the card does not exist or is soft-deleted.
  - [ ] **AC-8d:** Computes diff between the provided `user_ids` and existing `card_user` rows for the card.
  - [ ] **AC-8e:** Inserts `card_user` rows for each newly added user ID; dispatches `AssignToCardNotification` to each newly added user via `NotificationService`.
  - [ ] **AC-8f:** Deletes `card_user` rows for each removed user ID; dispatches `RemoveFromCardNotification` to each removed user via `NotificationService`.
  - [ ] **AC-8g:** If additions occurred, broadcasts `card_add_users` to **both** `board#{boardId}` AND `card#{cardId}`.
  - [ ] **AC-8h:** If removals occurred, broadcasts `card_remove_users` to **both** `board#{boardId}` AND `card#{cardId}`.
  - [ ] **AC-8i:** Passing `user_ids: []` removes all existing members from the card (full sync to empty set).
  - [ ] **AC-8j:** Returns `200` with updated card payload via `transformCard`.
  - [ ] **AC-9a:** `POST /api/client/card/[cardId]/delete-member/[userId]` applies `auth` + `active-user` server middleware.
  - [ ] **AC-9b:** Returns `404` if the card does not exist or the specified user is not a member of the card.
  - [ ] **AC-9c:** Deletes the single `card_user` row for the specified `userId` on the card.
  - [ ] **AC-9d:** Dispatches `RemoveFromCardNotification` to the removed user via `NotificationService`.
  - [ ] **AC-9e:** Broadcasts `card_remove_users` event to **both** `board#{boardId}` AND `card#{cardId}` (fire-and-forget).
  - [ ] **AC-9f:** Returns `200` with a success message.
  - [ ] **AC-10a:** `POST /api/client/card/tags/[id]` applies `auth` + `active-user` server middleware.
  - [ ] **AC-10b:** Validates request body against `syncCardTagsSchema`; returns `400` on invalid input.
  - [ ] **AC-10c:** Returns `404` if the card does not exist or is soft-deleted.
  - [ ] **AC-10d:** Performs a full replace: deletes all existing `card_tag` rows for the card, then re-inserts with the provided `tag_ids`.
  - [ ] **AC-10e:** Passing `tag_ids: []` results in all tags being cleared (delete all, insert nothing).
  - [ ] **AC-10f:** Dispatches `CardNewTagNotification` to all current card members via `NotificationService`.
  - [ ] **AC-10g:** Returns `200` with updated card payload via `transformCard`.
  - [ ] **AC-11a:** `POST /api/client/card/update-status/[id]` applies `auth` + `active-user` server middleware.
  - [ ] **AC-11b:** Validates request body against `updateCardStatusSchema`; returns `400` on invalid input.
  - [ ] **AC-11c:** Returns `404` if the card does not exist or the specified `user_id` does not have a row in `card_user` for this card.
  - [ ] **AC-11d:** Updates `card_user.status` to the provided `status` value for the given `user_id` on the card.
  - [ ] **AC-11e:** Dispatches `CardStatusChangeNotification` to all current card members via `NotificationService`.
  - [ ] **AC-11f:** Returns `200` with updated card payload via `transformCard`.
  - [ ] **AC-13 (partial):** `card_add_users` and `card_remove_users` both broadcast to `board#{boardId}` AND `card#{cardId}`; all broadcasts fire-and-forget.
  - [ ] **AC-14 (partial):** `AssignToCardNotification`, `RemoveFromCardNotification`, `CardNewTagNotification`, `CardStatusChangeNotification` all dispatched via `NotificationService` BullMQ queue.
  - [ ] **Edge case:** Large `items` array in update-position is wrapped in a Drizzle transaction — if any individual update fails, the entire batch is rolled back.
  - [ ] **Edge case:** Status update (`update-status`) for a `user_id` not present in `card_user` returns `404`.
  - [ ] **Edge case:** Delete-member for a `userId` not found in `card_user` returns `404`.
- **Dependencies:** `tasks/002-database-schema.md`, `tasks/004-authentication.md`, `tasks/005-api-transformers.md`, `tasks/007-audit-trail.md`, `tasks/008-notification-system.md`, `tasks/010-realtime-centrifugo.md`, Task 017-A, Task 017-B
- **Output Files:**
  - `server/api/client/card/update-position/[id].post.ts`
  - `server/api/client/card/members/[id].post.ts`
  - `server/api/client/card/[cardId]/delete-member/[userId].post.ts`
  - `server/api/client/card/tags/[id].post.ts`
  - `server/api/client/card/update-status/[id].post.ts`

---

## Wave 2 — Frontend (Frontend Agent)

### Task 017-E: Card API Composable & Board Store Card Slice

- **Agent:** Frontend
- **Spec:** `specs/client-card-management.md`
- **Section:** State Management — `use-card-api`, State Management — Board Store Card Slice
- **Acceptance Criteria:**
  - [ ] `use-card-api.ts` exports `useCardApi()` composable with explicit return type.
  - [ ] `createCard(boardListId, body)` calls `$fetch('POST /api/client/card/create/[boardListId]')` and returns `Promise<Card>`.
  - [ ] `quickCreateCard(boardListId, body)` calls `$fetch('POST /api/client/card/quick-create/[boardListId]')` and returns `Promise<Card>`.
  - [ ] `duplicateCard(id)` calls `$fetch('POST /api/client/card/duplicate/[id]')` and returns `Promise<Card>`.
  - [ ] `showCard(id)` calls `useFetch('GET /api/client/card/show/[id]')` and returns `Promise<Card>`.
  - [ ] `updateCard(id, body)` calls `$fetch('PUT /api/client/card/[id]')` and returns `Promise<Card>`.
  - [ ] `deleteCard(id)` calls `$fetch('DELETE /api/client/card/[id]')` and returns `Promise<void>`.
  - [ ] `updateCardPosition(id, body)` calls `$fetch('POST /api/client/card/update-position/[id]')` and returns `Promise<void>`.
  - [ ] `syncCardMembers(id, body)` calls `$fetch('POST /api/client/card/members/[id]')` and returns `Promise<Card>`.
  - [ ] `deleteCardMember(cardId, userId)` calls `$fetch('POST /api/client/card/[cardId]/delete-member/[userId]')` and returns `Promise<void>`.
  - [ ] `syncCardTags(id, body)` calls `$fetch('POST /api/client/card/tags/[id]')` and returns `Promise<Card>`.
  - [ ] `updateCardStatus(id, body)` calls `$fetch('POST /api/client/card/update-status/[id]')` and returns `Promise<Card>`.
  - [ ] No component calls `$fetch` or `useFetch` directly — all card API interactions go through this composable.
  - [ ] Board store (`app/stores/board.ts`) card slice handles `card_create` Centrifugo event: pushes the new card into the correct list in store (`lists[].cards`).
  - [ ] Board store handles `card_update` event: finds card in store by ID and merges updated fields.
  - [ ] Board store handles `card_delete` event: removes card from store by ID.
  - [ ] Board store handles `card_position` event: re-sorts cards in the affected list(s) by updated positions.
  - [ ] Board store handles `card_add_users` event: updates `users[]` array on the relevant card in store.
  - [ ] Board store handles `card_remove_users` event: removes the user from `users[]` on the relevant card in store.
- **Dependencies:** Task 017-C, Task 017-D
- **Output Files:**
  - `app/composables/api/use-card-api.ts`
  - `app/stores/board.ts` *(add card event handlers to existing store)*

---

### Task 017-F: Card Item Component, Card Detail Modal & useCardModal Composable

- **Agent:** Frontend
- **Spec:** `specs/client-card-management.md`
- **Section:** Component Tree — `app/components/card/Item.vue`, `app/modals/task-detail.vue`, `app/composables/use-card-modal.ts`, State Management — `useCardModal`, Edge Cases (AC-10)
- **Acceptance Criteria:**
  - [ ] `app/components/card/Item.vue` renders a kanban card chip displaying: card `name`, tag color chips (using `CardTag.color.value`), assignee avatar stack, and a status badge per the current user's `CardUserStatus`.
  - [ ] `Item.vue` emits a click event / calls `openModal({ cardId })` to open the card detail modal when the card chip is clicked.
  - [ ] `Item.vue` uses `defineProps<{ card: Card }>()` with the shared `Card` interface.
  - [ ] `use-card-modal.ts` exports `useCardModal()` composable returning: `openCardModal: Ref<OpenCardModalOptions | null>`, `isCreatingCommentWithAttachment: Ref<boolean>`, `openModal(options)`, `closeModal()`.
  - [ ] `openCardModal` is `null` when no modal is open; setting it to an `OpenCardModalOptions` object opens the card detail modal.
  - [ ] `closeModal()` sets `openCardModal` to `null` only if `isCreatingCommentWithAttachment.value` is `false`; it is a no-op (or shows a warning) when an attachment upload is in progress.
  - [ ] `OpenCardModalOptions` interface has `cardId: number`, optional `cardData?: Card`, optional `refreshCardData?: () => Promise<void>`.
  - [ ] `app/modals/task-detail.vue` is the card detail modal: renders the full `Card` payload including members, tags, comments, and attachment lists.
  - [ ] Modal fetches fresh card data via `showCard(cardId)` on open (using `cardData` prop as initial value if provided).
  - [ ] Modal renders `users[]` with each user's name, avatar, and their `CardUserStatus` badge (color from `CardUserStatusColorMap`).
  - [ ] Modal renders `tags[]` with each tag's name and the expanded `color.value` as a color swatch.
  - [ ] Modal renders `card_attachments[]` (not linked to a comment) in a dedicated attachments section.
  - [ ] Modal renders `comment_attachments[]` inline within their parent comments.
  - [ ] Modal does NOT close (guard on `closeModal`) while `isCreatingCommentWithAttachment` is `true` (edge case: prevent accidental loss of in-progress file upload).
  - [ ] All card mutation actions within the modal call the appropriate `useCardApi()` function — no direct `$fetch` calls.
- **Dependencies:** Task 017-E
- **Output Files:**
  - `app/components/card/Item.vue`
  - `app/modals/task-detail.vue`
  - `app/composables/use-card-modal.ts`

---

## Wave 3 — Tests (Testing Agent)

### Task 017-G: Server Endpoint Unit Tests

- **Agent:** Testing
- **Spec:** `specs/client-card-management.md`
- **Section:** All ACs (AC-1 through AC-14)
- **Acceptance Criteria:**
  - [ ] **Create card** — happy path: `201` + `transformCard` payload; invalid body → `400` Zod errors; unauthenticated → `401`.
  - [ ] **Create card** — with `user_ids`: inserts `card_user` rows; `CreateNewCardNotification` dispatched once per assigned user; `card_create` broadcast sent to `board#{boardId}`.
  - [ ] **Create card** — with `tag_ids`: inserts `card_tag` rows.
  - [ ] **Quick-create card** — happy path: `201`; auto-position is max+1 in list; self-assigned to `card_user`; `CreateNewCardNotification` dispatched; `card_create` broadcast sent.
  - [ ] **Duplicate card** — happy path: `201`; name ends with ` (copy)`; position is last in list; no members/tags; `card_create` broadcast sent.
  - [ ] **Duplicate card** — soft-deleted source card → `404`.
  - [ ] **Show card** — happy path: `200` + full transformer output including `tags[]`, `users[]` with pivot status, split attachments.
  - [ ] **Show card** — soft-deleted card → `404`.
  - [ ] **Update card** — name change → `card_update` broadcast to `board#{boardId}` AND `card#{cardId}`; `CardInfoChangeNotification` dispatched to all members.
  - [ ] **Update card** — description change → `card_update` broadcast to both channels; `CardDescriptionChangeNotification` dispatched.
  - [ ] **Update card** — position/list change → `card_update` broadcast to both channels; no notification dispatched.
  - [ ] **Update card** — partial update: only supplied fields written; unspecified fields unchanged.
  - [ ] **Update card** — soft-deleted card → `404`.
  - [ ] **Delete card** — happy path: `200` success message; `deleted_at` set; DB row still exists; `DeleteCardNotification` dispatched to all members; `card_delete` broadcast to `board#{boardId}`.
  - [ ] **Delete card** — already soft-deleted → `404`.
  - [ ] **Update position** — happy path: all `items` batch-updated in a transaction; `board_list_id` set on moved card; `card_position` broadcast to `board#{boardId}`; returns `200`.
  - [ ] **Update position** — invalid body → `400`.
  - [ ] **Sync members** — additions: `card_user` rows inserted; `AssignToCardNotification` dispatched per new user; `card_add_users` broadcast to both channels.
  - [ ] **Sync members** — removals: `card_user` rows deleted; `RemoveFromCardNotification` dispatched per removed user; `card_remove_users` broadcast to both channels.
  - [ ] **Sync members** — empty `user_ids: []` removes all members.
  - [ ] **Sync members** — soft-deleted card → `404`.
  - [ ] **Delete single member** — happy path: `200`; row deleted; `RemoveFromCardNotification` dispatched; `card_remove_users` broadcast to both channels.
  - [ ] **Delete single member** — user not on card → `404`.
  - [ ] **Sync tags** — happy path: all-delete + re-insert; `CardNewTagNotification` dispatched to all members; returns `200` + updated card.
  - [ ] **Sync tags** — empty `tag_ids: []` clears all tags.
  - [ ] **Sync tags** — soft-deleted card → `404`.
  - [ ] **Update status** — happy path: `card_user.status` updated; `CardStatusChangeNotification` dispatched to all members; returns `200` + updated card.
  - [ ] **Update status** — `user_id` not a card member → `404`.
  - [ ] All broadcast calls verified as fire-and-forget (do not affect response timing).
  - [ ] All notification calls verified to enqueue BullMQ jobs, not execute inline.
- **Dependencies:** Task 017-A, Task 017-B, Task 017-C, Task 017-D
- **Output Files:**
  - `tests/unit/server/api/client/card/create.test.ts`
  - `tests/unit/server/api/client/card/quick-create.test.ts`
  - `tests/unit/server/api/client/card/duplicate.test.ts`
  - `tests/unit/server/api/client/card/show.test.ts`
  - `tests/unit/server/api/client/card/update.test.ts`
  - `tests/unit/server/api/client/card/delete.test.ts`
  - `tests/unit/server/api/client/card/update-position.test.ts`
  - `tests/unit/server/api/client/card/sync-members.test.ts`
  - `tests/unit/server/api/client/card/delete-member.test.ts`
  - `tests/unit/server/api/client/card/sync-tags.test.ts`
  - `tests/unit/server/api/client/card/update-status.test.ts`

---

### Task 017-H: Component & Composable Unit Tests

- **Agent:** Testing
- **Spec:** `specs/client-card-management.md`
- **Section:** Component Tree, State Management
- **Acceptance Criteria:**
  - [ ] `use-card-modal.ts` — `openModal` sets `openCardModal` to the provided options object.
  - [ ] `use-card-modal.ts` — `closeModal` sets `openCardModal` to `null` when `isCreatingCommentWithAttachment` is `false`.
  - [ ] `use-card-modal.ts` — `closeModal` does NOT change `openCardModal` when `isCreatingCommentWithAttachment` is `true`.
  - [ ] `Item.vue` — renders card `name`.
  - [ ] `Item.vue` — renders one tag color chip per tag in `card.tags`.
  - [ ] `Item.vue` — renders assignee avatars for each user in `card.users`.
  - [ ] `Item.vue` — emits click / calls `openModal` when the card chip is clicked.
  - [ ] `use-card-api.ts` — each function calls the correct HTTP method and endpoint URL.
  - [ ] `use-card-api.ts` — `showCard` uses read composable (`useFetch`), all mutation functions use `$fetch`.
  - [ ] Board store card slice — `card_create` event handler pushes card to the matching list.
  - [ ] Board store card slice — `card_delete` event handler removes card from store.
  - [ ] Board store card slice — `card_update` event handler merges fields on the correct card.
  - [ ] Board store card slice — `card_position` event handler re-sorts cards in the affected list.
  - [ ] Board store card slice — `card_add_users` / `card_remove_users` handlers update `users[]` on the correct card.
- **Dependencies:** Task 017-E, Task 017-F
- **Output Files:**
  - `tests/unit/app/composables/use-card-modal.test.ts`
  - `tests/unit/app/composables/api/use-card-api.test.ts`
  - `tests/unit/app/components/card/Item.test.ts`
  - `tests/unit/app/stores/board-card-slice.test.ts`

---

### Task 017-I: E2E Card Workflow Tests

- **Agent:** Testing
- **Spec:** `specs/client-card-management.md`
- **Section:** User Stories, Edge Cases
- **Acceptance Criteria:**
  - [ ] **Full create flow:** User opens a board, creates a card with name + description + assignees + tags → card appears in the correct kanban list with correct tag chips and assignee avatars.
  - [ ] **Quick-create flow:** User types a card name in the quick-create input and submits → card appears at the bottom of the list; user is auto-assigned.
  - [ ] **Duplicate flow:** User duplicates a card → new card appears in the same list with ` (copy)` suffix; does not carry members or tags.
  - [ ] **Card detail modal:** User clicks a card → modal opens showing name, description, members, tags, comments, and attachments; modal can be closed normally.
  - [ ] **Update card:** User edits a card name in the modal → the kanban board reflects the updated name in real time.
  - [ ] **Soft delete:** User deletes a card → card disappears from the board; `deleted_at` is set server-side; card does not reappear on page refresh.
  - [ ] **Drag to reorder:** User drags a card to a different position within a list → order is persisted after page refresh.
  - [ ] **Drag to different list:** User drags a card to a different list → card appears in the new list after refresh.
  - [ ] **Member sync:** User adds a member to a card → member avatar appears; removes a member → avatar disappears.
  - [ ] **Tag sync:** User adds/removes tags on a card → tag chips on the kanban card update to reflect the change.
  - [ ] **Status update:** User changes their own status on a card → status badge updates on the card chip.
  - [ ] **isCreatingCommentWithAttachment guard:** While a file is being attached to a comment, attempting to close the modal does not close it.
- **Dependencies:** Task 017-C, Task 017-D, Task 017-E, Task 017-F
- **Output Files:**
  - `tests/e2e/card-workflow.spec.ts`
