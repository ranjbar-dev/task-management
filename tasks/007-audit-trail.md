# Tasks: Audit Trail & CardLog System
**Spec:** `specs/audit-trail.md`
**Status:** pending

## Overview

Implements the complete audit trail system for the task management application. This covers the `card_logs` Drizzle schema, the `createCardLog()` / `getCombinedActivity()` server utilities, integration of log calls into card/comment/attachment route handlers, two board-level log API endpoints (admin + client), the `CommentAndActivity.vue` component that renders the combined activity feed in the card detail modal, and Centrifugo real-time broadcasting of every new log entry. No new Pinia store is required — activity data is fetched as part of the existing card detail response.

---

## Wave 1 — Data Layer (API Agent)

### Task 007-A: `card_logs` Drizzle Schema + Shared Types
- **Agent:** API
- **Spec:** `specs/audit-trail.md`
- **Section:** AC-8; Data Model → Database Schema; TypeScript Interfaces
- **Acceptance Criteria:**
  - [ ] AC-8: `card_logs` table exists with columns: `id` (serial PK), `user_id` (integer NOT NULL FK → `users.id`), `card_id` (integer NOT NULL FK → `cards.id`), `board_id` (integer NOT NULL FK → `boards.id`), `type` (smallint NOT NULL), `data` (jsonb NOT NULL DEFAULT `{}`), `created_at` (timestamptz NOT NULL DEFAULT now()), `updated_at` (timestamptz NOT NULL DEFAULT now())
  - [ ] `type` smallint semantics documented: `0` = card operation, `1` = comment operation, `2` = attachment operation
  - [ ] `data` column is typed via Drizzle `.$type<{ message?: string; changes?: Record<string, any>; old_data?: Record<string, any> }>()`
  - [ ] `cardLogs` table is exported from `server/database/schema/cardLogs.ts` and re-exported from the schema index
  - [ ] `CardLogModel` interface is defined in `shared/types/cardLog.types.ts` with fields: `id`, `user_id`, `card_id`, `board_id`, `type: 0 | 1 | 2`, `data`, `createdAt`, `updatedAt`
  - [ ] `ActivityItem` interface is defined in `shared/types/cardLog.types.ts` with discriminated union fields for `type: 'comment' | 'log'` items
- **Dependencies:** `tasks/002-database-schema.md`
- **Output Files:**
  - `server/database/schema/cardLogs.ts`
  - `shared/types/cardLog.types.ts`

---

### Task 007-B: `createCardLog()` + `getCombinedActivity()` Utility
- **Agent:** API
- **Spec:** `specs/audit-trail.md`
- **Section:** AC-4; AC-5; AC-6; Component Tree (`server/utils/cardLog.ts`)
- **Acceptance Criteria:**
  - [ ] AC-4: `createCardLog()` stores `userId` from `event.context.user` on every log entry
  - [ ] AC-4: `createCardLog()` stores `cardId` of the affected card on every log entry
  - [ ] AC-4: `createCardLog()` resolves and stores `boardId` server-side via `cards → boardLists → boards` join (never accepted as a client-supplied value)
  - [ ] AC-4: Every log entry receives a `data.message` human-readable string describing the action
  - [ ] AC-4: Update calls supply `data.changes` as a `Record<string, any>` before/after diff
  - [ ] AC-4: Delete calls supply `data.old_data` as a full snapshot of the record prior to deletion
  - [ ] AC-5: `createCardLog()` inserts the log row and then calls `broadcastToCentrifugo()` targeting channel `card#{cardId}` with event name `log_create` and the full inserted row (from `.returning()`) as payload
  - [ ] AC-5: If `broadcastToCentrifugo()` throws, the error is caught and logged but does NOT cause `createCardLog()` to reject or throw — the log row has already been persisted
  - [ ] AC-6: `getCombinedActivity(cardId)` fetches comments (with `user` + `attachment` relations) and `cardLogs` (with `user` relation) for the given card in parallel
  - [ ] AC-6: The merged result is sorted chronologically by `createdAt` ascending
  - [ ] AC-6: The combined result is returned as `ActivityItem[]`
- **Dependencies:** `Task 007-A`, `tasks/010-realtime-centrifugo.md`
- **Output Files:**
  - `server/utils/cardLog.ts`

---

### Task 007-C: `createCardLog()` Integration — Card Route Handlers
- **Agent:** API
- **Spec:** `specs/audit-trail.md`
- **Section:** AC-1; Data Model → Log Type Enum (type 0)
- **Acceptance Criteria:**
  - [ ] AC-1: `createCardLog()` is called in the card `create` route handler with `type: 0` and a `data.message` describing the creation
  - [ ] AC-1: `createCardLog()` is called in the card `update` route handler with `type: 0`; `data.changes` is populated with a before/after diff of the mutated fields
  - [ ] AC-1: `createCardLog()` is called in the card `delete` route handler with `type: 0`; `data.old_data` is populated with the full card snapshot prior to deletion
- **Dependencies:** `Task 007-B`, `tasks/017-client-card-management.md`
- **Output Files:**
  - `server/api/client/card/create.post.ts` *(modified)*
  - `server/api/client/card/[id].put.ts` *(modified)*
  - `server/api/client/card/[id].delete.ts` *(modified)*

---

### Task 007-D: `createCardLog()` Integration — Comment Route Handlers
- **Agent:** API
- **Spec:** `specs/audit-trail.md`
- **Section:** AC-2; Data Model → Log Type Enum (type 1)
- **Acceptance Criteria:**
  - [ ] AC-2: `createCardLog()` is called in the comment `create` route handler with `type: 1` and a `data.message` describing the action
  - [ ] AC-2: `createCardLog()` is called in the comment `update` route handler with `type: 1` and a `data.message` describing the edit
  - [ ] AC-2: `createCardLog()` is called in the comment `delete` route handler with `type: 1` and a `data.message` describing the deletion
- **Dependencies:** `Task 007-B`, `tasks/018-client-comments.md`
- **Output Files:**
  - `server/api/client/comment/create.post.ts` *(modified)*
  - `server/api/client/comment/[id].put.ts` *(modified)*
  - `server/api/client/comment/[id].delete.ts` *(modified)*

---

### Task 007-E: `createCardLog()` Integration — Attachment Route Handlers
- **Agent:** API
- **Spec:** `specs/audit-trail.md`
- **Section:** AC-3; Data Model → Log Type Enum (type 2)
- **Acceptance Criteria:**
  - [ ] AC-3: `createCardLog()` is called in the attachment `create` route handler with `type: 2` and a `data.message` describing the upload
  - [ ] AC-3: `createCardLog()` is called in the attachment `update` route handler with `type: 2` and a `data.message` describing the update
  - [ ] AC-3: `createCardLog()` is called in the attachment `delete` route handler with `type: 2` and a `data.message` describing the deletion
- **Dependencies:** `Task 007-B`, `tasks/019-client-attachments.md`
- **Output Files:**
  - `server/api/client/attachment/create.post.ts` *(modified)*
  - `server/api/client/attachment/[id].put.ts` *(modified)*
  - `server/api/client/attachment/[id].delete.ts` *(modified)*

---

### Task 007-F: Board-Level Log API Endpoints
- **Agent:** API
- **Spec:** `specs/audit-trail.md`
- **Section:** AC-7; Data Model → API Endpoints Consumed
- **Acceptance Criteria:**
  - [ ] AC-7: `POST /api/admin/board/logs-table/[id]` exists, returns paginated/filterable logs for the given board, and is protected by admin middleware (`admin.ts`)
  - [ ] AC-7: `POST /api/client/board/logs-table/[id]` exists, returns paginated/filterable logs for the given board, and is protected by auth middleware + board membership check
  - [ ] AC-7: Both endpoints delegate pagination and filtering to the `SearchTable` service (`server/services/searchTable.ts`)
  - [ ] AC-7: Both endpoints filter `card_logs` by `board_id` matching the route `[id]` param
  - [ ] Both responses include the `user` relation on each log row (for display in the UI table)
- **Dependencies:** `Task 007-A`, `tasks/004-authentication.md`, `tasks/006-search-table-service.md`
- **Output Files:**
  - `server/api/admin/board/logs-table/[id].post.ts`
  - `server/api/client/board/logs-table/[id].post.ts`

---

## Wave 2 — Frontend (Frontend Agent)

### Task 007-G: CommentAndActivity Component + Real-Time Activity Update
- **Agent:** Frontend
- **Spec:** `specs/audit-trail.md`
- **Section:** AC-5 (sub-item 5); AC-6; Component Tree (`app/components/card-board/CommentAndActivity.vue`); State Management
- **Acceptance Criteria:**
  - [ ] AC-6: `CommentAndActivity.vue` renders the `ActivityItem[]` array returned by `getCombinedActivity()` (via the card show response) in chronological order ascending
  - [ ] AC-6: Comment items (`type: 'comment'`) display `body`, `user`, optional `attachment`, and `pin` indicator
  - [ ] AC-6: Log items (`type: 'log'`) display `data.message`, `user`, `logType`, and `createdAt` timestamp
  - [ ] AC-5: The `socketEventHandler` composable subscribes to `log_create` events on the `card#{cardId}` channel and updates the activity timeline (either by appending the new item directly from the broadcast payload or by re-fetching the combined activity)
  - [ ] No dedicated Pinia store for card logs — activity state is owned by the existing card composable/store
  - [ ] Component is integrated into the card detail modal
- **Dependencies:** `Task 007-B`, `tasks/010-realtime-centrifugo.md`, `tasks/021-frontend-architecture.md`
- **Output Files:**
  - `app/components/card-board/CommentAndActivity.vue`

---

## Wave 3 — Tests (Testing Agent)

### Task 007-H: Audit Trail Tests
- **Agent:** Testing
- **Spec:** `specs/audit-trail.md`
- **Section:** All ACs
- **Acceptance Criteria:**
  - [ ] Unit test: `createCardLog()` inserts a row with correct `userId`, `cardId`, `boardId`, `type`, and `data` fields
  - [ ] Unit test: `createCardLog()` calls `broadcastToCentrifugo()` with channel `card#{cardId}`, event `log_create`, and the inserted row as payload after a successful insert
  - [ ] Unit test: `createCardLog()` does not throw when `broadcastToCentrifugo()` rejects — the function resolves normally
  - [ ] Unit test: `getCombinedActivity(cardId)` merges comments and cardLogs, sorts by `createdAt` ascending, and returns `ActivityItem[]`
  - [ ] Unit test: `getCombinedActivity(cardId)` fetches comments and cardLogs in parallel (both DB calls are initiated before either resolves)
  - [ ] API test: `POST /api/admin/board/logs-table/[id]` returns `401` for unauthenticated requests and `403` for non-admin users
  - [ ] API test: `POST /api/admin/board/logs-table/[id]` returns paginated logs filtered by `board_id` for admin users
  - [ ] API test: `POST /api/client/board/logs-table/[id]` returns `401` for unauthenticated requests and `403` for non-members
  - [ ] API test: `POST /api/client/board/logs-table/[id]` returns paginated logs filtered by `board_id` for board members
- **Dependencies:** `Task 007-A`, `Task 007-B`, `Task 007-C`, `Task 007-D`, `Task 007-E`, `Task 007-F`, `Task 007-G`
- **Output Files:**
  - `tests/unit/server/utils/card-log.test.ts`
  - `tests/unit/server/api/admin/board/logs-table.test.ts`
  - `tests/unit/server/api/client/board/logs-table.test.ts`
