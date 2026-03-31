# Tasks: Client BoardList Management
**Spec:** `specs/client-board-list-management.md`
**Status:** pending

---

## Overview

Implements full CRUD management of board lists (kanban columns) for authenticated board members. Covers six Nitro API route handlers, the `boardList` transformer, three Zod validation schemas, shared TypeScript interfaces, a Pinia store with five actions, a server-side membership access gate for client routes, Centrifugo broadcast on every mutation, an API composable, the `card/list.vue` kanban column component with horizontal drag-and-drop via `vue-draggable-plus`, and optimistic reorder with error rollback.

Cross-reference: **Task 015** (`tasks/015-client-board-kanban.md`) — the kanban board page hosts the lists rendered by this feature; `boardListStore` is populated by the board page on load.

Execution order:
1. **Wave 1 (API Agent)** — shared types + Zod schemas + transformer first, then all six server route handlers with Centrifugo broadcasts and the membership gate utility.
2. **Wave 2 (Frontend Agent)** — API composable + Pinia store (including socket event handlers), then the `card/list.vue` component with drag-and-drop (depends on Wave 1).
3. **Wave 3 (Testing Agent)** — endpoint unit tests and drag-drop e2e tests (depends on Waves 1 & 2).

---

## Wave 1 — Data Layer (API Agent)

### Task 016-A: Shared Types, Zod Schemas & boardList Transformer

- **Agent:** API
- **Spec:** `specs/client-board-list-management.md`
- **Section:** Data Model → TypeScript Interfaces; Data Model → Zod Validation Schemas; Component Tree / File Structure → `shared/`; Data Model → Transformer
- **Acceptance Criteria:**
  - [ ] `BoardListModel` interface is defined in `shared/types/boardList.types.ts` with fields: `id: number`, `board_id: number`, `name: string`, `position: number`, `cards: CardModel[]`.
  - [ ] `CreateBoardListRequest` interface is defined with `name: string` and `position: number`.
  - [ ] `UpdateBoardListRequest` interface is defined with `name?: string` and `position?: number`.
  - [ ] `UpdatePositionRequest` interface is defined with `items: Array<{ id: number; position: number }>`.
  - [ ] `createBoardListSchema` validates `{ name: z.string().min(1).max(50), position: z.number().int() }` — returns 422 on failure.
  - [ ] `updateBoardListSchema` is `createBoardListSchema.partial()` — at least one field must be present at the route layer; schema itself is fully partial.
  - [ ] `updatePositionSchema` validates `{ items: z.array(z.object({ id: z.number().int(), position: z.number().int() })) }` — `items` must not be empty.
  - [ ] `transformBoardList(row, { cards })` returns `{ id, board_id, name, position, cards[] }` where `cards` is the pre-fetched and transformed card array passed in via the second argument.
  - [ ] Transformer is exported from `server/transformers/boardList.transformer.ts`.
- **Dependencies:** `tasks/002-database-schema.md`, `tasks/005-api-transformers.md`
- **Output Files:**
  - `shared/types/boardList.types.ts`
  - `shared/schemas/boardList.schema.ts`
  - `server/transformers/boardList.transformer.ts`

---

### Task 016-B: Server API Endpoints (6 Routes) with Membership Gate & Centrifugo Broadcasts

- **Agent:** API
- **Spec:** `specs/client-board-list-management.md`
- **Section:** Acceptance Criteria → all six endpoint ACs; Centrifugo real-time events; Edge Cases → Non-member access, Admin bypass, update-position with unknown ids
- **Acceptance Criteria:**

  **Membership access gate (shared logic):**
  - [ ] A reusable server utility (or inline check) verifies that the authenticated user is either a member of the target board (`boardUsers` junction) or has the `admin` role — used by all six endpoints.
  - [ ] Non-member requests return HTTP 403 before any DB query executes to prevent data leakage.
  - [ ] All six endpoints are protected by `auth` and `active-user` middleware; unauthenticated requests return HTTP 401.

  **GET `/api/client/board-list/[boardId]`:**
  - [ ] Returns HTTP 200 with an array of `BoardListModel` objects for the specified board via `transformBoardList`.
  - [ ] Each list object includes `{ id, board_id, name, position, cards[] }`.
  - [ ] Lists are ordered by `position` ascending.
  - [ ] Returns HTTP 403 if the authenticated user is not a board member (and not an admin).
  - [ ] Returns HTTP 404 if the board does not exist.
  - [ ] Returns an empty array `[]` (not 404) if the board exists but has no lists.

  **GET `/api/client/board-list/show/[id]`:**
  - [ ] Returns HTTP 200 with a single `BoardListModel` including nested `cards[]`, each card containing its assigned users.
  - [ ] Returns HTTP 404 if the list does not exist.
  - [ ] Returns HTTP 403 if the authenticated user is not a member of the list's parent board (and not an admin).

  **POST `/api/client/board-list/create/[boardId]`:**
  - [ ] Validates request body with `createBoardListSchema`; returns HTTP 422 with field-level errors on failure.
  - [ ] `name` must be 1–50 characters; `position` must be a non-negative integer.
  - [ ] Returns HTTP 201 with the created `BoardListModel` via `transformBoardList`.
  - [ ] After successful insert, broadcasts a `list_create` event to the `board#{boardId}` Centrifugo channel with the full new `BoardListModel` as payload.
  - [ ] Returns HTTP 403 if the user is not a board member (and not an admin).

  **PUT `/api/client/board-list/[id]`:**
  - [ ] Validates request body with `updateBoardListSchema`; returns HTTP 422 if body is empty or invalid.
  - [ ] Updates only the fields present in the request body (partial update).
  - [ ] Returns HTTP 200 with the updated `BoardListModel` via `transformBoardList`.
  - [ ] After successful update, broadcasts a `list_update` event to the `board#{boardId}` Centrifugo channel with the updated `BoardListModel` as payload.
  - [ ] Returns HTTP 404 if the list does not exist.
  - [ ] Returns HTTP 403 if the user is not a board member (and not an admin).

  **DELETE `/api/client/board-list/[id]`:**
  - [ ] Deletes the specified list; all associated cards (and their attachments, comments, card-users) are removed via cascade (`onDelete: 'cascade'` defined in schema — confirmed in `tasks/002-database-schema.md`).
  - [ ] Returns HTTP 200 with the deleted list id or a success message.
  - [ ] After successful delete, broadcasts a `list_delete` event to the `board#{boardId}` Centrifugo channel with payload `{ id: number }`.
  - [ ] Returns HTTP 404 if the list does not exist.
  - [ ] Returns HTTP 403 if the user is not a board member (and not an admin).

  **POST `/api/client/board-list/update-position/[id]`:**
  - [ ] Validates request body with `updatePositionSchema`; returns HTTP 422 if `items` is empty or contains non-integer ids/positions.
  - [ ] Validates that all list ids in `items` belong to the board identified by the `[id]` route param — returns HTTP 403 or HTTP 404 if any id is foreign or unknown.
  - [ ] Updates the `position` column for every supplied list id in a single database transaction.
  - [ ] Returns HTTP 200 with the full updated array of `BoardListModel` objects (or a success flag).
  - [ ] After successful reorder, broadcasts a `list_position` event to the `board#{boardId}` Centrifugo channel with the full reordered `BoardListModel[]` as payload.
  - [ ] Returns HTTP 403 if the user is not a board member (and not an admin).

- **Dependencies:** Task 016-A, `tasks/002-database-schema.md`, `tasks/004-authentication.md`, `tasks/005-api-transformers.md`, `tasks/010-realtime-centrifugo.md`
- **Output Files:**
  - `server/api/client/board-list/[boardId].get.ts`
  - `server/api/client/board-list/show/[id].get.ts`
  - `server/api/client/board-list/create/[boardId].post.ts`
  - `server/api/client/board-list/[id].put.ts`
  - `server/api/client/board-list/[id].delete.ts`
  - `server/api/client/board-list/update-position/[id].post.ts`

---

## Wave 2 — Frontend (Frontend Agent)

### Task 016-C: API Composable & boardListStore Pinia Store

- **Agent:** Frontend
- **Spec:** `specs/client-board-list-management.md`
- **Section:** State Management; Data Model → API Endpoints Consumed; Centrifugo real-time events → Store mutation table
- **Acceptance Criteria:**

  **API composable (`use-board-list-api.ts`):**
  - [ ] `getBoardLists(boardId)` calls `GET /api/client/board-list/[boardId]` via `useFetch` and returns `BoardListModel[]`.
  - [ ] `getBoardList(id)` calls `GET /api/client/board-list/show/[id]` via `useFetch` and returns a single `BoardListModel` with nested cards and card-users.
  - [ ] `createList(boardId, payload: CreateBoardListRequest)` calls `POST /api/client/board-list/create/[boardId]` via `$fetch` and returns the created `BoardListModel`.
  - [ ] `updateList(id, payload: UpdateBoardListRequest)` calls `PUT /api/client/board-list/[id]` via `$fetch` and returns the updated `BoardListModel`.
  - [ ] `deleteList(id)` calls `DELETE /api/client/board-list/[id]` via `$fetch`.
  - [ ] `reorderLists(boardId, items: UpdatePositionRequest['items'])` calls `POST /api/client/board-list/update-position/[boardId]` via `$fetch` and returns the updated `BoardListModel[]`.
  - [ ] All functions have explicit return types; raw `fetch()` is never used; API errors propagate to callers.

  **boardListStore (`app/stores/boardList.ts`):**
  - [ ] `getBoardLists(boardId)` calls the composable and populates the store with the ordered list array; subsequent calls for the same boardId refresh the data.
  - [ ] `createList(boardId, payload)` calls the composable, appends the result to the store, and maintains `position` sort order.
  - [ ] `updateList(id, payload)` calls the composable and merges the updated fields into the existing entry in the store.
  - [ ] `deleteList(id)` calls the composable and removes the entry from the store by id.
  - [ ] `reorderLists(boardId, items)` calls the composable and replaces the store's list array with the returned reordered array.
  - [ ] Centrifugo `list_create` event: appends new list to store, deduplicated by id (no duplicate if event arrives while local optimistic update already applied).
  - [ ] Centrifugo `list_update` event: merges updated fields into the matching list in the store.
  - [ ] Centrifugo `list_delete` event: removes the list matching the received `id` from the store.
  - [ ] Centrifugo `list_position` event: replaces the full list array in the store with the received reordered payload.
  - [ ] Store is implemented using Pinia Setup Store syntax (not Options API).
  - [ ] Components access the store only via composables — no direct store access from templates.

- **Dependencies:** Task 016-B
- **Output Files:**
  - `app/composables/api/use-board-list-api.ts`
  - `app/stores/boardList.ts`

---

### Task 016-D: card/list.vue Kanban Column Component

- **Agent:** Frontend
- **Spec:** `specs/client-board-list-management.md`
- **Section:** Acceptance Criteria → Drag-and-drop reorder; Component Tree / File Structure → `app/components/card/list.vue`; Edge Cases → Duplicate positions, Concurrent reorder conflicts
- **Acceptance Criteria:**
  - [ ] `app/components/card/list.vue` renders a single kanban column, receiving the `BoardListModel` as a prop (`defineProps<{ list: BoardListModel }>()`).
  - [ ] The component displays the list name and the collection of cards within the column.
  - [ ] An **edit** control allows renaming the list (calls `boardListStore.updateList`); a **delete** control triggers deletion (calls `boardListStore.deleteList`) with a confirmation step.
  - [ ] Lists are wrapped in a `vue-draggable-plus` horizontal drag container in the parent kanban page (cross-reference: `tasks/015-client-board-kanban.md`); the `card/list.vue` component itself is the drag item.
  - [ ] On a successful column drop, the handler calls `boardListStore.reorderLists(boardId, items)` with the complete updated `{ id, position }` array for all lists in the board (batch update — no individual PUT calls).
  - [ ] The board UI updates **optimistically**: the list order in the store is updated before the API call completes.
  - [ ] If the `reorderLists` API call fails, the original order is restored in the store and a `vue-sonner` error toast is shown to the user.
  - [ ] The component uses `<script setup lang="ts">` and Tailwind utility classes; no raw CSS.
  - [ ] No business logic lives in the `<template>` — drag-drop handler, edit, and delete logic are extracted into the composable or setup block.
- **Dependencies:** Task 016-C
- **Output Files:**
  - `app/components/card/list.vue`

---

## Wave 3 — Tests (Testing Agent)

### Task 016-E: API Endpoint Unit Tests

- **Agent:** Testing
- **Spec:** `specs/client-board-list-management.md`
- **Section:** Acceptance Criteria → all six endpoint ACs; Edge Cases
- **Acceptance Criteria:**
  - [ ] `GET /api/client/board-list/[boardId]` — happy path returns 200 with ordered `BoardListModel[]`; empty board returns `[]`; non-member returns 403; unknown board returns 404; unauthenticated returns 401.
  - [ ] `GET /api/client/board-list/show/[id]` — happy path returns 200 with nested cards and card-users; unknown list returns 404; non-member returns 403.
  - [ ] `POST /api/client/board-list/create/[boardId]` — happy path returns 201 with created model; invalid body returns 422 with field errors; non-member returns 403; verifies `list_create` broadcast is called with correct payload.
  - [ ] `PUT /api/client/board-list/[id]` — happy path returns 200 with updated model; empty body returns 422; unknown list returns 404; non-member returns 403; verifies `list_update` broadcast is called.
  - [ ] `DELETE /api/client/board-list/[id]` — happy path returns 200; unknown list returns 404; non-member returns 403; verifies cascade delete of child cards; verifies `list_delete` broadcast is called with `{ id }`.
  - [ ] `POST /api/client/board-list/update-position/[id]` — happy path returns 200 with reordered array; empty `items` returns 422; foreign list ids return 403/404; verifies single-transaction update; verifies `list_position` broadcast is called with full array.
  - [ ] `transformBoardList()` unit test: given a database row and a cards array, returns the correctly shaped `BoardListModel`.
  - [ ] Zod schema unit tests: `createBoardListSchema`, `updateBoardListSchema`, `updatePositionSchema` — valid inputs pass, invalid inputs return the expected error paths.
- **Dependencies:** Task 016-A, Task 016-B
- **Output Files:**
  - `tests/unit/server/api/client/board-list/boardId.get.test.ts`
  - `tests/unit/server/api/client/board-list/show-id.get.test.ts`
  - `tests/unit/server/api/client/board-list/create-boardId.post.test.ts`
  - `tests/unit/server/api/client/board-list/id.put.test.ts`
  - `tests/unit/server/api/client/board-list/id.delete.test.ts`
  - `tests/unit/server/api/client/board-list/update-position.post.test.ts`
  - `tests/unit/server/transformers/boardList.transformer.test.ts`
  - `tests/unit/shared/schemas/boardList.schema.test.ts`

---

### Task 016-F: Store, Composable & E2E Drag-and-Drop Tests

- **Agent:** Testing
- **Spec:** `specs/client-board-list-management.md`
- **Section:** Acceptance Criteria → Drag-and-drop reorder; State Management; Centrifugo real-time events → Store mutation table
- **Acceptance Criteria:**
  - [ ] `boardListStore` unit test — `createList` appends to store; `updateList` merges fields; `deleteList` removes by id; `reorderLists` replaces array.
  - [ ] `boardListStore` socket handler unit test — `list_create` appends deduplicated; `list_update` merges into matching id; `list_delete` removes by id; `list_position` replaces array.
  - [ ] `use-board-list-api` composable unit test — each function calls the correct endpoint with the correct method and payload.
  - [ ] E2E test (Playwright): authenticated board member can drag a list column to a new position; after drop, the reordered order persists on page reload.
  - [ ] E2E test (Playwright): when the `update-position` API call fails (mocked error), the original column order is restored and a toast error message is visible.
- **Dependencies:** Task 016-C, Task 016-D
- **Output Files:**
  - `tests/unit/app/stores/boardList.test.ts`
  - `tests/unit/app/composables/api/use-board-list-api.test.ts`
  - `tests/e2e/board-list-drag-drop.spec.ts`
