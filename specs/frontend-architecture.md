# Spec: Frontend Architecture

## Status: Approved

---

## Purpose

Define the complete frontend architecture for the Nuxt 4 task management application — covering all pages, layouts, components, Pinia stores, composables, plugins, route middleware, and TypeScript types. This spec is the authoritative reference for any developer implementing or extending the client-side codebase.

---

## User Stories

- **As an unauthenticated user**, I am redirected to `/auth/log-in` when I attempt to access any protected route, so that only authenticated users can use the application.
- **As an authenticated user**, I am automatically taken to my first private board when I visit the root `/` route, so that I land on useful content immediately after login.
- **As a board member**, I can view and interact with a kanban board at `/:id` — dragging & dropping cards, filtering by "my tasks", and receiving real-time updates — so that I can manage tasks collaboratively.
- **As an admin**, I can access `/board`, `/user`, `/tag`, and `/notification` admin pages from the sidebar, so that I can manage the application's master data.
- **As a user**, I can view and update my profile at `/profile`, so that I can manage my personal information and Telegram connection.
- **As any user**, my authentication state is restored on page reload from localStorage so that I do not have to log in again.
- **As any user**, all API requests automatically include my Bearer token, and I am logged out if the token becomes invalid (401), so that sessions are managed transparently.
- **As any user**, I can open a task detail modal from anywhere in the board, so that I can view and edit task information without leaving the board.
- **As any user**, I experience consistent dark-themed UI throughout the app with responsive layouts for mobile through desktop.

---

## Acceptance Criteria

### Auth Plugin Restore Flow
- AC-FE-01: On app startup, `0.auth.client.ts` reads the token from localStorage via `useAuth().getAuthToken()`.
- AC-FE-02: If a token exists, the plugin calls `authStore.refreshToken()` followed by `authStore.fetchUser()` before allowing navigation.
- AC-FE-03: If `refreshToken()` fails (network or 401), the plugin calls `authStore.logout()` and leaves state unauthenticated.
- AC-FE-04: The plugin runs exclusively on the client side (`client` suffix in filename).

### Route Guard
- AC-FE-05: `app/middleware/auth.ts` is applied globally to all routes.
- AC-FE-06: On every navigation, if `authStore.isLoggedIn` is `false`, the guard redirects to `/auth/log-in`.
- AC-FE-07: The `/auth/log-in` route is excluded from the redirect loop.

### `useApi` Composable
- AC-FE-08: All HTTP methods (`get`, `post`, `put`, `del`, `form`) automatically attach `Authorization: Bearer <token>` from `useAuth().getAuthToken()`.
- AC-FE-09: On a 401 response, `useApi` calls `authStore.logout()` and redirects to `/auth/log-in`.
- AC-FE-10: On a 422 response, `useApi` extracts validation errors and returns them in a structured format for display.
- AC-FE-11: On a 469 response, `useApi` extracts the `error` field and surfaces it to the caller.
- AC-FE-12: The `form<T>()` method sets `Content-Type: multipart/form-data` for file upload endpoints.
- AC-FE-13: All methods are fully typed with a generic `<T>` return type.
- AC-FE-14: Endpoints are organized as named groups: `auth`, `users`, `boards`, `board_lists`, `tags`, `cards`, `comments`, `attachments`, `notifications`, `centrifugo`, `telegram`.

### `useAuth` Composable
- AC-FE-15: `login(token)` persists the JWT to localStorage.
- AC-FE-16: `logout()` removes the JWT from localStorage.
- AC-FE-17: `getAuthToken()` reads and returns the JWT string (or `null`) from localStorage.
- AC-FE-18: `isAuthenticated()` returns `true` if a non-empty token exists in localStorage.

### `useCardModal` Composable
- AC-FE-19: `openCardModal` is a globally shared `ref` containing `{ cardId, refreshCardData, cardData }` or `null` when closed.
- AC-FE-20: `isCreatingCommentWithAttachment` is a boolean `ref` flag accessible from any component.
- AC-FE-21: Setting `openCardModal` to a non-null value causes the task detail modal to appear.

### Modal Base Wrapper
- AC-FE-22: `modal/index.vue` teleports its slot content to `<body>`.
- AC-FE-23: The modal renders a centered overlay covering the full viewport.
- AC-FE-24: On open, the modal content animates in with a scale-up transition; on close, it animates out with a scale-down transition.
- AC-FE-25: Clicking the overlay backdrop closes the modal (emits `close` event).

### Sidebar & GSAP Animation
- AC-FE-26: The sidebar is collapsible; clicking the toggle runs a GSAP animation that slides the sidebar in or out.
- AC-FE-27: `sidebarStore.isOpen` reflects the current open/closed state.
- AC-FE-28: When `isOpen` changes, the main content area adjusts its left margin via GSAP tween (not CSS transition alone).
- AC-FE-29: Admin navigation links (`/board`, `/user`, `/notification`, `/tag`) are rendered in the sidebar **only** when `authStore.user.role === 'admin'`.
- AC-FE-30: Non-admin users see no admin links in the sidebar.
- AC-FE-31: The sidebar displays the current user's avatar, display name, and a board list with icons from `BoardIconNameMap`.

### Dark Theme
- AC-FE-32: The application uses Tailwind CSS 4 dark mode with the `class` strategy — a `dark` class on the `<html>` element activates dark mode.
- AC-FE-33: The dark theme is the default; the `dark` class is always present.
- AC-FE-34: The color palette uses a custom violet/purple design token set defined in `tailwind.config.js`.
- AC-FE-35: No global custom CSS is written outside component `<style>` blocks; `@apply` is used sparingly within component styles only.

### Pinia Stores

#### `authStore`
- AC-FE-36: `user` holds the current `UserModel` or `null`.
- AC-FE-37: `token` holds the JWT string or `null`.
- AC-FE-38: `is_authenticated` is a boolean computed from `token` being non-null.
- AC-FE-39: `login(credentials)` POSTs to the auth endpoint, stores the token via `useAuth().login()`, sets `user` and `token`.
- AC-FE-40: `logout()` clears `user`, `token`, calls `useAuth().logout()`, and redirects to `/auth/log-in`.
- AC-FE-41: `refreshToken()` exchanges the stored token for a new one and updates `token`.
- AC-FE-42: `fetchUser()` fetches `/api/auth/me` and populates `user`.

#### `boardStore`
- AC-FE-43: `boards` holds an array of `BoardModel`.
- AC-FE-44: `privateBoardID` holds the ID of the current user's private board.
- AC-FE-45: `isLoading` is `true` while `fetchBoards()` is in flight.
- AC-FE-46: `fetchBoards()` fetches all boards for admins, or only the user's own boards for regular users.
- AC-FE-47: `updateBoardInList(board)` replaces the matching board entry in `boards` by ID.
- AC-FE-48: `removeBoardFromList(id)` removes the board with the given ID from `boards`.

#### `boardListStore`
- AC-FE-49: `boardLists` holds the ordered array of `BoardListModel` for the active board.
- AC-FE-50: `boardMembers` holds the array of `BoardUserModel` for the active board.
- AC-FE-51: `boardData` holds the `BoardModel` metadata for the active board.
- AC-FE-52: `isLoading` is `true` while `getBoardLists()` is in flight.
- AC-FE-53: `getBoardLists(boardId)` fetches all lists + members for the given board and populates state.
- AC-FE-54: `updateMember(member)` updates a single member's data in `boardMembers` by user ID.

#### `sidebarStore`
- AC-FE-55: `isOpen` defaults to `true` on first load.
- AC-FE-56: `toggle()` flips `isOpen`.
- AC-FE-57: `open()` sets `isOpen` to `true`; `close()` sets it to `false`.

#### `socketStore`
- AC-FE-58: `state` reflects the Centrifugo connection state (e.g., `disconnected`, `connecting`, `connected`).
- AC-FE-59: `centrifuge` holds the `Centrifuge` client instance or `null`.
- AC-FE-60: `subscriptions` is a `Map<string, Subscription>` keyed by channel name.
- AC-FE-61: `initCentrifuge()` instantiates the `Centrifuge` client with the WS endpoint and token.
- AC-FE-62: `socketConnect()` calls `.connect()` on the client instance.
- AC-FE-63: `socketDisconnect()` calls `.disconnect()` and clears `subscriptions`.
- AC-FE-64: `subscribeToChannels(channels)` subscribes to each channel and stores the subscription object.
- AC-FE-65: `unsubscribeFromChannels(channels)` unsubscribes and removes entries from `subscriptions`.
- AC-FE-66: `publishEvent(channel, event)` publishes a typed event payload to the given Centrifugo channel.

### Pages

#### `/auth/log-in`
- AC-FE-67: Renders a `card/logIn.vue` component centered on an empty layout (`login`).
- AC-FE-68: GSAP animates the login card on page load (fade/slide in).
- AC-FE-69: The form contains `username` and `password` fields validated by `loginSchema` (Zod).
- AC-FE-70: Invalid credentials display an inline error message below the form.
- AC-FE-71: Successful login calls `authStore.login()`, stores the token, and navigates to `/`.

#### `/` (index)
- AC-FE-72: On mount, reads `boardStore.privateBoardID` and calls `navigateTo(`/${privateBoardID}`)`.
- AC-FE-73: If `privateBoardID` is not yet available, waits for `boardStore.fetchBoards()` to complete.

#### `/:id` (Kanban Board)
- AC-FE-74: Uses the `board` layout (sidebar + PlanetOrbit + content + BoardMembers right panel).
- AC-FE-75: On mount, calls `boardListStore.getBoardLists(id)` to load the board's lists and members.
- AC-FE-76: Renders a `card/list.vue` for each item in `boardListStore.boardLists`.
- AC-FE-77: Each list contains a `card/Draggable.vue` that wraps `card/Item.vue` components.
- AC-FE-78: Dragging a card between lists or reordering within a list triggers a PUT request to update card position.
- AC-FE-79: A "Filter: My Tasks" toggle filters visible cards to only those assigned to the current user.
- AC-FE-80: Real-time updates from Centrifugo (card create/update/delete/move events) are reflected immediately without page reload.
- AC-FE-81: Clicking a card opens the task detail modal (`openCardModal` is set with the card's data).

#### `/board` (Admin: Board Management)
- AC-FE-82: Uses the `default` layout.
- AC-FE-83: Wrapped in `ManagementBackdrop`.
- AC-FE-84: Renders a `ui/Table` + `ui/Search` for board CRUD operations.
- AC-FE-85: Admins can create, edit, and delete boards via modal dialogs.
- AC-FE-86: Only accessible to admin users (middleware-enforced; non-admins see no admin sidebar links).

#### `/user` (Admin: User Management)
- AC-FE-87: Uses the `default` layout.
- AC-FE-88: Renders a searchable table of all users.
- AC-FE-89: Admins can create users, view/edit user details, change passwords, and soft-delete users via modals.

#### `/tag` (Admin: Tag Management)
- AC-FE-90: Uses the `default` layout.
- AC-FE-91: Renders a searchable table of all tags.
- AC-FE-92: Each tag displays its color chip (one of 9 colors from `TagColorEnum`).
- AC-FE-93: Admins can add, edit, and delete tags via modals; tag creation/edit includes a color picker.

#### `/notification` (Admin: Notification Logs)
- AC-FE-94: Uses the `default` layout.
- AC-FE-95: Renders a searchable, paginated table of notification log entries.
- AC-FE-96: Read-only view — no create/edit/delete actions.

#### `/profile` (User Profile)
- AC-FE-97: Uses the `default` layout.
- AC-FE-98: Renders a `card/profile.vue` showing the user's avatar, name, username, and Telegram status.
- AC-FE-99: User can update display name, upload/delete avatar, change password, and connect/disconnect Telegram.

### Layouts

#### `login` Layout
- AC-FE-100: Renders only a `<slot>` — no sidebar, no header.

#### `default` Layout
- AC-FE-101: Renders the collapsible `Sidebar` on the left and a scrollable main content area on the right.
- AC-FE-102: Sidebar collapse state is driven by `sidebarStore.isOpen` with GSAP tween on the container width.

#### `board` Layout
- AC-FE-103: Renders the collapsible `Sidebar` on the far left.
- AC-FE-104: Renders `PlanetOrbit` as a decorative background element.
- AC-FE-105: Renders main board content (lists) in the center.
- AC-FE-106: Renders `BoardMembers` as a right-side panel showing board member avatars/names.
- AC-FE-107: The `BoardMembers` panel is only present in the `board` layout, not in `default`.

### UI Kit Components
- AC-FE-108: `ui/Input.vue` accepts prefix/suffix icon slots and a `size` prop (`sm`, `md`, `lg`).
- AC-FE-109: `ui/Tag.vue` accepts a `color` prop matching `TagColorEnum` values and renders the correct background/text color.
- AC-FE-110: `ui/Avatar.vue` renders an `<img>` if an image URL is provided; falls back to the user's initials in a colored circle.
- AC-FE-111: `ui/Table.vue` + `ui/Search.vue` work together — `Table` accepts `columns`, `rows`, `loading` props; `Search` emits filter/sort/pagination events consumed by the parent page.
- AC-FE-112: `ui/DatePickerPersian.vue` wraps `vue3-persian-datetime-picker` for Jalali date input.
- AC-FE-113: `ui/DropDown.vue` is a single-select; `ui/MultiDropDown.vue` supports multiple selected values returned as an array.
- AC-FE-114: `ui/Spinner.vue` renders a loading indicator; used inside async sections and buttons during pending state.
- AC-FE-115: `ui/Collapse.vue` accepts a `title` slot and a `content` slot; toggles content visibility with a smooth height animation.
- AC-FE-116: `ui/Checkbox.vue` emits `update:modelValue` for `v-model` binding and accepts a `disabled` prop.

### Plugins
- AC-FE-117: `draggable.client.ts` registers `<Draggable>` globally so it is available in all components without import.
- AC-FE-118: `iconsax.client.ts` registers `<VsxIcon>` globally.
- AC-FE-119: `sonner.client.ts` registers `<Toaster>` globally and adds `$toast` to `nuxtApp.provide`, enabling `const { $toast } = useNuxtApp()` usage.
- AC-FE-120: `gsap.client.js` registers `ScrollTrigger` and `ScrollToPlugin` with GSAP and provides `$gsap` via `nuxtApp.provide`.
- AC-FE-121: The `0.` prefix on `0.auth.client.ts` ensures it runs before all other plugins.

---

## Component Tree / File Structure

```
app/
├── app.vue                              # Root component — wraps <NuxtLayout><NuxtPage>
├── error.vue                            # Nuxt error boundary page
├── assets/
│   ├── css/                             # Global CSS / Tailwind entry
│   ├── images/
│   └── fonts/
│
├── components/
│   ├── card/
│   │   ├── list.vue                     # Kanban column container
│   │   ├── Draggable.vue                # vue-draggable-plus wrapper for cards
│   │   ├── Item.vue                     # Single task card (name, tags, assignees, status)
│   │   ├── empty.vue                    # Empty state when list has no cards
│   │   ├── logIn.vue                    # Login form card
│   │   └── profile.vue                  # User profile display card
│   │
│   ├── card-board/
│   │   ├── CommentAndActivity.vue       # Combined comment + activity log feed
│   │   ├── CommentInput.vue             # Comment form with optional attachment
│   │   └── CommentItem.vue              # Single comment display
│   │
│   ├── modal/
│   │   ├── index.vue                    # Base modal wrapper (teleport + overlay + scale anim)
│   │   │
│   │   ├── board/
│   │   │   ├── Add.vue                  # Create/edit task modal
│   │   │   └── Delete.vue               # Delete confirmation modal
│   │   │
│   │   ├── board-management/
│   │   │   ├── Add.vue                  # Create board modal
│   │   │   ├── Details.vue              # View/edit board details modal
│   │   │   └── Edit.vue                 # Edit board modal
│   │   │
│   │   ├── user/
│   │   │   ├── Add.vue                  # Create user modal
│   │   │   ├── Detail.vue               # View/edit user modal
│   │   │   ├── Edit.vue                 # Edit user modal
│   │   │   ├── Password.vue             # Change password modal
│   │   │   └── Delete.vue               # Delete user confirmation modal
│   │   │
│   │   ├── task/
│   │   │   ├── Detail.vue               # Task detail modal (full card view)
│   │   │   └── ActivityLog.vue          # Task activity log modal
│   │   │
│   │   ├── tag/
│   │   │   ├── Add.vue                  # Create tag modal (with color picker)
│   │   │   ├── Edit.vue                 # Edit tag modal
│   │   │   └── Delete.vue               # Delete tag confirmation modal
│   │   │
│   │   ├── comment/
│   │   │   └── Delete.vue               # Delete comment confirmation modal
│   │   │
│   │   ├── telegram/
│   │   │   └── Disconnect.vue           # Telegram disconnect confirmation modal
│   │   │
│   │   └── logout/
│   │       └── Confirm.vue              # Logout confirmation modal
│   │
│   ├── layout/
│   │   ├── Sidebar.vue                  # Main sidebar (user, boards, admin nav, logout)
│   │   ├── BoardMembers.vue             # Right panel — board member list (board layout only)
│   │   ├── PlanetOrbit.vue              # Decorative animated orbital background
│   │   └── ManagementBackdrop.vue       # Styling wrapper for admin pages
│   │
│   └── ui/
│       ├── Icon.vue                     # Icon wrapper (@nuxt/icon or vue-iconsax)
│       ├── Spinner.vue                  # Loading indicator
│       ├── Checkbox.vue                 # Styled checkbox (v-model)
│       ├── Collapse.vue                 # Accordion/collapsible panel
│       ├── Option.vue                   # Select option item
│       ├── Input.vue                    # Text input with prefix/suffix + size variants
│       ├── DatePickerPersian.vue        # Jalali date picker (vue3-persian-datetime-picker)
│       ├── DropDown.vue                 # Single-select dropdown
│       ├── MultiDropDown.vue            # Multi-select dropdown
│       ├── File.vue                     # File upload input
│       ├── Table.vue                    # Data table with columns/rows/loading props
│       ├── Search.vue                   # Search/filter/sort/pagination control
│       ├── Tag.vue                      # Tag chip with color
│       └── Avatar.vue                   # User avatar (image or initials fallback)
│
├── composables/
│   ├── useAuth.ts                       # localStorage JWT wrapper
│   ├── useApi.ts                        # Typed HTTP client (all resource endpoints)
│   ├── useCardModal.ts                  # Global card modal state
│   └── api/                             # Resource-specific API composables (optional layer)
│
├── layouts/
│   ├── board.vue                        # Sidebar | PlanetOrbit | Content | BoardMembers
│   ├── default.vue                      # Sidebar | Scrollable Content
│   └── login.vue                        # Slot only (no chrome)
│
├── middleware/
│   └── auth.ts                          # Global route guard → /auth/log-in if unauthenticated
│
├── pages/
│   ├── auth/
│   │   └── log-in.vue                   # Login page (GSAP anim, loginSchema validation)
│   ├── index.vue                        # Root redirect → /:privateBoardID
│   ├── [id].vue                         # Kanban board page
│   ├── board/
│   │   └── index.vue                    # Admin: Board CRUD table
│   ├── user/
│   │   └── index.vue                    # Admin: User CRUD table
│   ├── tag/
│   │   └── index.vue                    # Admin: Tag CRUD table with color picker
│   ├── notification/
│   │   └── index.vue                    # Admin: Notification log viewer
│   └── profile/
│       └── index.vue                    # User profile page
│
├── plugins/
│   ├── 0.auth.client.ts                 # Runs first — restores auth from localStorage
│   ├── draggable.client.ts              # Registers <Draggable> globally
│   ├── iconsax.client.ts               # Registers <VsxIcon> globally
│   ├── sonner.client.ts                 # Registers <Toaster> + provides $toast
│   └── gsap.client.js                   # Registers ScrollTrigger + ScrollToPlugin, provides $gsap
│
├── stores/
│   ├── auth.ts                          # authStore — user, token, is_authenticated
│   ├── board.ts                         # boardStore — boards[], privateBoardID, isLoading
│   ├── boardList.ts                     # boardListStore — boardLists[], boardMembers[], boardData
│   ├── sidebar.ts                       # sidebarStore — isOpen, toggle/open/close
│   └── socket.ts                        # socketStore — Centrifugo client + subscriptions
│
├── types/
│   ├── user.types.ts                    # UserModel
│   ├── board.types.ts                   # BoardModel
│   ├── board-list.types.ts              # BoardListModel
│   ├── card.types.ts                    # CardModel, CardUserModel, CardLogModel
│   ├── comment.types.ts                 # CommentModel
│   ├── attachment.types.ts              # AttachmentModel
│   ├── tag.types.ts                     # TagModel
│   ├── notification.types.ts            # NotificationModel
│   ├── board-user.types.ts              # BoardUserModel
│   ├── status.types.ts                  # Status, StatusId
│   ├── job.types.ts                     # FailedJobModel
│   └── api.types.ts                     # RequestOptions, Response<T>, PaginatedResponse<T>, EnsureResponse<T>
│
└── utils/
    └── (auto-imported utility functions)
```

---

## Data Model

### TypeScript Interfaces

```typescript
// app/types/user.types.ts
interface UserModel {
  id: number
  username: string
  name: string
  email: string | null
  avatar: string | null
  role: 'admin' | 'user'
  is_active: boolean
  telegram_id: string | null
  created_at: string
  updated_at: string
}

// app/types/board.types.ts
interface BoardModel {
  id: number
  name: string
  icon: string          // key from BoardIconNameMap
  is_private: boolean
  owner_id: number
  created_at: string
  updated_at: string
}

// app/types/board-list.types.ts
interface BoardListModel {
  id: number
  board_id: number
  name: string
  position: number
  cards: CardModel[]
}

// app/types/board-user.types.ts
interface BoardUserModel {
  id: number
  board_id: number
  user: UserModel
  joined_at: string
}

// app/types/card.types.ts
interface CardModel {
  id: number
  board_list_id: number
  name: string
  description: string | null
  position: number
  due_date: string | null
  is_deleted: boolean
  tags: TagModel[]
  members: CardUserModel[]
  attachments: AttachmentModel[]
  logs: CardLogModel[]
  created_at: string
  updated_at: string
}

interface CardUserModel {
  id: number
  card_id: number
  user: UserModel
  status: StatusId
}

interface CardLogModel {
  id: number
  card_id: number
  user: UserModel
  type: 'created' | 'updated' | 'commented'
  message: string
  created_at: string
}

// app/types/comment.types.ts
interface CommentModel {
  id: number
  card_id: number
  user: UserModel
  content: string
  is_pinned: boolean
  attachment: AttachmentModel | null
  created_at: string
  updated_at: string
}

// app/types/attachment.types.ts
interface AttachmentModel {
  id: number
  card_id: number
  comment_id: number | null
  user: UserModel
  filename: string
  path: string
  mime_type: string
  size: number
  created_at: string
}

// app/types/tag.types.ts
interface TagModel {
  id: number
  name: string
  color: TagColor       // one of 9 TagColorEnum values
  created_at: string
  updated_at: string
}

// app/types/notification.types.ts
interface NotificationModel {
  id: number
  user_id: number
  type: string          // one of 18 notification type keys
  payload: Record<string, unknown>
  channel: 'telegram' | 'in-app'
  is_sent: boolean
  created_at: string
}

// app/types/status.types.ts
type StatusId = 0 | 1 | 2 | 3 | 4

interface Status {
  id: StatusId
  label: string
  color: string         // hex color code
}

// StatusId → Status mapping
// 0 → { label: 'None',    color: '#B6B1BE' }
// 1 → { label: 'Pending', color: '#EAB035' }
// 2 → { label: 'Doing',   color: '#A777E1' }
// 3 → { label: 'On Hold', color: '#E55151' }
// 4 → { label: 'Done',    color: '#48CC4C' }

// app/types/job.types.ts
interface FailedJobModel {
  id: number
  queue: string
  job_name: string
  data: Record<string, unknown>
  error: string
  failed_at: string
}

// app/types/api.types.ts
interface RequestOptions {
  headers?: Record<string, string>
  params?: Record<string, string | number | boolean>
}

interface Response<T> {
  data: T
  message: string
}

interface PaginatedResponse<T> {
  data: T[]
  meta: {
    total: number
    per_page: number
    current_page: number
    last_page: number
  }
  message: string
}

interface EnsureResponse<T> {
  data: T | null
  message: string
  errors?: Record<string, string[]>
}
```

---

## State Management

### Store Architecture

All stores use **Pinia Setup Stores** syntax (no Options API). Components access stores via composables, not directly. Stores do not import each other — composables act as mediators.

### `authStore` (`app/stores/auth.ts`)

```typescript
// State
const user = ref<UserModel | null>(null)
const token = ref<string | null>(null)
const is_authenticated = computed(() => token.value !== null)

// Actions
async function login(credentials: { username: string; password: string }): Promise<void>
async function logout(): Promise<void>
async function refreshToken(): Promise<void>
async function fetchUser(): Promise<void>
```

### `boardStore` (`app/stores/board.ts`)

```typescript
// State
const boards = ref<BoardModel[]>([])
const privateBoardID = ref<number | null>(null)
const isLoading = ref(false)

// Actions
async function fetchBoards(): Promise<void>   // admin → all, user → own only
function updateBoardInList(board: BoardModel): void
function removeBoardFromList(id: number): void
```

### `boardListStore` (`app/stores/boardList.ts`)

```typescript
// State
const boardLists = ref<BoardListModel[]>([])
const boardMembers = ref<BoardUserModel[]>([])
const boardData = ref<BoardModel | null>(null)
const isLoading = ref(false)

// Actions
async function getBoardLists(boardId: number): Promise<void>
function updateMember(member: BoardUserModel): void
```

### `sidebarStore` (`app/stores/sidebar.ts`)

```typescript
// State
const isOpen = ref(true)

// Actions
function toggle(): void
function open(): void
function close(): void
```

### `socketStore` (`app/stores/socket.ts`)

```typescript
// State
const state = ref<'disconnected' | 'connecting' | 'connected'>('disconnected')
const centrifuge = ref<Centrifuge | null>(null)
const subscriptions = ref<Map<string, Subscription>>(new Map())

// Actions
function initCentrifuge(token: string): void
function socketConnect(): void
function socketDisconnect(): void
function subscribeToChannels(channels: string[]): void
function unsubscribeFromChannels(channels: string[]): void
function publishEvent(channel: string, event: Record<string, unknown>): void
```

---

## Edge Cases

- **Auth plugin on first load (no stored token):** Plugin detects no token → skips `refreshToken`/`fetchUser` → state remains unauthenticated → route guard redirects to `/auth/log-in`.
- **Auth restore failure:** If `refreshToken()` throws (expired/invalid token), the plugin must call `logout()` and clear state before navigation proceeds; it must not leave a partial auth state.
- **Navigation to `/:id` with invalid board ID:** If `getBoardLists(id)` returns a 404, the page should redirect to `/` or display a "Board not found" error state.
- **`privateBoardID` is null at `/` redirect:** If `boardStore.privateBoardID` is null after `fetchBoards()`, the index page should fall back to the first board in `boards[]` or show an error.
- **Sidebar in mobile viewport:** On small screens (`< sm`), the sidebar may overlay content rather than pushing it; GSAP animation must account for mobile overlay mode vs. desktop push mode.
- **Modal teleport in SSR:** `modal/index.vue` teleports to `<body>` — this is a client-only operation; the component must guard against SSR rendering or use `<ClientOnly>`.
- **Card drag-drop optimistic update:** When a card is dragged to a new position, the UI updates immediately (optimistic) and reverts on API error.
- **Centrifugo token expiry mid-session:** `socketStore` must handle Centrifugo token refresh events and reconnect without full page reload.
- **Admin route access by non-admin:** The sidebar hides admin links, but direct URL navigation to `/board`, `/user`, `/tag`, `/notification` by non-admin users must also be rejected — enforced via server middleware (admin check on API routes) rather than client-only route guard.
- **File upload size limits:** The `ui/File.vue` component should validate file size client-side before submitting to match server-side limits.
- **Multiple modal open:** Only one modal should be visible at a time; opening a second modal while one is open should either close the first or stack correctly.

---

## Non-Goals

- **Server-side rendering (SSR):** The app is a client-side SPA. Nuxt's SSR features are not used; plugins are all `client`-only.
- **Light mode:** Only dark mode is supported; a light mode theme toggle is out of scope.
- **Internationalization (i18n):** All UI text is in English (except the Persian date picker). No i18n setup.
- **Offline support / PWA:** No service workers or offline caching.
- **Right-to-left (RTL) layout:** The Persian date picker supports Jalali dates, but the app layout is LTR only.
- **Mobile app / Capacitor:** This spec covers the web SPA only.
- **Component unit test setup:** Testing infrastructure is covered by a separate spec; this spec covers implementation structure only.
- **Backend API implementation:** Server routes, database schema, and transformers are covered by their respective specs.

---

## Dependencies

### Runtime Dependencies (Frontend-Relevant)

| Package | Version | Purpose |
|---------|---------|---------|
| `nuxt` | 4.x | Framework (Vue 3 + Nitro + Vite) |
| `vue` | 3.x | UI framework |
| `pinia` | latest | State management |
| `@pinia/nuxt` | latest | Pinia Nuxt module |
| `tailwindcss` | 4.x | Utility-first CSS |
| `gsap` | 3.13 | Animation (ScrollTrigger, ScrollToPlugin) |
| `vue-draggable-plus` | 4.x | Kanban drag-and-drop |
| `vue-iconsax` | 2.x | Icon set (VsxIcon) |
| `@nuxt/icon` | latest | Nuxt-native icon component |
| `vue-sonner` | latest | Toast notifications |
| `centrifuge` | latest | WebSocket client (Centrifugo JS SDK) |
| `vue3-persian-datetime-picker` | latest | Jalali date picker |
| `vueuse` | latest | VueUse composable utilities |
| `zod` | latest | Runtime validation (loginSchema on client) |

### Internal Spec Dependencies

| Spec | Relationship |
|------|-------------|
| `specs/authentication.md` | Defines JWT endpoints consumed by `authStore`, `useAuth`, and `0.auth.client.ts` |
| `specs/database-schema.md` | Defines entity shapes that TypeScript interfaces in `app/types/` mirror |
| `specs/validation-schemas.md` | Defines shared Zod schemas (e.g., `loginSchema`) used in client-side form validation |
| `specs/realtime-centrifugo.md` | Defines Centrifugo channel names and event payloads consumed by `socketStore` |
| `specs/client-board-kanban.md` | Defines board/list endpoints consumed by `boardListStore` and `/:id` page |
| `specs/client-card-management.md` | Defines card endpoints consumed by `useApi` card resource group |
| `specs/client-comments.md` | Defines comment endpoints and Centrifugo events for `CommentAndActivity.vue` |
| `specs/client-attachments.md` | Defines attachment endpoints consumed by `ui/File.vue` and `CommentInput.vue` |
| `specs/client-user-profile.md` | Defines user profile endpoints consumed by `/profile` page |
| `specs/admin-board-management.md` | Defines admin board endpoints consumed by `/board` page |
| `specs/admin-user-management.md` | Defines admin user endpoints consumed by `/user` page |
| `specs/admin-tag-management.md` | Defines admin tag endpoints consumed by `/tag` page and `TagColorEnum` |
| `specs/admin-notification-logs.md` | Defines notification log endpoints consumed by `/notification` page |
| `specs/infrastructure-setup.md` | Defines `nuxt.config.ts`, Tailwind config, and runtime environment variables |
