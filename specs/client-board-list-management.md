# Spec: Client BoardList Management

## Status: Approved

## Purpose

Provide full CRUD management of board lists (kanban columns) for authenticated board members. Covers fetching lists, creating/updating/deleting individual lists, batch-reordering list positions via drag-and-drop, and broadcasting real-time Centrifugo events to all connected board members after each mutation.

---

## User Stories

- As a board member, I want to see all lists (columns) for a board so I can view my kanban board.
- As a board member, I want to see a single list with its cards and card-member assignments so I can view detailed column content.
- As a board member, I want to create a new list on a board so I can add a new column to my workflow.
- As a board member, I want to rename or reposition a list so I can keep my board organised.
- As a board member, I want to delete a list so I can remove columns that are no longer needed.
- As a board member, I want to drag columns horizontally to reorder them so I can rearrange my workflow without manual input.
- As any other board member, I want changes (create / update / delete / reorder) made by a colleague to appear in real time so I always see the latest board state.

---

## Acceptance Criteria

### GET /api/client/board-list/[boardId] — Get all lists for a board
- [ ] Returns HTTP 200 with an array of `BoardListModel` objects for the specified board.
- [ ] Each list object includes `{ id, board_id, name, position, cards[] }` via `transformBoardList`.
- [ ] Lists are ordered by `position` ascending.
- [ ] Returns HTTP 403 if the authenticated user is not a member of the board (and not an admin).
- [ ] Returns HTTP 404 if the board does not exist.
- [ ] Requires `auth` and `active-user` middleware — unauthenticated requests return HTTP 401.

### GET /api/client/board-list/show/[id] — Get a single list with cards and card-users
- [ ] Returns HTTP 200 with a single `BoardListModel` including nested `cards[]` each with their assigned users.
- [ ] Returns HTTP 404 if the list does not exist.
- [ ] Returns HTTP 403 if the authenticated user is not a member of the list's parent board (and not an admin).
- [ ] Requires `auth` and `active-user` middleware.

### POST /api/client/board-list/create/[boardId] — Create a list
- [ ] Accepts `{ name: string, position: number }` in the request body (validated by `createBoardListSchema`).
- [ ] `name` must be 1–50 characters; `position` must be a non-negative integer.
- [ ] Returns HTTP 201 with the created `BoardListModel` via `transformBoardList`.
- [ ] Broadcasts a `list_create` event to the `board#{boardId}` Centrifugo channel with the new list payload.
- [ ] Returns HTTP 422 if validation fails, with field-level error details.
- [ ] Returns HTTP 403 if the user is not a board member (and not an admin).
- [ ] Requires `auth` and `active-user` middleware.

### PUT /api/client/board-list/[id] — Update a list
- [ ] Accepts a partial body `{ name?: string, position?: number }` validated by `updateBoardListSchema`.
- [ ] At least one field must be present; returns HTTP 422 if the body is empty or invalid.
- [ ] Returns HTTP 200 with the updated `BoardListModel` via `transformBoardList`.
- [ ] Broadcasts a `list_update` event to the `board#{boardId}` Centrifugo channel with the updated list payload.
- [ ] Returns HTTP 404 if the list does not exist.
- [ ] Returns HTTP 403 if the user is not a board member (and not an admin).
- [ ] Requires `auth` and `active-user` middleware.

### DELETE /api/client/board-list/[id] — Delete a list
- [ ] Deletes the specified list and all associated cards (cascade or explicit delete per schema constraints).
- [ ] Returns HTTP 200 with a success message or the deleted list id.
- [ ] Broadcasts a `list_delete` event to the `board#{boardId}` Centrifugo channel with `{ id }`.
- [ ] Returns HTTP 404 if the list does not exist.
- [ ] Returns HTTP 403 if the user is not a board member (and not an admin).
- [ ] Requires `auth` and `active-user` middleware.

### POST /api/client/board-list/update-position/[id] — Batch reorder lists
- [ ] Accepts `{ items: Array<{ id: number, position: number }> }` validated by `updatePositionSchema`.
- [ ] Updates the `position` column for every list id supplied in the `items` array in a single database transaction.
- [ ] Returns HTTP 200 with the updated list of `BoardListModel` objects (or a success flag).
- [ ] Broadcasts a `list_position` event to the `board#{boardId}` Centrifugo channel with the full reordered array.
- [ ] Returns HTTP 422 if `items` is empty or contains non-integer ids/positions.
- [ ] Returns HTTP 403 if the user is not a board member (and not an admin).
- [ ] Requires `auth` and `active-user` middleware.

### Drag-and-drop reorder (frontend)
- [ ] Lists are rendered as horizontally draggable columns using `vue-draggable-plus`.
- [ ] On a successful column drop, the frontend immediately calls `POST /api/client/board-list/update-position/[id]` with all affected `{ id, position }` pairs (batch update — not individual PUT calls).
- [ ] The board UI updates optimistically; if the API call fails the original order is restored and an error toast is shown.

### Centrifugo real-time events
- [ ] Every mutation (create / update / delete / reorder) emits the appropriate event on the `board#{boardId}` channel so all connected members see the change without refreshing.
- [ ] `list_create` — payload is the full new `BoardListModel`.
- [ ] `list_update` — payload is the updated `BoardListModel`.
- [ ] `list_delete` — payload contains at minimum `{ id: number }`.
- [ ] `list_position` — payload is the reordered array of `BoardListModel`.

---

## Component Tree / File Structure

```
server/api/client/board-list/
├── [boardId].get.ts              # GET all lists for a board
├── show/
│   └── [id].get.ts               # GET single list with cards + card-users
├── create/
│   └── [boardId].post.ts         # POST create list
├── [id].put.ts                   # PUT update list
├── [id].delete.ts                # DELETE list
└── update-position/
    └── [id].post.ts              # POST batch reorder positions

shared/schemas/
└── boardList.schema.ts           # createBoardListSchema, updateBoardListSchema, updatePositionSchema

server/transformers/
└── boardList.transformer.ts      # transformBoardList()

app/components/card/
└── list.vue                      # Kanban column component

app/stores/
└── boardList.ts                  # Pinia store — boardListStore

app/composables/api/
└── use-board-list-api.ts         # API composable wrapping all 6 endpoints
```

---

## Data Model

### API Endpoints Consumed

| Method | Endpoint | Request | Response |
|--------|----------|---------|----------|
| GET | `/api/client/board-list/[boardId]` | — | `BoardListModel[]` |
| GET | `/api/client/board-list/show/[id]` | — | `BoardListModel` (with nested cards + card-users) |
| POST | `/api/client/board-list/create/[boardId]` | `CreateBoardListRequest` | `BoardListModel` |
| PUT | `/api/client/board-list/[id]` | `UpdateBoardListRequest` | `BoardListModel` |
| DELETE | `/api/client/board-list/[id]` | — | `{ id: number }` |
| POST | `/api/client/board-list/update-position/[id]` | `UpdatePositionRequest` | `BoardListModel[]` |

### TypeScript Interfaces

```typescript
interface BoardListModel {
  id: number
  board_id: number
  name: string
  position: number
  cards: CardModel[]
}

interface CreateBoardListRequest {
  name: string      // min 1, max 50 chars
  position: number  // non-negative integer
}

interface UpdateBoardListRequest {
  name?: string
  position?: number
}

interface UpdatePositionRequest {
  items: Array<{ id: number; position: number }>
}
```

### Zod Validation Schemas

```typescript
// shared/schemas/boardList.schema.ts
export const createBoardListSchema = z.object({
  name:     z.string().min(1).max(50),
  position: z.number().int(),
})

export const updateBoardListSchema = createBoardListSchema.partial()

export const updatePositionSchema = z.object({
  items: z.array(
    z.object({
      id:       z.number().int(),
      position: z.number().int(),
    })
  ),
})
```

### Transformer

```typescript
// server/transformers/boardList.transformer.ts
// transformBoardList(boardListRow, { cards }) → BoardListModel
// Returns: { id, board_id, name, position, cards[] }
```

---

## State Management

**Store:** `app/stores/boardList.ts` (`boardListStore`)

| Action | Description |
|--------|-------------|
| `getBoardLists(boardId)` | Calls `GET /api/client/board-list/[boardId]` and populates the store with the ordered list array |
| `createList(boardId, payload)` | Calls `POST /api/client/board-list/create/[boardId]`, appends result to store |
| `updateList(id, payload)` | Calls `PUT /api/client/board-list/[id]`, merges result into store |
| `deleteList(id)` | Calls `DELETE /api/client/board-list/[id]`, removes entry from store |
| `reorderLists(id, items)` | Calls `POST /api/client/board-list/update-position/[id]`, replaces store order |

**Centrifugo socket events** (`socketStore` / `CentrifugoService`) must also update `boardListStore` when events arrive on the `board#{boardId}` channel:

| Event | Store mutation |
|-------|----------------|
| `list_create` | Append new list to store (deduplicated by id) |
| `list_update` | Merge updated fields into existing list in store |
| `list_delete` | Remove list by id from store |
| `list_position` | Replace full list array with reordered payload |

---

## Edge Cases

- **Duplicate position values** — The frontend must always send a contiguous, non-duplicate set of positions; the server should accept whatever integers are sent and persist them as-is (display order is derived by sorting `position` asc on retrieval).
- **Concurrent reorder conflicts** — Last write wins; no optimistic lock is applied. The Centrifugo `list_position` event will propagate the winning order to all clients.
- **Delete list with cards** — All child cards (and their attachments, comments, card-users) must be cleaned up. Confirm cascade behaviour is defined in the Drizzle schema (`onDelete: 'cascade'`).
- **Non-member attempting access** — Middleware membership check must run before any DB queries to avoid leaking list/card data.
- **Empty board** — `GET /api/client/board-list/[boardId]` returns an empty array `[]`, not 404.
- **Admin bypass** — Admin users are exempt from board-membership checks (admin middleware already ensures this at the route level for admin routes; for client routes, the membership check logic must explicitly allow admin role).
- **`update-position` with unknown list ids** — Server should validate that all ids in `items` belong to the specified board; return HTTP 403/404 if any id is foreign.

---

## Non-Goals

- Card CRUD is out of scope — see `specs/client-card-management.md`.
- Archiving or soft-deleting lists (delete is hard delete only).
- Per-list permission levels (access is board-level only).
- Pagination of lists within a board (all lists for a board are returned in one response).
- Undo/redo for drag-drop reorder.

---

## Dependencies

| Dependency | Purpose |
|------------|---------|
| `specs/database-schema.md` | `boardLists` table schema, `boardUsers` junction table, cascade rules |
| `specs/authentication.md` | `auth` and `active-user` server middleware, JWT verification |
| `specs/api-transformers.md` | `transformBoardList()` transformer contract |
| `specs/validation-schemas.md` | `createBoardListSchema`, `updateBoardListSchema`, `updatePositionSchema` |
| `specs/realtime-centrifugo.md` | `broadcastToCentrifugo()` utility, `board#{boardId}` channel, socket event handling in `socketStore` |
| `specs/client-card-management.md` | `CardModel` interface consumed inside `BoardListModel.cards[]` |
| `vue-draggable-plus` v4 | Horizontal drag-and-drop for kanban columns |
| BullMQ / Redis | Not directly used by boardList; inherited via platform infrastructure |
