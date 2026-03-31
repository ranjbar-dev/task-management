# Tasks: Admin User Management
**Spec:** `specs/admin-user-management.md`
**Status:** pending

---

## Overview

Full CRUD interface for admin-managed users. Seven Nitro API endpoints under `server/api/admin/user/`, all guarded by `auth` + `active-user` + `admin` middleware. Includes bcrypt password hashing, `initializeUserDefaults()` on user creation, auto-private-board on role promotion, and a modal-driven admin page at `app/pages/user/index.vue`.

Execution order: shared schemas/enum first → server utilities & transformer → API handlers → frontend page & modals → tests.

---

## Wave 1 — Data Layer (API Agent)

### Task 012-A: Shared Enum & Validation Schemas

- **Agent:** API
- **Spec:** `specs/admin-user-management.md`
- **Section:** TypeScript Interfaces, Acceptance Criteria AC-4, AC-5, AC-6
- **Acceptance Criteria:**
  - [ ] AC-4: `createUserSchema` validates `name` (1–50 chars), `username` (1–50 chars), `password` (min 8 chars); optional `role` (0 or 1, defaults to `UserRoleEnum.NormalUser`), optional `is_disable` (boolean, defaults to `false`)
  - [ ] AC-5: `updateUserSchema` makes all fields optional — `name`, `username`, `role`, `is_disable`
  - [ ] AC-6: `updatePasswordSchema` requires `current_password` (min 8), `new_password` (min 8), `confirm_password` (min 8); validates `new_password === confirm_password` at schema level (returns `422` otherwise)
  - [ ] `UserRoleEnum` exported as `const` object with `as const` — `{ NormalUser: 0, AdminUser: 1 }`; companion `UserRole` type derived from it
- **Dependencies:** `tasks/003-validation-schemas.md`
- **Output Files:**
  - `shared/enums/UserRoleEnum.ts`
  - `shared/schemas/user.schema.ts`

---

### Task 012-B: Server Utilities & User Transformer

- **Agent:** API
- **Spec:** `specs/admin-user-management.md`
- **Section:** Component Tree / File Structure, AC-4 (bcrypt), AC-6 (bcrypt), Data Model
- **Acceptance Criteria:**
  - [ ] AC-4: `hashPassword(plain: string): Promise<string>` hashes with bcrypt at `SALT_ROUNDS = 10`
  - [ ] AC-6: `verifyPassword(plain: string, hash: string): Promise<boolean>` compares via bcrypt
  - [ ] AC-4: `initializeUserDefaults(userId: number): Promise<void>` sets `telegram.id` to a 12-character random token and sets all 12 notification preference keys to `true` for the given user
  - [ ] AC-1 / AC-2 / AC-3 / AC-4 / AC-5 / AC-7: `transformUser(user: UserModel): UserResponse` — maps DB row to `UserResponse`; `password` field is **never** included in the output
- **Dependencies:** `tasks/002-database-schema.md`, Task 012-A
- **Output Files:**
  - `server/utils/password.ts`
  - `server/utils/userDefaults.ts`
  - `server/transformers/user.transformer.ts`

---

### Task 012-C: API Route Handlers

- **Agent:** API
- **Spec:** `specs/admin-user-management.md`
- **Section:** AC-1 through AC-7, Edge Cases
- **Acceptance Criteria:**
  - [ ] AC-1: `GET /api/admin/user/all` — returns all users as `transformUser` array; excludes `password`; guarded by `auth` + `active-user` + `admin` middleware (returns `401`/`403` without valid session/role)
  - [ ] AC-2: `POST /api/admin/user/table` — accepts SearchTable service parameters (pagination, sort, filters); returns paginated result via `transformUser`; same middleware guards
  - [ ] AC-3: `GET /api/admin/user/[id]` — returns single user via `transformUser`; returns `404` when user does not exist; same middleware guards
  - [ ] AC-4: `POST /api/admin/user/create` — validates body with `createUserSchema` (returns `422` on error); returns `409` on duplicate username; hashes password with `hashPassword`; inserts user; calls `initializeUserDefaults()`; returns created user via `transformUser` with HTTP `201`; same middleware guards
  - [ ] AC-5: `PUT /api/admin/user/[id]` — validates body with `updateUserSchema` (returns `422` on error); returns `404` when user not found; returns `409` on username conflict; auto-creates private board if role is promoted to `AdminUser` and user has no private board; returns updated user via `transformUser`; same middleware guards
  - [ ] AC-6: `POST /api/admin/user/update-password/[id]` — validates body with `updatePasswordSchema` (returns `422` on mismatch); returns `404` when user not found; verifies `current_password` via `verifyPassword` (returns `400`/`401` on mismatch); hashes `new_password` and persists; returns HTTP `200`; same middleware guards
  - [ ] AC-7: `DELETE /api/admin/user/[id]` — permanently deletes user; returns `404` when not found; returns `200` (or `204`) on success; same middleware guards
- **Dependencies:** `tasks/002-database-schema.md`, `tasks/004-authentication.md`, `tasks/005-api-transformers.md`, `tasks/006-search-table-service.md`, `tasks/008-notification-system.md`, Task 012-A, Task 012-B
- **Output Files:**
  - `server/api/admin/user/all.get.ts`
  - `server/api/admin/user/table.post.ts`
  - `server/api/admin/user/create.post.ts`
  - `server/api/admin/user/[id].get.ts`
  - `server/api/admin/user/[id].put.ts`
  - `server/api/admin/user/[id].delete.ts`
  - `server/api/admin/user/update-password/[id].post.ts`

---

## Wave 2 — Frontend (Frontend Agent)

### Task 012-D: Admin User Page, Composable & Modals

- **Agent:** Frontend
- **Spec:** `specs/admin-user-management.md`
- **Section:** AC-8, AC-9, Component Tree / File Structure, State Management
- **Acceptance Criteria:**
  - [ ] AC-8: `app/pages/user/index.vue` uses `default` layout; access restricted to `AdminUser` role via route middleware (redirect for `NormalUser`); renders a paginated, searchable user table driven by `POST /api/admin/user/table`
  - [ ] AC-8: `user-table.vue` manages local pagination/sort/filter state; uses `useAsyncData` with reactive keys; surfaces per-row action buttons for Add, Edit, Detail, Password, Delete
  - [ ] AC-9: `user-add-modal.vue` — form with `name`, `username`, `password`, `role`, `is_disable`; calls `POST /api/admin/user/create` via composable; refreshes table on success; surfaces Zod validation errors inline; API errors via `vue-sonner` toast
  - [ ] AC-9: `user-detail-modal.vue` — read-only display of all `UserResponse` fields; populated via `GET /api/admin/user/[id]`
  - [ ] AC-9: `user-edit-modal.vue` — pre-populated form with `name`, `username`, `role`, `is_disable`; calls `PUT /api/admin/user/[id]` via composable; refreshes table on success; surfaces Zod/API errors
  - [ ] AC-9: `user-password-modal.vue` — form with `current_password`, `new_password`, `confirm_password`; calls `POST /api/admin/user/update-password/[id]` via composable; surfaces Zod/API errors
  - [ ] AC-9: `user-delete-modal.vue` — confirmation prompt; calls `DELETE /api/admin/user/[id]` via composable; refreshes table on success; surfaces API errors via toast
  - [ ] All components use `<script setup lang="ts">`, `defineProps<T>()`, `defineEmits<T>()` — no Options API
  - [ ] `use-user-admin-api.ts` wraps all 7 endpoints; `useFetch`/`useAsyncData` for reads, `$fetch` for mutations; explicit return types on all composable functions
- **Dependencies:** Task 012-A, Task 012-B, Task 012-C
- **Output Files:**
  - `app/pages/user/index.vue`
  - `app/composables/api/use-user-admin-api.ts`
  - `app/components/user/user-table.vue`
  - `app/components/user/user-add-modal.vue`
  - `app/components/user/user-detail-modal.vue`
  - `app/components/user/user-edit-modal.vue`
  - `app/components/user/user-password-modal.vue`
  - `app/components/user/user-delete-modal.vue`

---

## Wave 3 — Tests (Testing Agent)

### Task 012-E: Unit & E2E Tests

- **Agent:** Testing
- **Spec:** `specs/admin-user-management.md`
- **Section:** AC-1 through AC-9, Edge Cases
- **Acceptance Criteria:**
  - [ ] AC-1: unit test — `all.get.ts` returns transformed user array; unauthenticated/non-admin returns correct status codes
  - [ ] AC-2: unit test — `table.post.ts` returns paginated result; invalid SearchTable params return `422`
  - [ ] AC-3: unit test — `[id].get.ts` returns user on valid ID; returns `404` on unknown ID
  - [ ] AC-4: unit test — `create.post.ts` happy path returns `201` with `UserResponse`; duplicate username returns `409`; invalid body returns `422`; password is hashed (stored hash ≠ plain); `initializeUserDefaults` is called
  - [ ] AC-5: unit test — `[id].put.ts` happy path returns updated user; unknown ID returns `404`; username conflict returns `409`; role promotion with no existing private board triggers board creation; role promotion with existing board skips duplicate creation
  - [ ] AC-6: unit test — `update-password/[id].post.ts` happy path returns `200`; `current_password` mismatch returns `400`/`401`; `new_password ≠ confirm_password` returns `422`; unknown ID returns `404`
  - [ ] AC-7: unit test — `[id].delete.ts` happy path returns `200`/`204`; unknown ID returns `404`
  - [ ] Edge case unit tests — unauthenticated → `401`; disabled user → `403`; non-admin → `403` (covers all 7 endpoints)
  - [ ] AC-8 / AC-9: e2e test — admin login → navigates to user page → creates user → edits user → views detail → changes password → deletes user; non-admin redirect verified
- **Dependencies:** Task 012-A, Task 012-B, Task 012-C, Task 012-D
- **Output Files:**
  - `tests/unit/server/api/admin/user/all.get.test.ts`
  - `tests/unit/server/api/admin/user/table.post.test.ts`
  - `tests/unit/server/api/admin/user/[id].get.test.ts`
  - `tests/unit/server/api/admin/user/create.post.test.ts`
  - `tests/unit/server/api/admin/user/[id].put.test.ts`
  - `tests/unit/server/api/admin/user/[id].delete.test.ts`
  - `tests/unit/server/api/admin/user/update-password.post.test.ts`
  - `tests/e2e/admin-users.spec.ts`
