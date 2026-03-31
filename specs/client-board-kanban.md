# Spec: Client Board Access & Kanban View

## Status: Approved

## Purpose

Provide authenticated users with a kanban board view of their assigned boards. Users can view, interact with, and manage cards across lists in real time. The board page is the primary workspace for the application — it supports drag-and-drop reordering, task filtering, quick card creation, and live updates via Centrifugo WebSocket. Admin users bypass member restrictions and can access any board.

---

## User Stories

- As a **member user**, I want to see all boards I belong to listed in the sidebar so I can navigate between them.
- As a **member user**, I want to open a board and see its lists and cards in a kanban layout so I can track work.
- As a **member user**, I want to filter the board to show only cards assigned to me so I can focus on my tasks.
- As a **member user**, I want to quickly add a card to any list by typing a name inline so I don't have to open a full form.
- As a **member user**, I want to drag cards between lists so I can update their status visually.
- As a **member user**, I want to see other board members in a right panel so I know who is on this board.
- As a **member user**, I want the board to update in real time when another user makes changes so I always see current state.
- As an **admin user**, I want to access any board regardless of membership so I can manage the workspace.
- As any **authenticated user**, I want visiting `/` to redirect me to my private board so I land in a useful starting point.

---

## Acceptance Criteria

### AC-1: Board List Endpoint (`GET /api/client/board/all`)
- [ ] Returns only boards where the current user is a member (`boardUsers` junction)
- [ ] Admin users receive all boards in the system
- [ ] Response includes: `id`, `name`, `status`, `icon` per board
- [ ] Requires `auth` + `active-user` middleware; returns 401 if unauthenticated

### AC-2: Board Show Endpoint (`GET /api/client/board/show/[id]`)
- [ ] Returns full board data for the given `id`
- [ ] Non-member, non-admin users receive a 403 response
- [ ] Admin users can retrieve any board

### AC-3: Board Overview Endpoint (`GET /api/client/board/overview/[id]`)
- [ ] Returns the complete board state: `users`, `lists`, `cards`, `card.users`
- [ ] Non-member, non-admin users receive a 403 response
- [ ] Cards are nested within their respective lists
- [ ] Used by `boardListStore.getBoardLists()` to populate the kanban view

### AC-4: Board Logs Table Endpoint (`POST /api/client/board/logs-table/[id]`)
- [ ] Returns paginated activity log entries for the given board
- [ ] Accepts SearchTable filter/pagination/sort parameters in request body
- [ ] Non-member, non-admin users receive a 403 response

### AC-5: Access Control
- [ ] All four endpoints enforce the `auth` and `active-user` server middleware
- [ ] A user who is not a board member and not an admin receives a 403 when accessing any board-scoped endpoint
- [ ] Admin users bypass membership checks on all board endpoints

### AC-6: Home Page Redirect (`/`)
- [ ] `app/pages/index.vue` reads `boardStore.privateBoardID`
- [ ] If `privateBoardID` is set, immediately navigates to `/:privateBoardID`
- [ ] If `privateBoardID` is `null` (not yet loaded), waits for `boardStore.fetchBoards()` to resolve before redirecting

### AC-7: Kanban Page Load (`/:id`)
- [ ] `app/pages/[id].vue` uses the `board` layout
- [ ] On mount, calls `boardListStore.getBoardLists(boardId)` which hits `GET /api/client/board/overview/[id]`
- [ ] While loading, a loading state is shown; `boardListStore.isLoading` drives this
- [ ] On 403 response, user is redirected away with an appropriate error message
- [ ] Lists are rendered in ascending `position` order
- [ ] Cards within each list are rendered in ascending `position` order

### AC-8: Kanban Display
- [ ] Each list is rendered as a `components/card/list.vue` column
- [ ] Each list renders its cards via `components/card/Draggable.vue` and `components/card/Item.vue`
- [ ] A list with zero cards shows `components/card/empty.vue` placeholder
- [ ] Card items show at minimum: name, assigned users (avatars), tags, status indicator

### AC-9: Drag-and-Drop Reordering
- [ ] Cards can be dragged between any two lists using `vue-draggable-plus`
- [ ] Cards can be reordered within the same list
- [ ] After a successful drop, the store is updated optimistically; the position update API call is fired (defined in the card management spec)
- [ ] Drag handles are accessible and visually indicated

### AC-10: "My Tasks" Filter
- [ ] A filter toggle is available on the board page
- [ ] When active, only cards where the current user appears in `card.users` are displayed
- [ ] Lists with no matching cards show the empty state component while the filter is active
- [ ] Filter state is local to the page (not persisted to server or store)

### AC-11: Quick-Add Card
- [ ] Each list column contains an inline input field for quick card creation
- [ ] Submitting the input (Enter key or confirm button) creates a card with the entered name in that list (calls the card creation endpoint defined in the card management spec)
- [ ] On success, the new card appears at the bottom of the list
- [ ] On failure, an error toast is shown via `vue-sonner`
- [ ] The input clears and collapses after submission or cancellation (Escape key)

### AC-12: Board Members Panel
- [ ] `components/layout/BoardMembers.vue` renders in the right panel of the `board` layout
- [ ] Displays all members of the currently active board from `boardListStore.boardMembers`
- [ ] Updates reactively when `boardListStore.boardMembers` changes (e.g., via real-time event)

### AC-13: Sidebar Navigation
- [ ] `components/layout/Sidebar.vue` renders in the left panel of the `board` layout
- [ ] Reads board list from `boardStore.boards`
- [ ] Each board entry displays its icon (resolved via `BoardIconNameMap`) and name
- [ ] Clicking a board entry navigates to `/:id`
- [ ] The currently active board is highlighted
- [ ] Sidebar is collapsible; collapse/expand is animated via GSAP and state is managed by `sidebarStore`
- [ ] Sidebar contains: user profile/avatar, board list, admin navigation (admin users only), logout button

### AC-14: Real-Time Board Events
- [ ] On mount of `[id].vue`, frontend subscribes to the `board#{id}` Centrifugo channel
- [ ] On unmount, the subscription is torn down
- [ ] The following events are handled:

| Event | Action |
|---|---|
| `board_update` | Update board info in `boardListStore.boardData` |
| `board_delete` | Navigate user away from the board (e.g., to `/`) |
| `list_create` | Append new list to `boardListStore.boardLists` |
| `list_update` | Update matching list in `boardListStore.boardLists` |
| `list_delete` | Remove matching list from `boardListStore.boardLists` |
| `list_position` | Reorder `boardListStore.boardLists` by new position values |
| `card_create` | Append card to the correct list's `.cards` array |
| `card_update` | Update matching card in its list |
| `card_delete` | Remove matching card from its list |
| `card_position` | Reorder cards within a list (or move between lists) |
| `card_add_users` | Append users to the matching card's `.users` array |
| `card_remove_users` | Remove users from the matching card's `.users` array |

- [ ] Real-time updates from other users do not interfere with an in-progress drag operation

### AC-15: Activity Log Modal
- [ ] A UI control on the board page opens an activity log modal
- [ ] The modal calls `POST /api/client/board/logs-table/[id]` with default pagination on open
- [ ] Log entries are displayed in reverse-chronological order
- [ ] Pagination controls are available within the modal

---

## Component Tree / File Structure

```
app/
├── pages/
│   ├── index.vue                    # Redirect to privateBoardID
│   └── [id].vue                     # Main kanban board page
├── layouts/
│   └── board.vue                    # Three-column layout: sidebar | content | members
├── components/
│   ├── card/
│   │   ├── list.vue                 # Single kanban column (list name + cards + quick-add)
│   │   ├── Draggable.vue            # vue-draggable-plus cards container
│   │   ├── Item.vue                 # Individual card tile
│   │   └── empty.vue                # Empty list placeholder
│   └── layout/
│       ├── Sidebar.vue              # Left sidebar: profile, board nav, admin links, logout
│       ├── BoardMembers.vue         # Right panel: member list for active board
│       └── PlanetOrbit.vue          # Decorative animated orbital element
└── stores/
    ├── board.ts                     # boardStore — board list for sidebar navigation
    └── boardList.ts                 # boardListStore — active board overview (lists/cards/members)

server/api/client/board/
├── all.get.ts                       # GET  /api/client/board/all
├── show/
│   └── [id].get.ts                  # GET  /api/client/board/show/[id]
├── overview/
│   └── [id].get.ts                  # GET  /api/client/board/overview/[id]
└── logs-table/
    └── [id].post.ts                 # POST /api/client/board/logs-table/[id]
```

---

## Data Model

### API Endpoints Consumed

| Method | Endpoint | Request | Response |
|--------|----------|---------|----------|
| GET | `/api/client/board/all` | — | `BoardModel[]` (id, name, status, icon) |
| GET | `/api/client/board/show/[id]` | — | `BoardModel` (full) |
| GET | `/api/client/board/overview/[id]` | — | `{ users: UserModel[], board_lists: BoardListModel[] }` (cards nested) |
| POST | `/api/client/board/logs-table/[id]` | `SearchTableRequest` | `{ data: LogEntry[], total: number, per_page: number, current_page: number }` |

### TypeScript Interfaces

```typescript
// shared/types/board.types.ts

interface BoardModel {
  id: number
  user_id: number
  name: string
  description: string
  status: number           // BoardStatusEnum
  icon: number             // BoardIconEnum
  icon_name?: string
  members: UserModel[]
  board_lists: BoardListModel[]
}

interface BoardListModel {
  id: number
  board_id: number
  name: string
  position: number
  cards: CardModel[]
}

interface CardModel {
  id: number
  board_list_id: number
  name: string
  position: number
  status: number           // CardStatusEnum
  tags: TagModel[]
  users: UserModel[]       // Assigned members
}

interface UserModel {
  id: number
  name: string
  email: string
  avatar: string | null
  role: number             // UserRoleEnum
}

interface TagModel {
  id: number
  name: string
  color: string
}

// SearchTable request body (shared)
interface SearchTableRequest {
  page: number
  per_page: number
  sort_by?: string
  sort_dir?: 'asc' | 'desc'
  filters?: Record<string, unknown>
}
```

---

## State Management

### `boardStore` (`app/stores/board.ts`)

| Property | Type | Description |
|---|---|---|
| `boards` | `BoardModel[]` | All boards accessible to the current user |
| `privateBoardID` | `number \| null` | ID of the user's private board (used for home redirect) |
| `isLoading` | `boolean` | Loading state for board list fetch |

| Action | Description |
|---|---|
| `fetchBoards()` | Calls `GET /api/client/board/all`; admin gets all boards, normal user gets member boards |
| `updateBoardInList(board)` | Replaces a board entry by id in `boards` |
| `removeBoardFromList(boardId)` | Removes a board entry by id from `boards` |

### `boardListStore` (`app/stores/boardList.ts`)

| Property | Type | Description |
|---|---|---|
| `boardLists` | `BoardListModel[]` | Ordered list of kanban columns with nested cards |
| `boardMembers` | `UserModel[]` | Members of the currently viewed board |
| `boardData` | `BoardModel \| null` | Metadata of the currently viewed board |
| `isLoading` | `boolean` | Loading state for overview fetch |

| Action | Description |
|---|---|
| `getBoardLists(boardId)` | Calls `GET /api/client/board/overview/[id]`; populates all three state properties |
| `updateMember(member)` | Updates or appends a member in `boardMembers` by id |

### `sidebarStore` (`app/stores/sidebar.ts`)

Manages sidebar collapsed/expanded state. Used by `Sidebar.vue` to drive GSAP transitions. (Defined in full in the frontend-architecture spec.)

---

## Edge Cases

| Scenario | Expected Behaviour |
|---|---|
| User navigates to `/:id` of a board they are not a member of | Server returns 403; page redirects to `/` with an error toast |
| Board is deleted while user is viewing it | `board_delete` WebSocket event triggers navigation to `/` |
| `privateBoardID` is `null` when landing on `/` | Wait for `fetchBoards()` before redirecting; if still null after fetch, show an error or empty state |
| Centrifugo connection drops while on board page | Reconnection is handled by `CentrifugoService` (see realtime-centrifugo spec); store state may be stale — show a reconnecting indicator |
| Drag ends while a real-time `card_position` event arrives | In-flight drag takes precedence; real-time update is applied after drop completes |
| List has no cards and filter is active | Show `card/empty.vue` in that column |
| Quick-add submitted with empty or whitespace-only input | Prevent API call; show inline validation message |
| Board overview returns an empty `board_lists` array | Render an empty board state (no columns) with guidance |
| User is an admin but not explicitly a board member | Admin bypasses membership check at the server; board is accessible and the admin appears in `boardMembers` only if explicitly added |

---

## Non-Goals

- Creating, editing, renaming, or deleting lists (covered in `client-board-list-management.md`)
- Full card detail view / editing card fields (covered in `client-card-management.md`)
- Card member assignment or status updates (covered in `client-card-members-status.md`)
- Comments and attachments on cards (covered in `client-comments.md` and `client-attachments.md`)
- Admin board CRUD or member sync (covered in `admin-board-management.md`)
- Notification delivery or preference management
- User profile editing

---

## Dependencies

| Dependency | Spec |
|---|---|
| Authentication middleware (`auth`, `active-user`) | `specs/authentication.md` |
| Drizzle schemas: `boards`, `boardUsers`, `boardLists`, `cards`, `cardUsers` | `specs/database-schema.md` |
| Board/list/card/user transformers | `specs/api-transformers.md` |
| SearchTable service (logs-table endpoint) | `specs/search-table-service.md` |
| Centrifugo WebSocket real-time events | `specs/realtime-centrifugo.md` |
| Card create / position endpoints | `specs/client-card-management.md` |
| Board list position endpoints | `specs/client-board-list-management.md` |
| Frontend architecture (layouts, plugins, route middleware) | `specs/frontend-architecture.md` |
| Shared Zod validation schemas | `specs/validation-schemas.md` |
