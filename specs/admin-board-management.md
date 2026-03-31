# Spec: Admin Board Management

## Status: Approved

## Purpose

Provide administrators with full lifecycle control over boards: create, read, update, delete, member management, activity log viewing, and icon selection. All operations are protected by JWT authentication + admin role enforcement and broadcast real-time events to connected clients via Centrifugo. Notifications are dispatched to affected board members on destructive or membership-changing actions.

---

## User Stories

- As an admin, I can view a paginated, searchable table of all boards so I can quickly find and manage any board in the system.
- As an admin, I can create a new board with a name, description, status, and icon so that it is available to assign members to.
- As an admin, I can update a board's name, description, status, or icon so that board metadata stays accurate.
- As an admin, I can delete a board so that obsolete boards are removed and members are notified.
- As an admin, I can view a board overview (including its lists, cards, and members) so I can inspect full board contents.
- As an admin, I can view a paginated activity log for any board so I can audit what has happened on it.
- As an admin, I can sync board members (add/remove users in bulk) so that membership is always accurate.
- As an admin, I can retrieve the list of all boards (lightweight, with user counts) for use in dropdowns and summary views.
- As an admin, I can retrieve all available board icons so the UI can render icon pickers correctly.

---

## Acceptance Criteria

### AC-1 — List All Boards (`GET /api/admin/board/all`)
- [ ] Returns an array of all boards, each including `id`, `name`, `description`, `icon`, `icon_name`, `status`, and `users_count`.
- [ ] Response is produced by `transformBoard()`.
- [ ] Route is protected by `auth`, `active-user`, and `admin` middleware.
- [ ] Returns `200` with an empty array when no boards exist.

### AC-2 — Paginated Board Table (`POST /api/admin/board/table`)
- [ ] Accepts SearchTable service payload (filters, pagination, sort).
- [ ] Returns paginated board rows with total count.
- [ ] Sortable and filterable by board name, status, and user count.
- [ ] Route is protected by `auth`, `active-user`, and `admin` middleware.
- [ ] Returns `400` on invalid SearchTable payload.

### AC-3 — Create Board (`POST /api/admin/board/create`)
- [ ] Validates request body against `createBoardSchema` (name: 1–50 chars, description: required, status: 0|1, icon: 0–14).
- [ ] Creates a new board row in the database.
- [ ] Returns `201` with the transformed board object.
- [ ] Returns `422` with field-level Zod errors on validation failure.
- [ ] Broadcasts a `board_create` event to the board channel via `broadcastToCentrifugo()`.
- [ ] Dispatches `CreateNewBoardNotification` to any members assigned at creation time.
- [ ] Route is protected by `auth`, `active-user`, and `admin` middleware.

### AC-4 — Update Board (`PUT /api/admin/board/[id]`)
- [ ] Validates request body against `updateBoardSchema` (all fields optional, same constraints as create).
- [ ] Returns `404` if the board does not exist.
- [ ] Returns `422` with field-level Zod errors on validation failure.
- [ ] Updates only the provided fields (partial update).
- [ ] Returns `200` with the updated transformed board object.
- [ ] Broadcasts a `board_update` event to the board channel via `broadcastToCentrifugo()`.
- [ ] If `status` is changed, dispatches `BoardStatusChangeNotification` to all current board members.
- [ ] If `name` or `description` is changed, dispatches `BoardInfoChangeNotification` to all current board members.
- [ ] Route is protected by `auth`, `active-user`, and `admin` middleware.

### AC-5 — Delete Board (`DELETE /api/admin/board/[id]`)
- [ ] Returns `404` if the board does not exist.
- [ ] Deletes the board and all cascaded records (lists, cards, logs, memberships) per DB cascade rules.
- [ ] Returns `200` (or `204`) on successful deletion.
- [ ] Dispatches `DeleteBoardNotification` to **all** board members before the board is deleted.
- [ ] Broadcasts a `board_delete` event to the board channel via `broadcastToCentrifugo()`.
- [ ] Route is protected by `auth`, `active-user`, and `admin` middleware.

### AC-6 — Board Overview (`GET /api/admin/board/overview/[id]`)
- [ ] Returns `404` if the board does not exist.
- [ ] Returns the full board object including: board fields, all `board_lists` (ordered), all `cards` within each list, and all `members`.
- [ ] Response is produced by `transformBoard()` with full relational data.
- [ ] Returns `200` with the transformed board overview.
- [ ] Route is protected by `auth`, `active-user`, and `admin` middleware.

### AC-7 — Board Activity Log Table (`POST /api/admin/board/logs-table/[id]`)
- [ ] Returns `404` if the board does not exist.
- [ ] Accepts SearchTable service payload (filters, pagination, sort).
- [ ] Returns paginated activity log entries for the specified board.
- [ ] Returns `400` on invalid SearchTable payload.
- [ ] Route is protected by `auth`, `active-user`, and `admin` middleware.

### AC-8 — Sync Board Members (`POST /api/admin/board/members/[id]`)
- [ ] Returns `404` if the board does not exist.
- [ ] Validates request body against `syncBoardMembersSchema` (`user_ids`: array of integers).
- [ ] Replaces the current member set with the provided `user_ids` (idempotent sync — add new, remove missing).
- [ ] Returns `422` on validation failure.
- [ ] Returns `200` with the updated board member list.
- [ ] Dispatches `JoinToBoardNotification` to each **newly added** user.
- [ ] Dispatches `RemoveFromBoardNotification` to each **removed** user.
- [ ] Broadcasts a `board_add_users` event to the board channel via `broadcastToCentrifugo()`.
- [ ] Route is protected by `auth`, `active-user`, and `admin` middleware.

### AC-9 — List Board Icons (`GET /api/admin/board/icons`)
- [ ] Returns the full `BoardIconEnum` map as an array of `{ value: number, key: string, icon_name: string }` objects (15 entries).
- [ ] Response is static/derived from `BoardIconEnum` and `BoardIconNameMap`.
- [ ] Returns `200` with the icon list.
- [ ] Route is protected by `auth`, `active-user`, and `admin` middleware.

### AC-10 — Frontend Admin Board Page
- [ ] Page `app/pages/board/index.vue` uses the `default` layout.
- [ ] Renders a `Table/Search` component backed by `POST /api/admin/board/table`.
- [ ] Columns include: icon, name, description, status badge, member count, actions.
- [ ] "Add Board" button opens the Board Add modal with the create form.
- [ ] Each row has a "View" action that opens the Board Details modal (calls overview endpoint).
- [ ] Each row has an "Edit" action that opens the Board Edit modal (pre-filled via row data).
- [ ] Each row has a "Delete" action that opens a confirmation modal; on confirm calls the delete endpoint.
- [ ] Each row has a "Members" action that opens the member sync modal (pre-filled with current member IDs).
- [ ] All mutations (create/update/delete/sync) trigger a table refresh.
- [ ] Icon picker renders all 15 icons from `GET /api/admin/board/icons` as selectable options.
- [ ] Status field uses `BoardStatusEnum` values (Active / Deactive).
- [ ] Toast notifications confirm success/failure of each action.

---

## Component Tree / File Structure

```
app/
└── pages/
    └── board/
        └── index.vue                        # Admin board list page (layout: default)

app/components/
└── board/
    ├── add-modal.vue                        # Create board form modal
    ├── edit-modal.vue                       # Edit board form modal (partial update)
    ├── details-modal.vue                    # Board overview modal (read-only)
    ├── delete-modal.vue                     # Delete confirmation modal
    ├── members-modal.vue                    # Member sync modal (multi-select users)
    └── icon-picker.vue                      # Icon selection sub-component

server/api/admin/board/
├── all.get.ts                               # GET  /api/admin/board/all
├── table.post.ts                            # POST /api/admin/board/table
├── create.post.ts                           # POST /api/admin/board/create
├── [id].put.ts                              # PUT  /api/admin/board/[id]
├── [id].delete.ts                           # DELETE /api/admin/board/[id]
├── icons.get.ts                             # GET  /api/admin/board/icons
├── overview/
│   └── [id].get.ts                          # GET  /api/admin/board/overview/[id]
├── logs-table/
│   └── [id].post.ts                         # POST /api/admin/board/logs-table/[id]
└── members/
    └── [id].post.ts                         # POST /api/admin/board/members/[id]

shared/
└── schemas/
    └── board.schema.ts                      # createBoardSchema, updateBoardSchema, syncBoardMembersSchema

server/transformers/
└── board.transformer.ts                     # transformBoard()
```

---

## Data Model

### API Endpoints Consumed

| Method   | Endpoint                               | Request Body                  | Response                                    |
|----------|----------------------------------------|-------------------------------|---------------------------------------------|
| `GET`    | `/api/admin/board/all`                 | —                             | `BoardModel[]`                              |
| `POST`   | `/api/admin/board/table`              | `SearchTablePayload`          | `{ data: BoardModel[], total: number }`     |
| `POST`   | `/api/admin/board/create`             | `CreateBoardRequest`          | `BoardModel`                                |
| `PUT`    | `/api/admin/board/[id]`               | `UpdateBoardRequest`          | `BoardModel`                                |
| `DELETE` | `/api/admin/board/[id]`               | —                             | `{ message: string }`                       |
| `GET`    | `/api/admin/board/overview/[id]`      | —                             | `BoardModel` (with lists, cards, members)   |
| `POST`   | `/api/admin/board/logs-table/[id]`    | `SearchTablePayload`          | `{ data: BoardLog[], total: number }`       |
| `POST`   | `/api/admin/board/members/[id]`       | `SyncBoardMembersRequest`     | `UserModel[]`                               |
| `GET`    | `/api/admin/board/icons`              | —                             | `BoardIconOption[]`                         |

### TypeScript Interfaces

```typescript
// Shared enums
export const BoardStatusEnum = {
  Deactive: 0,
  Active: 1,
} as const
export type BoardStatus = (typeof BoardStatusEnum)[keyof typeof BoardStatusEnum]

export const BoardIconEnum = {
  bolt: 0, box: 1, brush: 2, camera: 3, crown: 4,
  database: 5, euro: 6, folder: 7, home: 8, map: 9,
  mobile: 10, rocket: 11, star: 12, store: 13, tag: 14,
} as const
export type BoardIcon = (typeof BoardIconEnum)[keyof typeof BoardIconEnum]

export const BoardIconNameMap: Record<BoardIcon, string> = {
  0: 'bolt', 1: 'inventory_2', 2: 'brush', 3: 'photo_camera',
  4: 'crown', 5: 'database', 6: 'euro', 7: 'folder',
  8: 'home', 9: 'map', 10: 'smartphone', 11: 'rocket_launch',
  12: 'star', 13: 'store', 14: 'sell',
}

// Board domain
interface BoardModel {
  id: number
  user_id: number
  name: string
  description: string
  status: BoardStatus
  icon: BoardIcon
  icon_name: string              // resolved from BoardIconNameMap
  users_count?: number           // included in list/table responses
  members: UserModel[]           // included in overview + member sync response
  board_lists: BoardListModel[]  // included in overview response
}

interface CreateBoardRequest {
  name: string          // 1–50 chars, required
  description: string   // required
  status?: BoardStatus  // defaults to Active (1) if omitted
  icon?: BoardIcon      // defaults to 0 if omitted
}

type UpdateBoardRequest = Partial<CreateBoardRequest>

interface SyncBoardMembersRequest {
  user_ids: number[]
}

interface BoardIconOption {
  value: BoardIcon
  key: string       // e.g. "bolt"
  icon_name: string // e.g. "bolt" (Material Symbol name)
}
```

### Validation Schemas (`shared/schemas/board.schema.ts`)

```typescript
export const createBoardSchema = z.object({
  name:        z.string().min(1).max(50),
  description: z.string().min(1),
  status:      z.coerce.number().int().min(0).max(1).optional(),
  icon:        z.coerce.number().int().min(0).max(14).optional(),
})

export const updateBoardSchema = createBoardSchema.partial()

export const syncBoardMembersSchema = z.object({
  user_ids: z.array(z.number().int()),
})
```

---

## State Management

### Pinia Store: `app/stores/board.ts`

| State Key          | Type            | Description                                  |
|--------------------|-----------------|----------------------------------------------|
| `boards`           | `BoardModel[]`  | Cached list from `GET /api/admin/board/all`  |
| `currentBoard`     | `BoardModel \| null` | Board currently open in details/edit modal |
| `isLoading`        | `boolean`       | Global loading flag for board operations     |

**Actions:**
- `fetchAllBoards()` — calls `GET /api/admin/board/all`, populates `boards`
- `createBoard(payload)` — calls `POST /api/admin/board/create`, appends to `boards`
- `updateBoard(id, payload)` — calls `PUT /api/admin/board/[id]`, replaces entry in `boards`
- `deleteBoard(id)` — calls `DELETE /api/admin/board/[id]`, removes entry from `boards`
- `fetchBoardOverview(id)` — calls `GET /api/admin/board/overview/[id]`, sets `currentBoard`
- `syncMembers(id, userIds)` — calls `POST /api/admin/board/members/[id]`, updates `currentBoard.members`

### Composable: `app/composables/api/use-board-api.ts`

Wraps all board API calls with `$fetch` (mutations) and `useFetch` (reads). Components interact with the store via the composable — no direct `$fetch` in `.vue` files.

---

## Edge Cases

| Scenario | Expected Behaviour |
|---|---|
| Delete board with active members | `DeleteBoardNotification` sent to all members before deletion; responds `200` |
| Sync members with empty `user_ids: []` | All existing members removed; `RemoveFromBoardNotification` sent to each |
| Sync members with no changes | No notifications dispatched; `board_add_users` broadcast still fires |
| Update board with no changed fields | No notifications dispatched; `board_update` broadcast still fires; returns updated board |
| Invalid icon value (e.g., `icon: 99`) | Returns `422` — `z.coerce.number().int().min(0).max(14)` rejects |
| Non-existent board ID in any `[id]` route | Returns `404` with a descriptive error message |
| Admin deletes own board | Allowed — no special restriction; follows normal delete flow |
| Board name at max length (50 chars) | Accepted; 51 chars returns `422` |
| `user_ids` array contains non-integer | Returns `422` — `z.array(z.number().int())` rejects |
| Concurrent member sync requests | Last write wins — DB transaction ensures consistency |

---

## Non-Goals

- This spec does not cover client-facing board endpoints (see `client-board-kanban.md`).
- This spec does not cover card or board-list management within boards (see `client-card-management.md`, `client-board-list-management.md`).
- Role-based visibility rules for boards (boards are admin-managed only; clients access boards they are members of).
- Soft delete / archive boards — delete is permanent.
- Bulk-create or bulk-delete boards.
- Board export or import.

---

## Dependencies

| Dependency | Type | Notes |
|---|---|---|
| `auth` server middleware | Server | JWT validation via `jose`; blocks unauthenticated requests |
| `active-user` server middleware | Server | Blocks suspended/disabled users |
| `admin` server middleware | Server | Requires `role === UserRoleEnum.AdminUser` |
| `transformBoard()` transformer | Server | `server/transformers/board.transformer.ts` |
| `SearchTable` service | Server | Used by `table.post.ts` and `logs-table/[id].post.ts` |
| `broadcastToCentrifugo()` utility | Server | `server/utils/broadcast.ts` |
| `NotificationService` | Server | Dispatches all 5 board notification types |
| `BullMQ` notification job | Server | Async delivery of notifications (see `notification-system.md`) |
| `createBoardSchema` / `updateBoardSchema` / `syncBoardMembersSchema` | Shared | `shared/schemas/board.schema.ts` |
| `BoardStatusEnum` / `BoardIconEnum` / `BoardIconNameMap` | Shared | `shared/enums/` |
| `BoardModel` / `UserModel` / `BoardListModel` interfaces | Shared | `shared/types/` |
| `database-schema.md` spec | Spec | Defines `boards`, `board_users`, `board_lists`, `cards` Drizzle schemas |
| `notification-system.md` spec | Spec | Defines `DeleteBoardNotification`, `BoardStatusChangeNotification`, `BoardInfoChangeNotification`, `JoinToBoardNotification`, `RemoveFromBoardNotification`, `CreateNewBoardNotification` |
| `realtime-centrifugo.md` spec | Spec | Defines `board_create`, `board_update`, `board_delete`, `board_add_users` event contracts |
| `search-table-service.md` spec | Spec | Defines `SearchTablePayload` schema and pagination contract |
| `api-transformers.md` spec | Spec | Defines `transformBoard()` output contract |
