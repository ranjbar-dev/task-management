# Tasks: Admin Notification Logs
**Spec:** `specs/admin-notification-logs.md`
**Status:** pending

---

## Overview

Implements a read-only notification log viewer for administrators. Covers two Nitro API endpoints (list-all and SearchTable-paginated), the `transformNotification` transformer, shared TypeScript interfaces, a Zod validation schema, and the frontend log viewer page with delivery status badges and filter controls. No mutation operations are exposed at any layer.

Execution order:
1. **Wave 1 (API Agent)** — shared types + Zod schema + transformer, then the two server route handlers.
2. **Wave 2 (Frontend Agent)** — API composable and the log viewer page (depends on Wave 1).
3. **Wave 3 (Testing Agent)** — unit tests for both endpoints and an e2e test for the log viewer page (depends on Waves 1 & 2).

---

## Wave 1 — Data Layer (API Agent)

### Task 014-A: Shared Types, Zod Schema & Notification Transformer

- **Agent:** API
- **Spec:** `specs/admin-notification-logs.md`
- **Section:** Data Model → TypeScript Interfaces, Data Model → Transformer, API Endpoints Consumed → SearchTable Request Body
- **Acceptance Criteria:**
  - [ ] AC-1 (partial): `transformNotification(row)` returns `{ id, type, notifiable, data, read_at, created_at }` — `created_at` formatted as `"Y-m-d H:i:s"`, `data` returned as raw JSON string, `notifiable` resolved from the `notifiable_id` foreign key.
  - [ ] AC-2 (partial): A Zod schema validates the SearchTable request body for the `/table` endpoint — fields: `page` (number), `per_page` (number), optional `sort` (`column` + `direction`), optional `filters` (`type?: string`, `notifiable_id?: number`, `read_at?: boolean | null`). Returns 422 on validation failure.
  - [ ] `NotificationModel` interface is defined in `shared/types/notification.types.ts` with fields: `id: string`, `type: string`, `notifiable: UserModel`, `data: string`, `read_at: string | null`, `created_at: string`.
  - [ ] `NotificationData` interface is defined in `shared/types/notification.types.ts` with `status?: 'pending' | 'sent' | 'failed'`.
- **Dependencies:** `tasks/005-api-transformers.md`, `tasks/006-search-table-service.md`, `tasks/008-notification-system.md`, `tasks/002-database-schema.md`
- **Output Files:**
  - `shared/types/notification.types.ts`
  - `shared/schemas/notification.schema.ts`
  - `server/transformers/notification.transformer.ts`

---

### Task 014-B: Admin Notification API Endpoints

- **Agent:** API
- **Spec:** `specs/admin-notification-logs.md`
- **Section:** Acceptance Criteria → AC-1, AC-2
- **Acceptance Criteria:**
  - [ ] AC-1: `GET /api/admin/notification/all` returns all notifications (no pagination) as an array of transformed `NotificationModel` objects using `transformNotification`.
  - [ ] AC-1: Endpoint is protected by `auth`, `active-user`, and `admin` middleware (`UserRoleEnum.AdminUser`).
  - [ ] AC-1: Returns `403` if the authenticated user is not an admin.
  - [ ] AC-1: Returns `401` if no valid JWT is present.
  - [ ] AC-2: `POST /api/admin/notification/table` accepts a `SearchTableRequest` body and returns `{ data: NotificationModel[], meta: PaginationMeta }`.
  - [ ] AC-2: Default sort is `created_at` DESC (most recent first).
  - [ ] AC-2: Results are filterable by `type` (exact or partial match).
  - [ ] AC-2: Results are filterable by `notifiable_id` (recipient user ID).
  - [ ] AC-2: Results are filterable by `read_at` (`null` → unread, non-null → read).
  - [ ] AC-2: Response envelope includes pagination metadata: `total`, `per_page`, `current_page`, `last_page`.
  - [ ] AC-2: Endpoint is protected by `auth`, `active-user`, and `admin` middleware.
  - [ ] AC-2: Returns `422` if the request body fails Zod validation.
- **Dependencies:** Task 014-A, `tasks/004-authentication.md`, `tasks/005-api-transformers.md`, `tasks/006-search-table-service.md`, `tasks/008-notification-system.md`, `tasks/002-database-schema.md`
- **Output Files:**
  - `server/api/admin/notification/all.get.ts`
  - `server/api/admin/notification/table.post.ts`

---

## Wave 2 — Frontend (Frontend Agent)

### Task 014-C: Admin Notification Log Viewer Page

- **Agent:** Frontend
- **Spec:** `specs/admin-notification-logs.md`
- **Section:** Acceptance Criteria → AC-3, AC-4, AC-5; State Management; Edge Cases
- **Acceptance Criteria:**
  - [ ] AC-3: Page at `app/pages/notification/index.vue` is accessible only to authenticated admin users; non-admins are redirected to home via route middleware.
  - [ ] AC-3: Page uses the `default` layout (`definePageMeta({ layout: 'default' })`).
  - [ ] AC-3: Renders a searchable, paginated table using the `Table/Search` UI component.
  - [ ] AC-3: Table displays columns: **Type**, **Recipient**, **Delivery Status**, **Read At**, **Created At**.
  - [ ] AC-3: Delivery status is parsed from the `data` JSON field and displayed as a badge/chip: `pending` → neutral/grey, `sent` → green/success, `failed` → red/destructive.
  - [ ] AC-3: `read_at` displays as a formatted datetime string when present, or "Unread" when `null`.
  - [ ] AC-3: `created_at` is displayed in `"Y-m-d H:i:s"` format (as returned by the transformer).
  - [ ] AC-3: Filter controls allow filtering by notification type (dropdown of all 18 types), recipient user, and read/unread status.
  - [ ] AC-3: Table re-fetches (or sorts client-side) on column header click; `created_at` is the default sort column with DESC direction.
  - [ ] AC-3: No create, edit, or delete actions are exposed anywhere on the page.
  - [ ] AC-3: Loading state is shown while the table data is fetching.
  - [ ] AC-3: Empty state ("No notifications found") is shown when no notifications match the current filters.
  - [ ] AC-4: `data` field is `JSON.parse()`d on the client; the `status` property is extracted.
  - [ ] AC-4: If `status` is missing, unrecognised, or `data` is not valid JSON, the cell renders "Unknown" without throwing.
  - [ ] AC-4: `pending` → neutral/grey badge, `sent` → green/success badge, `failed` → red/destructive badge.
  - [ ] AC-5: All 18 notification type strings are represented in the type filter dropdown and render without error in the Type column.
  - [ ] **Edge cases:** Network or server error on table fetch shows an error toast and preserves the previous table state. Non-admin redirect is enforced. `data` parse failure renders "Unknown" silently.
  - [ ] **Composable:** `useAdminNotificationApi()` exposes `fetchAll(): Promise<NotificationModel[]>` and `fetchTable(params: SearchTableRequest): Promise<{ data: NotificationModel[], meta: PaginationMeta }>` with explicit return types. Uses `$fetch` for mutations and `useFetch`/`useAsyncData` for reads per project convention.
  - [ ] **Page-level state:** `notifications`, `pagination`, `filters`, `sort`, `loading` are managed locally (no Pinia store).
- **Dependencies:** Task 014-A, Task 014-B, `tasks/004-authentication.md`
- **Output Files:**
  - `app/composables/api/use-admin-notification-api.ts`
  - `app/pages/notification/index.vue`

---

## Wave 3 — Tests (Testing Agent)

### Task 014-D: Notification Log Tests

- **Agent:** Testing
- **Spec:** `specs/admin-notification-logs.md`
- **Section:** All ACs
- **Acceptance Criteria:**
  - [ ] **Unit — `all.get.ts`:** Happy path returns 200 with an array of transformed notification objects. Returns 401 with no/invalid JWT. Returns 403 when authenticated user is not an admin.
  - [ ] **Unit — `table.post.ts`:** Happy path returns 200 with `{ data, meta }` envelope and correct `PaginationMeta` fields. Default sort is `created_at` DESC. Filter by `type` narrows results. Filter by `notifiable_id` narrows results. Filter by `read_at` (null → unread, non-null → read) narrows results. Returns 422 on invalid request body (missing `page`, missing `per_page`, unknown field types). Returns 401 with no JWT. Returns 403 for non-admin user.
  - [ ] **Unit — transformer:** `transformNotification` formats `created_at` as `"Y-m-d H:i:s"`, returns `data` as raw JSON string, resolves `notifiable` from the user record.
  - [ ] **e2e — log viewer page:** Admin can navigate to `/notification` and see the paginated table. Applying a type filter re-fetches and narrows the results. Applying a read/unread filter updates the table. Clicking the `created_at` column header toggles sort direction. Non-admin user visiting `/notification` is redirected. Empty-state message is shown when filters produce no results.
- **Dependencies:** Task 014-A, Task 014-B, Task 014-C
- **Output Files:**
  - `tests/unit/server/api/admin/notification/all.get.test.ts`
  - `tests/unit/server/api/admin/notification/table.post.test.ts`
  - `tests/unit/server/transformers/notification.transformer.test.ts`
  - `tests/e2e/admin/notification-logs.spec.ts`
