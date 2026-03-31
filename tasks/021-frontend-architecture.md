# Tasks: Frontend Architecture
**Spec:** `specs/frontend-architecture.md`
**Status:** pending

---

## Overview

This task file covers all shared/foundational frontend infrastructure for the Nuxt 4 task management application: app entry point, 3 layouts, global route middleware, 12 TypeScript type definition files, 3 core composables, 5 Pinia stores, 5 client-only plugins, 14 UI kit components, 4 layout components, the base modal wrapper, and the two entry-point pages (`/auth/log-in` and `/`).

**Pages covered by feature task files (do NOT duplicate here):**
- `/:id` kanban board page → `tasks/015-client-board-kanban.md`
- `/board` admin page → `tasks/011-admin-board-management.md`
- `/user` admin page → `tasks/012-admin-user-management.md`
- `/tag` admin page → `tasks/013-admin-tag-management.md`
- `/notification` admin page → `tasks/014-admin-notification-logs.md`
- `/profile` page → `tasks/020-client-user-profile.md`

**Feature-specific modals (do NOT duplicate here):**
- `modal/board/`, `modal/board-management/` → `tasks/011`, `tasks/015`, `tasks/017`
- `modal/user/` → `tasks/012`
- `modal/task/` → `tasks/017-client-card-management.md`
- `modal/tag/` → `tasks/013`
- `modal/comment/` → `tasks/018-client-comments.md`
- `modal/telegram/` → `tasks/020`
- `modal/logout/` → included in Task 021-G (layout sidebar dependency)

**Feature-specific card components (do NOT duplicate here):**
- `card/list.vue`, `card/Draggable.vue`, `card/Item.vue`, `card/empty.vue` → `tasks/015`
- `card/profile.vue` → `tasks/020`
- `card-board/` components → `tasks/018`

**Global dependencies for all tasks in this file:**
- `tasks/001-infrastructure-setup.md` — Nuxt config, Tailwind config, Docker env
- `tasks/003-validation-schemas.md` — Zod schemas (e.g., `loginSchema`) used in client forms
- `tasks/004-authentication.md` — JWT endpoints consumed by `authStore` and `0.auth.client.ts`
- `tasks/010-realtime-centrifugo.md` — Centrifugo WS endpoint and token endpoint consumed by `socketStore`

---

## Wave 1 — Foundation (Frontend Agent)

> All Wave 1 tasks are independent of each other and can run in parallel.

---

### Task 021-A: App Entry Point, Layouts & Route Middleware
- **Agent:** Frontend
- **Spec:** `specs/frontend-architecture.md`
- **Section:** Layouts (AC-FE-100–107), Dark Theme (AC-FE-32–35), Pages structure
- **Acceptance Criteria:**
  - [ ] AC-FE-05: `app/middleware/auth.ts` is applied globally to all routes.
  - [ ] AC-FE-06: On every navigation, if `authStore.isLoggedIn` is `false`, the guard redirects to `/auth/log-in`.
  - [ ] AC-FE-07: The `/auth/log-in` route is excluded from the redirect loop.
  - [ ] AC-FE-32: The application uses Tailwind CSS 4 dark mode with the `class` strategy — a `dark` class on `<html>` activates dark mode.
  - [ ] AC-FE-33: The dark theme is the default; the `dark` class is always present on `<html>`.
  - [ ] AC-FE-34: The color palette uses a custom violet/purple design token set defined in `tailwind.config.js`.
  - [ ] AC-FE-35: No global custom CSS outside component `<style>` blocks; `@apply` used sparingly within component styles only.
  - [ ] AC-FE-100: `login` layout renders only a `<slot>` — no sidebar, no header.
  - [ ] AC-FE-101: `default` layout renders the collapsible `Sidebar` on the left and a scrollable main content area on the right.
  - [ ] AC-FE-102: `default` layout sidebar collapse state is driven by `sidebarStore.isOpen` with GSAP tween on container width.
  - [ ] AC-FE-103: `board` layout renders the collapsible `Sidebar` on the far left.
  - [ ] AC-FE-104: `board` layout renders `PlanetOrbit` as a decorative background element.
  - [ ] AC-FE-105: `board` layout renders main board content (lists) in the center.
  - [ ] AC-FE-106: `board` layout renders `BoardMembers` as a right-side panel.
  - [ ] AC-FE-107: The `BoardMembers` panel is only present in the `board` layout, not in `default`.
- **Dependencies:** `tasks/001-infrastructure-setup.md`
- **Output Files:**
  - `app/app.vue`
  - `app/error.vue`
  - `app/middleware/auth.ts`
  - `app/layouts/login.vue`
  - `app/layouts/default.vue`
  - `app/layouts/board.vue`

---

### Task 021-B: TypeScript Type Definitions
- **Agent:** Frontend
- **Spec:** `specs/frontend-architecture.md`
- **Section:** Data Model → TypeScript Interfaces
- **Acceptance Criteria:**
  - [ ] `UserModel` — id, username, name, email, avatar, role (`'admin' | 'user'`), is_active, telegram_id, created_at, updated_at.
  - [ ] `BoardModel` — id, name, icon (key from `BoardIconNameMap`), is_private, owner_id, created_at, updated_at.
  - [ ] `BoardListModel` — id, board_id, name, position, cards (`CardModel[]`).
  - [ ] `BoardUserModel` — id, board_id, user (`UserModel`), joined_at.
  - [ ] `CardModel` — id, board_list_id, name, description, position, due_date, is_deleted, tags, members (`CardUserModel[]`), attachments, logs, created_at, updated_at.
  - [ ] `CardUserModel` — id, card_id, user (`UserModel`), status (`StatusId`).
  - [ ] `CardLogModel` — id, card_id, user, type (`'created' | 'updated' | 'commented'`), message, created_at.
  - [ ] `CommentModel` — id, card_id, user, content, is_pinned, attachment (`AttachmentModel | null`), created_at, updated_at.
  - [ ] `AttachmentModel` — id, card_id, comment_id, user, filename, path, mime_type, size, created_at.
  - [ ] `TagModel` — id, name, color (`TagColor`), created_at, updated_at.
  - [ ] `NotificationModel` — id, user_id, type, payload, channel, is_sent, created_at.
  - [ ] `StatusId` typed as `0 | 1 | 2 | 3 | 4`; `Status` interface with id, label, color (hex); status map documented in code.
  - [ ] `FailedJobModel` — id, queue, job_name, data, error, failed_at.
  - [ ] `RequestOptions`, `Response<T>`, `PaginatedResponse<T>` (with meta pagination), `EnsureResponse<T>` defined in `api.types.ts`.
  - [ ] All interfaces use strict TypeScript — no `any`; unions used where applicable.
- **Dependencies:** `tasks/001-infrastructure-setup.md`, `tasks/002-database-schema.md`
- **Output Files:**
  - `app/types/user.types.ts`
  - `app/types/board.types.ts`
  - `app/types/board-list.types.ts`
  - `app/types/board-user.types.ts`
  - `app/types/card.types.ts`
  - `app/types/comment.types.ts`
  - `app/types/attachment.types.ts`
  - `app/types/tag.types.ts`
  - `app/types/notification.types.ts`
  - `app/types/status.types.ts`
  - `app/types/job.types.ts`
  - `app/types/api.types.ts`

---

### Task 021-C: Core Composables
- **Agent:** Frontend
- **Spec:** `specs/frontend-architecture.md`
- **Section:** `useApi` Composable (AC-FE-08–14), `useAuth` Composable (AC-FE-15–18), `useCardModal` Composable (AC-FE-19–21)
- **Acceptance Criteria:**
  - [ ] AC-FE-08: All HTTP methods (`get`, `post`, `put`, `del`, `form`) in `useApi` automatically attach `Authorization: Bearer <token>` from `useAuth().getAuthToken()`.
  - [ ] AC-FE-09: On a 401 response, `useApi` calls `authStore.logout()` and redirects to `/auth/log-in`.
  - [ ] AC-FE-10: On a 422 response, `useApi` extracts validation errors and returns them in a structured format for display.
  - [ ] AC-FE-11: On a 469 response, `useApi` extracts the `error` field and surfaces it to the caller.
  - [ ] AC-FE-12: `form<T>()` method sets `Content-Type: multipart/form-data` for file upload endpoints.
  - [ ] AC-FE-13: All `useApi` methods are fully typed with a generic `<T>` return type.
  - [ ] AC-FE-14: Endpoints are organized as named groups: `auth`, `users`, `boards`, `board_lists`, `tags`, `cards`, `comments`, `attachments`, `notifications`, `centrifugo`, `telegram`.
  - [ ] AC-FE-15: `useAuth.login(token)` persists the JWT to localStorage.
  - [ ] AC-FE-16: `useAuth.logout()` removes the JWT from localStorage.
  - [ ] AC-FE-17: `useAuth.getAuthToken()` reads and returns the JWT string or `null` from localStorage.
  - [ ] AC-FE-18: `useAuth.isAuthenticated()` returns `true` if a non-empty token exists in localStorage.
  - [ ] AC-FE-19: `openCardModal` is a globally shared `ref` containing `{ cardId, refreshCardData, cardData }` or `null` when closed.
  - [ ] AC-FE-20: `isCreatingCommentWithAttachment` is a boolean `ref` flag accessible from any component.
  - [ ] AC-FE-21: Setting `openCardModal` to a non-null value causes the task detail modal to appear.
- **Dependencies:** `tasks/001-infrastructure-setup.md`, `tasks/003-validation-schemas.md`, `tasks/004-authentication.md`
- **Output Files:**
  - `app/composables/useAuth.ts`
  - `app/composables/useApi.ts`
  - `app/composables/useCardModal.ts`

---

### Task 021-D: Pinia Stores
- **Agent:** Frontend
- **Spec:** `specs/frontend-architecture.md`
- **Section:** Pinia Stores (AC-FE-36–66), State Management
- **Acceptance Criteria:**
  - [ ] AC-FE-36: `authStore.user` holds the current `UserModel` or `null`.
  - [ ] AC-FE-37: `authStore.token` holds the JWT string or `null`.
  - [ ] AC-FE-38: `authStore.is_authenticated` is a boolean computed from `token` being non-null.
  - [ ] AC-FE-39: `authStore.login(credentials)` POSTs to the auth endpoint, stores the token via `useAuth().login()`, sets `user` and `token`.
  - [ ] AC-FE-40: `authStore.logout()` clears `user`, `token`, calls `useAuth().logout()`, and redirects to `/auth/log-in`.
  - [ ] AC-FE-41: `authStore.refreshToken()` exchanges the stored token for a new one and updates `token`.
  - [ ] AC-FE-42: `authStore.fetchUser()` fetches `/api/auth/me` and populates `user`.
  - [ ] AC-FE-43: `boardStore.boards` holds an array of `BoardModel`.
  - [ ] AC-FE-44: `boardStore.privateBoardID` holds the ID of the current user's private board.
  - [ ] AC-FE-45: `boardStore.isLoading` is `true` while `fetchBoards()` is in flight.
  - [ ] AC-FE-46: `boardStore.fetchBoards()` fetches all boards for admins, or only the user's own boards for regular users.
  - [ ] AC-FE-47: `boardStore.updateBoardInList(board)` replaces the matching board entry in `boards` by ID.
  - [ ] AC-FE-48: `boardStore.removeBoardFromList(id)` removes the board with the given ID from `boards`.
  - [ ] AC-FE-49: `boardListStore.boardLists` holds the ordered array of `BoardListModel` for the active board.
  - [ ] AC-FE-50: `boardListStore.boardMembers` holds the array of `BoardUserModel` for the active board.
  - [ ] AC-FE-51: `boardListStore.boardData` holds the `BoardModel` metadata for the active board.
  - [ ] AC-FE-52: `boardListStore.isLoading` is `true` while `getBoardLists()` is in flight.
  - [ ] AC-FE-53: `boardListStore.getBoardLists(boardId)` fetches all lists + members for the given board and populates state.
  - [ ] AC-FE-54: `boardListStore.updateMember(member)` updates a single member's data in `boardMembers` by user ID.
  - [ ] AC-FE-55: `sidebarStore.isOpen` defaults to `true` on first load.
  - [ ] AC-FE-56: `sidebarStore.toggle()` flips `isOpen`.
  - [ ] AC-FE-57: `sidebarStore.open()` sets `isOpen` to `true`; `close()` sets it to `false`.
  - [ ] AC-FE-58: `socketStore.state` reflects the Centrifugo connection state (`disconnected`, `connecting`, `connected`).
  - [ ] AC-FE-59: `socketStore.centrifuge` holds the `Centrifuge` client instance or `null`.
  - [ ] AC-FE-60: `socketStore.subscriptions` is a `Map<string, Subscription>` keyed by channel name.
  - [ ] AC-FE-61: `socketStore.initCentrifuge()` instantiates the `Centrifuge` client with the WS endpoint and token.
  - [ ] AC-FE-62: `socketStore.socketConnect()` calls `.connect()` on the client instance.
  - [ ] AC-FE-63: `socketStore.socketDisconnect()` calls `.disconnect()` and clears `subscriptions`.
  - [ ] AC-FE-64: `socketStore.subscribeToChannels(channels)` subscribes to each channel and stores the subscription object.
  - [ ] AC-FE-65: `socketStore.unsubscribeFromChannels(channels)` unsubscribes and removes entries from `subscriptions`.
  - [ ] AC-FE-66: `socketStore.publishEvent(channel, event)` publishes a typed event payload to the given Centrifugo channel.
  - [ ] All stores use Pinia Setup Stores syntax (no Options API); stores do not import each other.
- **Dependencies:** `tasks/001-infrastructure-setup.md`, `tasks/004-authentication.md`, `tasks/010-realtime-centrifugo.md`
- **Consumed by:** `tasks/011`, `tasks/012`, `tasks/013`, `tasks/014`, `tasks/015`, `tasks/016`, `tasks/017`, `tasks/018`, `tasks/019`, `tasks/020`
- **Output Files:**
  - `app/stores/auth.ts`
  - `app/stores/board.ts`
  - `app/stores/boardList.ts`
  - `app/stores/sidebar.ts`
  - `app/stores/socket.ts`

---

### Task 021-E: Client-Only Plugins
- **Agent:** Frontend
- **Spec:** `specs/frontend-architecture.md`
- **Section:** Auth Plugin Restore Flow (AC-FE-01–04), Plugins (AC-FE-117–121)
- **Acceptance Criteria:**
  - [ ] AC-FE-01: On app startup, `0.auth.client.ts` reads the token from localStorage via `useAuth().getAuthToken()`.
  - [ ] AC-FE-02: If a token exists, the plugin calls `authStore.refreshToken()` followed by `authStore.fetchUser()` before allowing navigation.
  - [ ] AC-FE-03: If `refreshToken()` fails (network or 401), the plugin calls `authStore.logout()` and leaves state unauthenticated.
  - [ ] AC-FE-04: The plugin runs exclusively on the client side (`client` suffix in filename).
  - [ ] AC-FE-117: `draggable.client.ts` registers `<Draggable>` globally so it is available in all components without import.
  - [ ] AC-FE-118: `iconsax.client.ts` registers `<VsxIcon>` globally.
  - [ ] AC-FE-119: `sonner.client.ts` registers `<Toaster>` globally and adds `$toast` to `nuxtApp.provide`, enabling `const { $toast } = useNuxtApp()`.
  - [ ] AC-FE-120: `gsap.client.js` registers `ScrollTrigger` and `ScrollToPlugin` with GSAP and provides `$gsap` via `nuxtApp.provide`.
  - [ ] AC-FE-121: The `0.` prefix on `0.auth.client.ts` ensures it runs before all other plugins.
  - [ ] Edge case: If no token is stored on first load, auth plugin skips `refreshToken`/`fetchUser` and leaves state unauthenticated cleanly.
  - [ ] Edge case: If `refreshToken()` throws (expired/invalid), the plugin calls `logout()` and clears state before navigation — no partial auth state permitted.
- **Dependencies:** `tasks/001-infrastructure-setup.md`, `tasks/004-authentication.md`
- **Note:** Plugin 021-E depends on stores from Task 021-D — execute 021-D before 021-E within the same wave.
- **Output Files:**
  - `app/plugins/0.auth.client.ts`
  - `app/plugins/draggable.client.ts`
  - `app/plugins/iconsax.client.ts`
  - `app/plugins/sonner.client.ts`
  - `app/plugins/gsap.client.js`

---

## Wave 2 — UI Components (Frontend Agent)

> All Wave 2 tasks depend on Wave 1 completion. Tasks 021-F, 021-G, 021-H, and 021-I are independent of each other and can run in parallel.

---

### Task 021-F: UI Kit Components
- **Agent:** Frontend
- **Spec:** `specs/frontend-architecture.md`
- **Section:** UI Kit Components (AC-FE-108–116)
- **Acceptance Criteria:**
  - [ ] AC-FE-108: `ui/Input.vue` accepts prefix/suffix icon slots and a `size` prop (`sm`, `md`, `lg`).
  - [ ] AC-FE-109: `ui/Tag.vue` accepts a `color` prop matching `TagColorEnum` values and renders the correct background/text color.
  - [ ] AC-FE-110: `ui/Avatar.vue` renders an `<img>` if an image URL is provided; falls back to user's initials in a colored circle.
  - [ ] AC-FE-111: `ui/Table.vue` accepts `columns`, `rows`, and `loading` props; `ui/Search.vue` emits filter/sort/pagination events consumed by parent pages.
  - [ ] AC-FE-112: `ui/DatePickerPersian.vue` wraps `vue3-persian-datetime-picker` for Jalali date input.
  - [ ] AC-FE-113: `ui/DropDown.vue` is a single-select; `ui/MultiDropDown.vue` supports multiple selected values returned as an array.
  - [ ] AC-FE-114: `ui/Spinner.vue` renders a loading indicator; used inside async sections and buttons during pending state.
  - [ ] AC-FE-115: `ui/Collapse.vue` accepts a `title` slot and a `content` slot; toggles content visibility with a smooth height animation.
  - [ ] AC-FE-116: `ui/Checkbox.vue` emits `update:modelValue` for `v-model` binding and accepts a `disabled` prop.
  - [ ] `ui/Icon.vue` wraps `@nuxt/icon`/`vue-iconsax` rendering into a unified component interface.
  - [ ] `ui/Option.vue` renders a single selectable option item (used inside `DropDown`/`MultiDropDown`).
  - [ ] `ui/File.vue` renders a file upload input; validates file size client-side before submitting to match server-side limits.
  - [ ] All components use `<script setup lang="ts">`, `defineProps<T>()`, and `defineEmits<T>()` — no Options API.
  - [ ] All components apply dark-theme Tailwind classes consistent with the violet/purple design token palette.
- **Dependencies:** Wave 1 (021-A, 021-B)
- **Consumed by:** `tasks/011`, `tasks/012`, `tasks/013`, `tasks/014`, `tasks/015`, `tasks/016`, `tasks/017`, `tasks/018`, `tasks/019`, `tasks/020`
- **Output Files:**
  - `app/components/ui/Icon.vue`
  - `app/components/ui/Spinner.vue`
  - `app/components/ui/Checkbox.vue`
  - `app/components/ui/Collapse.vue`
  - `app/components/ui/Option.vue`
  - `app/components/ui/Input.vue`
  - `app/components/ui/DatePickerPersian.vue`
  - `app/components/ui/DropDown.vue`
  - `app/components/ui/MultiDropDown.vue`
  - `app/components/ui/File.vue`
  - `app/components/ui/Table.vue`
  - `app/components/ui/Search.vue`
  - `app/components/ui/Tag.vue`
  - `app/components/ui/Avatar.vue`

---

### Task 021-G: Layout Components
- **Agent:** Frontend
- **Spec:** `specs/frontend-architecture.md`
- **Section:** Sidebar & GSAP Animation (AC-FE-26–31), Layouts (AC-FE-103–107)
- **Acceptance Criteria:**
  - [ ] AC-FE-26: `Sidebar.vue` is collapsible; clicking the toggle runs a GSAP animation that slides the sidebar in or out.
  - [ ] AC-FE-27: `sidebarStore.isOpen` reflects the current open/closed state.
  - [ ] AC-FE-28: When `isOpen` changes, the main content area adjusts its left margin via GSAP tween (not CSS transition alone).
  - [ ] AC-FE-29: Admin navigation links (`/board`, `/user`, `/notification`, `/tag`) are rendered in `Sidebar.vue` **only** when `authStore.user.role === 'admin'`.
  - [ ] AC-FE-30: Non-admin users see no admin links in the sidebar.
  - [ ] AC-FE-31: The sidebar displays the current user's avatar, display name, and a board list with icons from `BoardIconNameMap`.
  - [ ] `BoardMembers.vue` renders the list of board member avatars and names for the active board, sourcing data from `boardListStore.boardMembers`.
  - [ ] `PlanetOrbit.vue` renders a decorative animated orbital background element (GSAP or CSS animation).
  - [ ] `ManagementBackdrop.vue` wraps admin page content with the correct dark-themed styling backdrop.
  - [ ] `modal/logout/Confirm.vue` renders a logout confirmation modal using `modal/index.vue` as the base wrapper.
  - [ ] Edge case: On mobile viewport (`< sm`), `Sidebar.vue` uses overlay mode; GSAP animation accounts for overlay vs. push mode.
- **Dependencies:** Wave 1 (021-A, 021-C, 021-D, 021-E)
- **Output Files:**
  - `app/components/layout/Sidebar.vue`
  - `app/components/layout/BoardMembers.vue`
  - `app/components/layout/PlanetOrbit.vue`
  - `app/components/layout/ManagementBackdrop.vue`
  - `app/components/modal/logout/Confirm.vue`

---

### Task 021-H: Modal Base Wrapper
- **Agent:** Frontend
- **Spec:** `specs/frontend-architecture.md`
- **Section:** Modal Base Wrapper (AC-FE-22–25)
- **Acceptance Criteria:**
  - [ ] AC-FE-22: `modal/index.vue` teleports its slot content to `<body>`.
  - [ ] AC-FE-23: The modal renders a centered overlay covering the full viewport.
  - [ ] AC-FE-24: On open, the modal content animates in with a scale-up transition; on close, it animates out with a scale-down transition.
  - [ ] AC-FE-25: Clicking the overlay backdrop closes the modal (emits `close` event).
  - [ ] Edge case: `modal/index.vue` guards against SSR rendering — teleport to `<body>` is client-only (use `<ClientOnly>` or `mounted` guard).
  - [ ] Edge case: Only one modal is visible at a time; opening a second modal while one is open either closes the first or stacks correctly.
- **Dependencies:** Wave 1 (021-A, 021-E)
- **Consumed by:** `tasks/011`, `tasks/012`, `tasks/013`, `tasks/014`, `tasks/017`, `tasks/018`, `tasks/019`, `tasks/020`
- **Output Files:**
  - `app/components/modal/index.vue`

---

### Task 021-I: Auth Pages & Login Form
- **Agent:** Frontend
- **Spec:** `specs/frontend-architecture.md`
- **Section:** Pages — `/auth/log-in` (AC-FE-67–71), `/` index (AC-FE-72–73)
- **Acceptance Criteria:**
  - [ ] AC-FE-67: `/auth/log-in` renders `card/logIn.vue` centered on the `login` layout (no sidebar, no header).
  - [ ] AC-FE-68: GSAP animates the login card on page load (fade/slide in).
  - [ ] AC-FE-69: The form contains `username` and `password` fields validated by `loginSchema` (Zod).
  - [ ] AC-FE-70: Invalid credentials display an inline error message below the form.
  - [ ] AC-FE-71: Successful login calls `authStore.login()`, stores the token, and navigates to `/`.
  - [ ] AC-FE-72: `index.vue` on mount reads `boardStore.privateBoardID` and calls `navigateTo(`/${privateBoardID}`)`.
  - [ ] AC-FE-73: If `privateBoardID` is not yet available, `index.vue` waits for `boardStore.fetchBoards()` to complete.
  - [ ] Edge case: If `privateBoardID` is still `null` after `fetchBoards()`, fall back to the first board in `boards[]` or display an error state.
  - [ ] `definePageMeta()` used for page-level metadata on both pages.
  - [ ] `NuxtLink` used instead of `<a>` or `router-link` wherever applicable.
- **Dependencies:** Wave 1 (021-A, 021-C, 021-D), `tasks/003-validation-schemas.md`
- **Output Files:**
  - `app/pages/auth/log-in.vue`
  - `app/pages/index.vue`
  - `app/components/card/logIn.vue`

---

## Wave 3 — Tests (Testing Agent)

> Wave 3 depends on all Wave 1 and Wave 2 tasks completing successfully.

---

### Task 021-J: Frontend Architecture Tests
- **Agent:** Testing
- **Spec:** `specs/frontend-architecture.md`
- **Section:** All ACs (verification pass)
- **Acceptance Criteria:**

  **Auth Plugin (0.auth.client.ts):**
  - [ ] Test: Token exists in localStorage → `refreshToken()` then `fetchUser()` are called in order.
  - [ ] Test: `refreshToken()` throws → `logout()` is called; `authStore.token` is `null` after plugin runs.
  - [ ] Test: No token in localStorage → neither `refreshToken()` nor `fetchUser()` is called.

  **Route Guard (middleware/auth.ts):**
  - [ ] Test: Unauthenticated user navigating to `/` is redirected to `/auth/log-in`.
  - [ ] Test: Navigation to `/auth/log-in` does not trigger a redirect loop.
  - [ ] Test: Authenticated user navigating to `/` is not redirected.

  **`useAuth` Composable:**
  - [ ] Test: `login(token)` → token is persisted in localStorage.
  - [ ] Test: `logout()` → token is removed from localStorage.
  - [ ] Test: `getAuthToken()` → returns stored token or `null`.
  - [ ] Test: `isAuthenticated()` → returns `true` when token exists, `false` otherwise.

  **`useApi` Composable:**
  - [ ] Test: All HTTP methods attach `Authorization: Bearer <token>` header.
  - [ ] Test: 401 response → `authStore.logout()` is called and redirect occurs.
  - [ ] Test: 422 response → validation errors are returned in structured format.
  - [ ] Test: `form()` method sets `Content-Type: multipart/form-data`.

  **`useCardModal` Composable:**
  - [ ] Test: Setting `openCardModal` to a non-null value → `isOpen` becomes `true` (or equivalent reactive state).
  - [ ] Test: `isCreatingCommentWithAttachment` is a shared ref accessible across component instances.

  **Pinia Stores:**
  - [ ] Test: `authStore.is_authenticated` is `true` when `token` is non-null, `false` otherwise.
  - [ ] Test: `authStore.logout()` clears user and token.
  - [ ] Test: `sidebarStore.toggle()` flips `isOpen`; `open()` and `close()` set it absolutely.
  - [ ] Test: `boardStore.updateBoardInList(board)` replaces the correct board by ID.
  - [ ] Test: `boardStore.removeBoardFromList(id)` removes the correct board.
  - [ ] Test: `boardListStore.updateMember(member)` updates the correct member by user ID.

  **`modal/index.vue`:**
  - [ ] Test: Component mounts with teleport target `<body>`.
  - [ ] Test: Clicking the overlay backdrop emits `close`.
  - [ ] Test: Open transition applies scale-up CSS/GSAP class; close applies scale-down.

  **UI Kit Components:**
  - [ ] Test: `ui/Avatar.vue` renders `<img>` when URL provided; renders initials circle when no URL.
  - [ ] Test: `ui/Checkbox.vue` emits `update:modelValue` on change; is disabled when `disabled` prop is set.
  - [ ] Test: `ui/Input.vue` renders prefix/suffix slots; applies correct size class for each `size` prop value.
  - [ ] Test: `ui/Tag.vue` applies correct Tailwind class for each `TagColorEnum` value.
  - [ ] Test: `ui/Spinner.vue` renders a visible loading element.
  - [ ] Test: `ui/Table.vue` renders correct number of rows; shows loading state when `loading` prop is `true`.

  **Plugins:**
  - [ ] Test: `gsap.client.js` provides `$gsap` via `nuxtApp.provide`; `ScrollTrigger` and `ScrollToPlugin` are registered.
  - [ ] Test: `sonner.client.ts` provides `$toast` via `nuxtApp.provide`.

  **E2E — Login Flow:**
  - [ ] E2E: User visits `/` → redirected to `/auth/log-in`.
  - [ ] E2E: User submits invalid credentials → inline error message is visible.
  - [ ] E2E: User submits valid credentials → redirected to kanban board (`/:id`).
  - [ ] E2E: Authenticated user reloads page → stays on current route (auth restored from localStorage).

- **Dependencies:** All Wave 1 (021-A, 021-B, 021-C, 021-D, 021-E) + All Wave 2 (021-F, 021-G, 021-H, 021-I)
- **Output Files:**
  - `tests/unit/app/plugins/0.auth.client.test.ts`
  - `tests/unit/app/middleware/auth.test.ts`
  - `tests/unit/app/composables/useAuth.test.ts`
  - `tests/unit/app/composables/useApi.test.ts`
  - `tests/unit/app/composables/useCardModal.test.ts`
  - `tests/unit/app/stores/auth.test.ts`
  - `tests/unit/app/stores/board.test.ts`
  - `tests/unit/app/stores/boardList.test.ts`
  - `tests/unit/app/stores/sidebar.test.ts`
  - `tests/unit/app/components/modal/index.test.ts`
  - `tests/unit/app/components/ui/Avatar.test.ts`
  - `tests/unit/app/components/ui/Checkbox.test.ts`
  - `tests/unit/app/components/ui/Input.test.ts`
  - `tests/unit/app/components/ui/Tag.test.ts`
  - `tests/unit/app/components/ui/Spinner.test.ts`
  - `tests/unit/app/components/ui/Table.test.ts`
  - `tests/e2e/auth.spec.ts`

---

## Execution Order Summary

```
Wave 1 (parallel):
  021-A  App Entry Point, Layouts & Route Middleware
  021-B  TypeScript Type Definitions
  021-C  Core Composables
  021-D  Pinia Stores          ← 021-E depends on this; complete 021-D first
  021-E  Client-Only Plugins

Wave 2 (parallel, after Wave 1):
  021-F  UI Kit Components (14)
  021-G  Layout Components + Logout Modal
  021-H  Modal Base Wrapper
  021-I  Auth Pages & Login Form

Wave 3 (after Wave 1 + Wave 2):
  021-J  Frontend Architecture Tests
```

## Cross-Reference: Downstream Consumers

| Task | Consumes from 021 |
|------|-------------------|
| `011-admin-board-management` | `ui/Table`, `ui/Search`, `ui/Input`, `ui/DropDown`, `ui/Spinner`, `ui/Avatar`, `modal/index.vue`, `ManagementBackdrop`, `authStore`, `boardStore` |
| `012-admin-user-management` | `ui/Table`, `ui/Search`, `ui/Input`, `ui/Checkbox`, `ui/Avatar`, `modal/index.vue`, `ManagementBackdrop`, `authStore` |
| `013-admin-tag-management` | `ui/Table`, `ui/Search`, `ui/Input`, `ui/Tag`, `modal/index.vue`, `ManagementBackdrop`, `authStore` |
| `014-admin-notification-logs` | `ui/Table`, `ui/Search`, `ui/Spinner`, `modal/index.vue`, `ManagementBackdrop`, `authStore` |
| `015-client-board-kanban` | `boardListStore`, `boardStore`, `socketStore`, `useCardModal`, `useApi`, `ui/Avatar`, `ui/Tag`, `ui/Spinner`, `ui/Checkbox`, `board` layout |
| `016-client-board-list-management` | `boardListStore`, `useApi`, `socketStore`, `ui/Input`, `ui/Spinner`, `modal/index.vue` |
| `017-client-card-management` | `useCardModal`, `useApi`, `boardListStore`, `ui/Input`, `ui/DropDown`, `ui/MultiDropDown`, `ui/Tag`, `ui/DatePickerPersian`, `ui/Avatar`, `ui/Spinner`, `modal/index.vue` |
| `018-client-comments` | `useCardModal`, `useApi`, `socketStore`, `ui/Avatar`, `ui/File`, `ui/Spinner`, `modal/index.vue` |
| `019-client-attachments` | `useApi`, `ui/File`, `ui/Spinner`, `ui/Avatar`, `modal/index.vue` |
| `020-client-user-profile` | `authStore`, `useApi`, `ui/Input`, `ui/Avatar`, `ui/Spinner`, `default` layout |
