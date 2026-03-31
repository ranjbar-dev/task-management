# Tasks: Admin Board Management
**Spec:** `specs/admin-board-management.md`
**Status:** pending

---

## Overview

Full lifecycle admin control over boards: CRUD, member sync, icon list, paginated table, and activity log. 9 API endpoints under `server/api/admin/board/`, guarded by `auth` + `active-user` + `admin` middleware. Mutations broadcast Centrifugo events and dispatch notifications via `NotificationService`. Frontend delivers the admin board list page at `app/pages/board/index.vue` using a table + 5 modal components.

**Execution order:** Wave 1 tasks (A → B → C) must complete before Wave 2. Wave 2 must complete before Wave 3.

---

## Wave 1 — Data Layer (API Agent)

### Task 011-A: Shared Validation Schemas & Types
- **Agent:** API
- **Spec:** `specs/admin-board-management.md`
- **Section:** Validation Schemas, TypeScript Interfaces, Data Model
- **Acceptance Criteria:**
  - [ ] AC-3 (partial): `createBoardSchema` — `name: z.string().min(1).max(50)`, `description: z.string().min(1)`, `status: z.coerce.number().int().min(0).max(1).optional()`, `icon: z.coerce.number().int().min(0).max(14).optional()`
  - [ ] AC-4 (partial): `updateBoardSchema = createBoardSchema.partial()` — all fields optional, same constraints
  - [ ] AC-8 (partial): `syncBoardMembersSchema` — `user_ids: z.array(z.number().int())`
  - [ ] AC-9 (partial): `BoardIconEnum` const object (15 entries: bolt=0 … tag=14), `BoardIconNameMap` mapping each icon value to its Material Symbol name, `BoardStatusEnum` const object (Deactive=0, Active=1)
  - [ ] AC-3 / AC-4: Returns `422` with field-level errors when body fails any Zod schema — schema shapes enforce this contract
  - [ ] TypeScript types derived: `BoardStatus`, `BoardIcon`, `BoardIconOption`, `BoardModel`, `CreateBoardRequest`, `UpdateBoardRequest`, `SyncBoardMembersRequest`
- **Dependencies:** `tasks/003-validation-schemas.md`
- **Output Files:**
  - `shared/schemas/board.schema.ts`
  - `shared/types/board.types.ts`

---

### Task 011-B: Board Transformer
- **Agent:** API
- **Spec:** `specs/admin-board-management.md`
- **Section:** Component Tree / File Structure, AC-1, AC-3, AC-4, AC-6
- **Acceptance Criteria:**
  - [ ] AC-1 (partial): `transformBoard()` produces output with `id`, `name`, `description`, `icon`, `icon_name`, `status`, and `users_count` fields
  - [ ] AC-3 (partial): `transformBoard()` used in create response
  - [ ] AC-4 (partial): `transformBoard()` used in update response
  - [ ] AC-6 (partial): `transformBoard()` supports full relational output — includes `board_lists` (ordered), nested `cards` within each list, and `members` array when relational data is passed in
  - [ ] `icon_name` is resolved from `BoardIconNameMap` keyed on the board's `icon` value
  - [ ] `users_count` is included in list and table responses; omitted or zero when not applicable
- **Dependencies:** `tasks/005-api-transformers.md`, Task 011-A
- **Output Files:**
  - `server/transformers/board.transformer.ts`

---

### Task 011-C: Server API Route Handlers (All 9 Endpoints)
- **Agent:** API
- **Spec:** `specs/admin-board-management.md`
- **Section:** AC-1 through AC-9, Edge Cases
- **Acceptance Criteria:**

  **GET /api/admin/board/all — AC-1**
  - [ ] Returns array of all boards; each item passed through `transformBoard()`
  - [ ] Includes `users_count` per board
  - [ ] Returns `200` with empty array when no boards exist
  - [ ] Route guarded by `auth`, `active-user`, `admin` middleware

  **POST /api/admin/board/table — AC-2**
  - [ ] Accepts `SearchTablePayload` (filters, pagination, sort)
  - [ ] Returns `{ data: BoardModel[], total: number }` (paginated)
  - [ ] Supports sort and filter by board `name`, `status`, and `users_count`
  - [ ] Returns `400` on invalid `SearchTablePayload`
  - [ ] Route guarded by `auth`, `active-user`, `admin` middleware

  **POST /api/admin/board/create — AC-3**
  - [ ] Validates body against `createBoardSchema`; returns `422` with field-level Zod errors on failure
  - [ ] Inserts new board row in database
  - [ ] Returns `201` with transformed board object via `transformBoard()`
  - [ ] Broadcasts `board_create` event to board channel via `broadcastToCentrifugo()`
  - [ ] Dispatches `CreateNewBoardNotification` to any members assigned at creation time
  - [ ] Route guarded by `auth`, `active-user`, `admin` middleware

  **PUT /api/admin/board/[id] — AC-4**
  - [ ] Returns `404` if board does not exist
  - [ ] Validates body against `updateBoardSchema`; returns `422` on failure
  - [ ] Performs partial update — only provided fields are written
  - [ ] Returns `200` with updated transformed board object
  - [ ] Broadcasts `board_update` event via `broadcastToCentrifugo()`
  - [ ] If `status` changed: dispatches `BoardStatusChangeNotification` to all current board members
  - [ ] If `name` or `description` changed: dispatches `BoardInfoChangeNotification` to all current board members
  - [ ] No notification dispatched if neither `status`, `name`, nor `description` changed
  - [ ] Route guarded by `auth`, `active-user`, `admin` middleware

  **DELETE /api/admin/board/[id] — AC-5**
  - [ ] Returns `404` if board does not exist
  - [ ] Dispatches `DeleteBoardNotification` to **all** current members **before** deletion
  - [ ] Deletes the board; cascaded records (lists, cards, logs, memberships) removed via DB cascade rules
  - [ ] Broadcasts `board_delete` event via `broadcastToCentrifugo()`
  - [ ] Returns `200` on successful deletion
  - [ ] Route guarded by `auth`, `active-user`, `admin` middleware

  **GET /api/admin/board/overview/[id] — AC-6**
  - [ ] Returns `404` if board does not exist
  - [ ] Returns full board object: board fields + `board_lists` (ordered) + nested `cards` + `members`
  - [ ] Response produced by `transformBoard()` with full relational data
  - [ ] Returns `200` with transformed overview
  - [ ] Route guarded by `auth`, `active-user`, `admin` middleware

  **POST /api/admin/board/logs-table/[id] — AC-7**
  - [ ] Returns `404` if board does not exist
  - [ ] Accepts `SearchTablePayload`; returns paginated activity log entries for the specified board
  - [ ] Returns `400` on invalid `SearchTablePayload`
  - [ ] Route guarded by `auth`, `active-user`, `admin` middleware

  **POST /api/admin/board/members/[id] — AC-8**
  - [ ] Returns `404` if board does not exist
  - [ ] Validates body against `syncBoardMembersSchema`; returns `422` on failure
  - [ ] Replaces current member set with provided `user_ids` (idempotent sync — add new, remove stale)
  - [ ] Dispatches `JoinToBoardNotification` to each **newly added** user
  - [ ] Dispatches `RemoveFromBoardNotification` to each **removed** user
  - [ ] No notifications dispatched when the member set is unchanged
  - [ ] Empty `user_ids: []` removes all existing members and dispatches `RemoveFromBoardNotification` to each
  - [ ] Broadcasts `board_add_users` event via `broadcastToCentrifugo()`
  - [ ] Returns `200` with updated board member list
  - [ ] Route guarded by `auth`, `active-user`, `admin` middleware

  **GET /api/admin/board/icons — AC-9**
  - [ ] Returns full `BoardIconEnum` map as array of `{ value, key, icon_name }` objects (15 entries)
  - [ ] Response is derived from `BoardIconEnum` and `BoardIconNameMap` (static, no DB query)
  - [ ] Returns `200` with icon list
  - [ ] Route guarded by `auth`, `active-user`, `admin` middleware

- **Dependencies:** `tasks/002-database-schema.md`, `tasks/004-authentication.md`, `tasks/006-search-table-service.md`, `tasks/008-notification-system.md`, `tasks/010-realtime-centrifugo.md`, Task 011-A, Task 011-B
- **Output Files:**
  - `server/api/admin/board/all.get.ts`
  - `server/api/admin/board/table.post.ts`
  - `server/api/admin/board/create.post.ts`
  - `server/api/admin/board/[id].put.ts`
  - `server/api/admin/board/[id].delete.ts`
  - `server/api/admin/board/icons.get.ts`
  - `server/api/admin/board/overview/[id].get.ts`
  - `server/api/admin/board/logs-table/[id].post.ts`
  - `server/api/admin/board/members/[id].post.ts`

---

## Wave 2 — Frontend (Frontend Agent)

### Task 011-D: Pinia Store & API Composable
- **Agent:** Frontend
- **Spec:** `specs/admin-board-management.md`
- **Section:** State Management
- **Acceptance Criteria:**
  - [ ] `boardStore` (Setup Store) has state: `boards: BoardModel[]`, `currentBoard: BoardModel | null`, `isLoading: boolean`
  - [ ] `fetchAllBoards()` — calls `GET /api/admin/board/all` via `useFetch`, populates `boards`
  - [ ] `createBoard(payload)` — calls `POST /api/admin/board/create` via `$fetch`, appends result to `boards`
  - [ ] `updateBoard(id, payload)` — calls `PUT /api/admin/board/[id]` via `$fetch`, replaces matching entry in `boards`
  - [ ] `deleteBoard(id)` — calls `DELETE /api/admin/board/[id]` via `$fetch`, removes entry from `boards`
  - [ ] `fetchBoardOverview(id)` — calls `GET /api/admin/board/overview/[id]` via `$fetch`, sets `currentBoard`
  - [ ] `syncMembers(id, userIds)` — calls `POST /api/admin/board/members/[id]` via `$fetch`, updates `currentBoard.members`
  - [ ] `use-board-api.ts` composable wraps all 9 endpoint calls; components use the composable — no direct `$fetch` in `.vue` files
  - [ ] Composable uses `useFetch` for read endpoints and `$fetch` for mutation endpoints
  - [ ] Composable and store have explicit TypeScript return types
- **Dependencies:** Task 011-A, Task 011-C
- **Output Files:**
  - `app/stores/board.ts`
  - `app/composables/api/use-board-api.ts`

---

### Task 011-E: Admin Board Page & Modal Components
- **Agent:** Frontend
- **Spec:** `specs/admin-board-management.md`
- **Section:** AC-10, Component Tree / File Structure
- **Acceptance Criteria:**
  - [ ] AC-10: `app/pages/board/index.vue` uses `default` layout (`definePageMeta({ layout: 'default' })`)
  - [ ] AC-10: Page renders a searchable, paginated table backed by `POST /api/admin/board/table`; columns: icon, name, description, status badge, member count, actions
  - [ ] AC-10: "Add Board" button opens `board/add-modal.vue`
  - [ ] AC-10: Row "View" action opens `board/details-modal.vue` (calls overview endpoint, read-only)
  - [ ] AC-10: Row "Edit" action opens `board/edit-modal.vue` (pre-filled from row data, dispatches partial update)
  - [ ] AC-10: Row "Delete" action opens `board/delete-modal.vue` (confirmation); on confirm calls delete endpoint
  - [ ] AC-10: Row "Members" action opens `board/members-modal.vue` (multi-select users, pre-filled with current member IDs)
  - [ ] AC-10: All mutations (create / update / delete / sync) trigger a table refresh after success
  - [ ] AC-10: `board/add-modal.vue` and `board/edit-modal.vue` embed `board/icon-picker.vue`; icon picker fetches all 15 icons from `GET /api/admin/board/icons` and renders them as selectable options
  - [ ] AC-10: Status field uses `BoardStatusEnum` values (Active / Deactive)
  - [ ] AC-10: Toast notifications (via `vue-sonner`) confirm success or failure for every action
  - [ ] No business logic in `<template>` — all API interaction goes through the composable
  - [ ] All components use `<script setup lang="ts">` with typed `defineProps` and `defineEmits`
- **Dependencies:** Task 011-A, Task 011-D
- **Output Files:**
  - `app/pages/board/index.vue`
  - `app/components/board/add-modal.vue`
  - `app/components/board/edit-modal.vue`
  - `app/components/board/details-modal.vue`
  - `app/components/board/delete-modal.vue`
  - `app/components/board/members-modal.vue`
  - `app/components/board/icon-picker.vue`

---

## Wave 3 — Tests (Testing Agent)

### Task 011-F: Unit Tests — API Endpoints
- **Agent:** Testing
- **Spec:** `specs/admin-board-management.md`
- **Section:** AC-1 through AC-9, Edge Cases
- **Acceptance Criteria:**
  - [ ] `all.get.ts` — happy path returns `200` with transformed array; empty array when no boards
  - [ ] `table.post.ts` — valid payload returns paginated result; invalid payload returns `400`
  - [ ] `create.post.ts` — valid body returns `201` with board; invalid body returns `422`; `broadcastToCentrifugo` called with `board_create`; `CreateNewBoardNotification` dispatched when members assigned
  - [ ] `[id].put.ts` — valid update returns `200`; non-existent ID returns `404`; invalid body returns `422`; `BoardStatusChangeNotification` dispatched on status change; `BoardInfoChangeNotification` dispatched on name/description change; no notification when no relevant field changes; `broadcastToCentrifugo` always called with `board_update`
  - [ ] `[id].delete.ts` — existing board returns `200`; non-existent returns `404`; `DeleteBoardNotification` dispatched before deletion; `broadcastToCentrifugo` called with `board_delete`
  - [ ] `overview/[id].get.ts` — existing board returns full relational object; non-existent returns `404`
  - [ ] `logs-table/[id].post.ts` — valid payload returns paginated logs; non-existent board returns `404`; invalid payload returns `400`
  - [ ] `members/[id].post.ts` — valid sync returns `200` with updated members; non-existent board returns `404`; invalid body returns `422`; `JoinToBoardNotification` sent to new members only; `RemoveFromBoardNotification` sent to removed members only; empty `user_ids` removes all; `broadcastToCentrifugo` called with `board_add_users`
  - [ ] `icons.get.ts` — returns `200` with exactly 15 icon entries; each entry has `value`, `key`, `icon_name` fields
  - [ ] Edge case: `icon: 99` on create/update returns `422`
  - [ ] Edge case: `name` at 50 chars accepted; 51 chars returns `422`
  - [ ] Edge case: `user_ids` containing non-integer returns `422`
- **Dependencies:** Task 011-A, Task 011-B, Task 011-C
- **Output Files:**
  - `tests/unit/server/api/admin/board/all.get.test.ts`
  - `tests/unit/server/api/admin/board/table.post.test.ts`
  - `tests/unit/server/api/admin/board/create.post.test.ts`
  - `tests/unit/server/api/admin/board/id.put.test.ts`
  - `tests/unit/server/api/admin/board/id.delete.test.ts`
  - `tests/unit/server/api/admin/board/icons.get.test.ts`
  - `tests/unit/server/api/admin/board/overview.id.get.test.ts`
  - `tests/unit/server/api/admin/board/logs-table.id.post.test.ts`
  - `tests/unit/server/api/admin/board/members.id.post.test.ts`

---

### Task 011-G: E2E Tests — Admin Board CRUD Flow
- **Agent:** Testing
- **Spec:** `specs/admin-board-management.md`
- **Section:** AC-10, User Stories, Edge Cases
- **Acceptance Criteria:**
  - [ ] Admin logs in and navigates to the board list page — table renders with correct columns
  - [ ] Admin creates a board via Add modal — table refreshes, new row appears, success toast shown
  - [ ] Admin edits a board via Edit modal — updated data reflects in table row, success toast shown
  - [ ] Admin views board details via View modal — overview data loads (lists, cards, members visible)
  - [ ] Admin syncs board members via Members modal — member count updates in table row
  - [ ] Admin deletes a board via Delete confirmation modal — row removed from table, success toast shown
  - [ ] Icon picker in Add/Edit modal renders all 15 icons as selectable options
  - [ ] Status badge correctly reflects Active / Deactive enum value
  - [ ] Attempting to create board with empty name — form shows validation error, submission blocked
  - [ ] Search/filter in table narrows displayed results
- **Dependencies:** Task 011-A, Task 011-C, Task 011-D, Task 011-E
- **Output Files:**
  - `tests/e2e/admin-boards.spec.ts`
