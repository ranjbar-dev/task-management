# Tasks: Client Board Access & Kanban View
**Spec:** `specs/client-board-kanban.md`
**Status:** pending

---

## Overview

Implements the primary kanban workspace: four server-side board endpoints (list, show, overview, logs-table), client-side Pinia stores (`boardStore`, `boardListStore`), the `board` three-column layout, the main `[id].vue` kanban page, all card/layout components (list, draggable, item, empty, sidebar, members panel), real-time Centrifugo event handling, and the home-page redirect. Drag-and-drop uses `vue-draggable-plus`; live updates flow through `socketStore`.

Execution order is strict: Wave 1 (server routes + client data layer) → Wave 2 (UI) → Wave 3 (tests). Tasks within the same wave are independent and may run in parallel.

---

## Wave 1 — Data Layer (API Agent)

### Task 015-A: Server Board API Endpoints
- **Agent:** API
- **Spec:** `specs/client-board-kanban.md`
- **Section:** AC-1, AC-2, AC-3, AC-4, AC-5 — Board List/Show/Overview/Logs-Table Endpoints & Access Control
- **Acceptance Criteria:**
  - [ ] AC-1: `GET /api/client/board/all` returns member boards only (or all boards for admins); response shape is `id`, `name`, `status`, `icon` per entry; requires `auth` + `active-user` middleware; returns 401 if unauthenticated
  - [ ] AC-2: `GET /api/client/board/show/[id]` returns full board data; non-member + non-admin users receive 403; admin bypasses membership check
  - [ ] AC-3: `GET /api/client/board/overview/[id]` returns `{ users, board_lists }` with `cards` nested in each list; non-member + non-admin users receive 403; used by `boardListStore.getBoardLists()`
  - [ ] AC-4: `POST /api/client/board/logs-table/[id]` returns paginated activity log entries; accepts `SearchTableRequest` body; non-member + non-admin users receive 403
  - [ ] AC-5: All four endpoints enforce `auth` and `active-user` server middleware; non-member + non-admin users receive 403 on any board-scoped endpoint; admin users bypass membership checks on all endpoints
- **Dependencies:**
  - `tasks/002-database-schema.md` — `boards`, `boardUsers`, `boardLists`, `cards`, `cardUsers`, `cardLogs` tables
  - `tasks/004-authentication.md` — `auth` + `active-user` server middleware, JWT user extraction
  - `tasks/005-api-transformers.md` — `boardTransformer`, `boardListTransformer`, `cardTransformer`, `userTransformer`, `apiResponse`, `apiError`
  - `tasks/006-search-table-service.md` — `SearchTable` service for logs-table pagination/filtering
  - `tasks/007-audit-trail.md` — `getCombinedActivity()` (card log entries consumed by logs-table endpoint)
- **Output Files:**
  - `server/api/client/board/all.get.ts`
  - `server/api/client/board/show/[id].get.ts`
  - `server/api/client/board/overview/[id].get.ts`
  - `server/api/client/board/logs-table/[id].post.ts`

---

### Task 015-B: Client-Side Data Layer — Types, Composables & Pinia Stores
- **Agent:** API
- **Spec:** `specs/client-board-kanban.md`
- **Section:** Data Model → TypeScript Interfaces; State Management — `boardStore`, `boardListStore`; AC-6, AC-7 (store actions consumed by pages)
- **Acceptance Criteria:**
  - [ ] AC-1 (types): `BoardModel`, `BoardListModel`, `CardModel`, `UserModel`, `TagModel`, `SearchTableRequest` interfaces are defined in `shared/types/board.types.ts` matching the spec's TypeScript Interfaces section exactly
  - [ ] AC-6 (store): `boardStore` exposes `boards: BoardModel[]`, `privateBoardID: number | null`, `isLoading: boolean`
  - [ ] AC-6 (store): `boardStore.fetchBoards()` calls `GET /api/client/board/all` and populates `boards` and `privateBoardID`
  - [ ] AC-6 (store): `boardStore.updateBoardInList(board)` replaces a board by `id` in `boards`
  - [ ] AC-6 (store): `boardStore.removeBoardFromList(boardId)` removes a board by `id` from `boards`
  - [ ] AC-7 (store): `boardListStore` exposes `boardLists: BoardListModel[]`, `boardMembers: UserModel[]`, `boardData: BoardModel | null`, `isLoading: boolean`
  - [ ] AC-7 (store): `boardListStore.getBoardLists(boardId)` calls `GET /api/client/board/overview/[id]` and populates all three state properties
  - [ ] AC-7 (store): `boardListStore.updateMember(member)` updates or appends a member in `boardMembers` by `id`
  - [ ] Composable `app/composables/api/use-board-api.ts` wraps all four board endpoints as typed functions; composable has an explicit return type
  - [ ] All Pinia stores use Setup Store syntax (`defineStore` with `setup()` function), not Options API
- **Dependencies:**
  - Task 015-A (server routes must exist before composable can reference them)
  - `tasks/004-authentication.md` — auth state (`useAuthStore`) for identifying the current user
  - `tasks/005-api-transformers.md` — response shapes returned by the board endpoints
- **Output Files:**
  - `shared/types/board.types.ts`
  - `app/composables/api/use-board-api.ts`
  - `app/stores/board.ts`
  - `app/stores/boardList.ts`

---

## Wave 2 — Frontend (Frontend Agent)

### Task 015-C: Kanban Pages & Board Layout
- **Agent:** Frontend
- **Spec:** `specs/client-board-kanban.md`
- **Section:** AC-6 (home redirect), AC-7 (kanban page load), AC-15 (activity log modal), Component Tree
- **Acceptance Criteria:**
  - [ ] AC-6: `app/pages/index.vue` reads `boardStore.privateBoardID`; if set, navigates immediately to `/:privateBoardID`; if `null`, awaits `boardStore.fetchBoards()` before redirecting; if still `null` after fetch, shows an error or empty state
  - [ ] AC-7: `app/pages/[id].vue` declares `definePageMeta({ layout: 'board' })`
  - [ ] AC-7: On mount, calls `boardListStore.getBoardLists(boardId)` using the route param `id`
  - [ ] AC-7: While `boardListStore.isLoading` is `true`, a loading state is displayed
  - [ ] AC-7: A 403 response from the overview endpoint redirects the user away with an appropriate error toast (via `vue-sonner`)
  - [ ] AC-7: Lists are rendered in ascending `position` order; cards within each list are rendered in ascending `position` order
  - [ ] AC-15: A UI control on the board page opens an activity log modal
  - [ ] AC-15: The modal calls `POST /api/client/board/logs-table/[id]` with default pagination on open
  - [ ] AC-15: Log entries are displayed in reverse-chronological order with pagination controls inside the modal
  - [ ] `app/layouts/board.vue` implements a three-column layout: left sidebar (`Sidebar.vue`), centre content slot, right panel (`BoardMembers.vue`)
  - [ ] No business logic in `<template>` — all logic is extracted to composables or store actions
  - [ ] All components use `<script setup lang="ts">` with props typed via `defineProps<T>()`
- **Dependencies:**
  - Task 015-A (server endpoints)
  - Task 015-B (`boardStore`, `boardListStore`, `use-board-api.ts`, shared types)
- **Output Files:**
  - `app/pages/index.vue`
  - `app/pages/[id].vue`
  - `app/layouts/board.vue`

---

### Task 015-D: Kanban Card & Interactive Components
- **Agent:** Frontend
- **Spec:** `specs/client-board-kanban.md`
- **Section:** AC-8 (display), AC-9 (drag-and-drop), AC-10 (My Tasks filter), AC-11 (quick-add card), Component Tree
- **Acceptance Criteria:**
  - [ ] AC-8: `components/card/list.vue` renders a single kanban column: list name header, `Draggable.vue` cards container, and quick-add input
  - [ ] AC-8: `components/card/Draggable.vue` wraps `vue-draggable-plus` to contain the sortable card list
  - [ ] AC-8: `components/card/Item.vue` renders a single card tile showing at minimum: name, assigned user avatars, tags (with colour), and a status indicator
  - [ ] AC-8: `components/card/empty.vue` is shown inside a list when it has zero visible cards (zero total, or all filtered out)
  - [ ] AC-9: Cards can be dragged between any two lists and reordered within the same list using `vue-draggable-plus`
  - [ ] AC-9: After a successful drop, the store (`boardListStore.boardLists`) is updated optimistically; the position-update API call is fired (cross-references `tasks/017-client-card-management.md`)
  - [ ] AC-9: Drag handles are visually indicated and accessible
  - [ ] AC-10: A "My Tasks" toggle is available on the board page (inside `[id].vue` or `card/list.vue`)
  - [ ] AC-10: When active, only cards where the current user appears in `card.users` are displayed; lists with no matching cards show `card/empty.vue`
  - [ ] AC-10: Filter state is local to the page (not persisted to server or store)
  - [ ] AC-11: Each list column contains an inline quick-add input field
  - [ ] AC-11: Submitting (Enter key or confirm button) creates a card with the entered name via the card creation endpoint (cross-references `tasks/017-client-card-management.md`)
  - [ ] AC-11: On success, the new card appears at the bottom of the list; on failure, error toast shown via `vue-sonner`
  - [ ] AC-11: Submitting with empty or whitespace-only input is prevented with an inline validation message; no API call is fired
  - [ ] AC-11: Input clears and collapses after submission or Escape key press
- **Dependencies:**
  - Task 015-A (server endpoints)
  - Task 015-B (`boardListStore`, shared types)
  - Task 015-C (`board` layout must exist before components render within it)
- **Output Files:**
  - `app/components/card/list.vue`
  - `app/components/card/Draggable.vue`
  - `app/components/card/Item.vue`
  - `app/components/card/empty.vue`

---

### Task 015-E: Layout Components & Real-Time Integration
- **Agent:** Frontend
- **Spec:** `specs/client-board-kanban.md`
- **Section:** AC-12 (board members panel), AC-13 (sidebar navigation), AC-14 (real-time board events)
- **Acceptance Criteria:**
  - [ ] AC-12: `components/layout/BoardMembers.vue` renders all members of the active board from `boardListStore.boardMembers`; updates reactively when `boardMembers` changes
  - [ ] AC-13: `components/layout/Sidebar.vue` reads board list from `boardStore.boards`; each entry displays its icon (resolved via `BoardIconNameMap`) and name; clicking navigates to `/:id`; the currently active board is highlighted
  - [ ] AC-13: Sidebar is collapsible with GSAP animation; collapse/expand state is managed by `sidebarStore`
  - [ ] AC-13: Sidebar contains: user profile/avatar, board list, admin navigation (admin users only), logout button
  - [ ] AC-14: On mount of `[id].vue`, frontend subscribes to the `board#{id}` Centrifugo channel via `socketStore`; on unmount, the subscription is torn down
  - [ ] AC-14: `board_update` event updates `boardListStore.boardData`
  - [ ] AC-14: `board_delete` event navigates the user to `/`
  - [ ] AC-14: `list_create` event appends the new list to `boardListStore.boardLists`
  - [ ] AC-14: `list_update` event updates the matching list in `boardListStore.boardLists`
  - [ ] AC-14: `list_delete` event removes the matching list from `boardListStore.boardLists`
  - [ ] AC-14: `list_position` event reorders `boardListStore.boardLists` by new position values
  - [ ] AC-14: `card_create` event appends the card to the correct list's `.cards` array
  - [ ] AC-14: `card_update` event updates the matching card in its list
  - [ ] AC-14: `card_delete` event removes the matching card from its list
  - [ ] AC-14: `card_position` event reorders cards within a list or moves a card between lists
  - [ ] AC-14: `card_add_users` event appends users to the matching card's `.users` array
  - [ ] AC-14: `card_remove_users` event removes users from the matching card's `.users` array
  - [ ] AC-14: Real-time updates from other users do not interfere with an in-progress drag operation (in-flight drag takes precedence; real-time update is applied after drop completes)
  - [ ] `components/layout/PlanetOrbit.vue` decorative animated orbital element is present in the layout
- **Dependencies:**
  - Task 015-A (server endpoints)
  - Task 015-B (`boardStore`, `boardListStore`, shared types)
  - Task 015-C (`board` layout shell)
  - `tasks/010-realtime-centrifugo.md` — `socketStore` and `CentrifugoService` for channel subscription lifecycle
- **Output Files:**
  - `app/components/layout/Sidebar.vue`
  - `app/components/layout/BoardMembers.vue`
  - `app/components/layout/PlanetOrbit.vue`

---

## Wave 3 — Tests (Testing Agent)

### Task 015-F: Server API Endpoint Tests
- **Agent:** Testing
- **Spec:** `specs/client-board-kanban.md`
- **Section:** AC-1 through AC-5 — all four board endpoints + access control
- **Acceptance Criteria:**
  - [ ] AC-1: `all.get.ts` — happy path returns member boards only for regular users; admin receives all boards; 401 for unauthenticated requests
  - [ ] AC-2: `show/[id].get.ts` — happy path returns full board for member; 403 for non-member + non-admin; admin bypasses check
  - [ ] AC-3: `overview/[id].get.ts` — happy path returns `{ users, board_lists }` with nested cards; 403 for non-member + non-admin; admin bypasses check; cards and lists ordered by `position`
  - [ ] AC-4: `logs-table/[id].post.ts` — happy path returns paginated log entries; 403 for non-member + non-admin; validates `SearchTableRequest` body shape
  - [ ] AC-5: All four endpoints return 401 without a valid JWT; all four return 403 for non-member non-admin users
- **Dependencies:**
  - Task 015-A (implementation must exist)
  - Task 015-B (shared types for response shape assertions)
  - `tasks/002-database-schema.md`, `tasks/004-authentication.md`
- **Output Files:**
  - `tests/unit/server/api/client/board/all.get.test.ts`
  - `tests/unit/server/api/client/board/show/[id].get.test.ts`
  - `tests/unit/server/api/client/board/overview/[id].get.test.ts`
  - `tests/unit/server/api/client/board/logs-table/[id].post.test.ts`

---

### Task 015-G: Component Tests & E2E Kanban Flow
- **Agent:** Testing
- **Spec:** `specs/client-board-kanban.md`
- **Section:** AC-6 through AC-15 — all frontend behaviour including drag-drop, filter, quick-add, real-time events
- **Acceptance Criteria:**
  - [ ] AC-6: Unit test — `index.vue` redirects to `/:privateBoardID` when set; waits for `fetchBoards()` when `privateBoardID` is `null`
  - [ ] AC-7: Unit test — `[id].vue` shows loading state while `boardListStore.isLoading` is `true`; redirects with error toast on 403
  - [ ] AC-8: Component test — `card/Item.vue` renders name, user avatars, tags, and status indicator from props
  - [ ] AC-8: Component test — `card/list.vue` renders `card/empty.vue` when `cards` array is empty
  - [ ] AC-9: E2E — drag a card from one list to another; verify the card appears in the target list; verify the position-update API is called
  - [ ] AC-10: Component test — "My Tasks" filter toggle shows only cards assigned to the current user; columns with no matching cards show `card/empty.vue`
  - [ ] AC-11: Component test — quick-add with valid name fires card creation and appends card to the list; whitespace-only input is blocked; Escape collapses the input without API call
  - [ ] AC-12: Component test — `BoardMembers.vue` renders the member list from `boardListStore.boardMembers` reactively
  - [ ] AC-13: Component test — `Sidebar.vue` renders board entries with icon and name; active board is highlighted; clicking navigates to `/:id`
  - [ ] AC-14: Unit test — `board_delete` WebSocket event triggers navigation to `/`; `card_create` appends card to the correct list; `list_position` reorders `boardListStore.boardLists`; in-progress drag blocks concurrent `card_position` real-time update
  - [ ] AC-15: E2E — opening the activity log modal triggers the logs-table API call with default pagination; entries display in reverse-chronological order; pagination controls advance the page
  - [ ] Edge case: navigating to `/:id` for a non-member board shows error toast and redirects to `/`
  - [ ] Edge case: board overview returns empty `board_lists` — renders empty board state
- **Dependencies:**
  - Task 015-A, 015-B, 015-C, 015-D, 015-E (all implementation must exist)
  - `tasks/010-realtime-centrifugo.md` — mock `socketStore` for real-time event tests
- **Output Files:**
  - `tests/unit/app/pages/index.test.ts`
  - `tests/unit/app/pages/[id].test.ts`
  - `tests/unit/app/components/card/Item.test.ts`
  - `tests/unit/app/components/card/list.test.ts`
  - `tests/unit/app/components/layout/Sidebar.test.ts`
  - `tests/unit/app/components/layout/BoardMembers.test.ts`
  - `tests/e2e/kanban-drag-drop.spec.ts`
  - `tests/e2e/kanban-board.spec.ts`

---

## Cross-References

| Related Task File | Dependency Direction |
|---|---|
| `tasks/016-client-board-list-management.md` | List CRUD endpoints and `list_create/update/delete/position` events are consumed by Task 015-E handlers — implement 016 before wiring real-time list mutations |
| `tasks/017-client-card-management.md` | Card creation endpoint (AC-11 quick-add) and card position endpoint (AC-9 drag-drop) are defined in 017 — implement 017 before Task 015-D fires those calls |
| `tasks/010-realtime-centrifugo.md` | `socketStore` and `CentrifugoService` must exist (Task 015-E depends on it) |
| `tasks/021-frontend-architecture.md` | `sidebarStore` (AC-13 collapse) is defined there; `board` layout conventions are governed there |
