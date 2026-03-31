# Tasks: Admin Tag Management
**Spec:** `specs/admin-tag-management.md`
**Status:** pending

---

## Overview

Full CRUD control over globally-defined tags used in the task management system. Tags have a `name`, a `color` (one of 9 `TagColorEnum` values stored as a `smallint`), and an optional `description`. They are attached to cards via a JSONB array of IDs on `cards.tags`.

Work is split across three waves:

| Wave | Agent | Scope |
|------|-------|-------|
| 1 | API | Enum, Zod schemas, transformer, 7 API endpoints, client composable + types |
| 2 | Frontend | Tag chip component, color picker component, admin page, 3 modals |
| 3 | Testing | Unit tests (transformer + endpoints) and E2E tests |

---

## Wave 1 — Data Layer (API Agent)

### Task 013-A: TagColorEnum + Zod Validation Schemas

- **Agent:** API
- **Spec:** `specs/admin-tag-management.md`
- **Section:** Data Model → TagColorEnum; Dependencies table; AC-3; AC-4; AC-9
- **Acceptance Criteria:**
  - [ ] AC-3: `createTagSchema` validates `name` (required, 1–50 chars), `color` (required, integer 0–8), `description` (optional, max 255 chars)
  - [ ] AC-4: `updateTagSchema` is `createTagSchema.partial()` — all fields optional
  - [ ] AC-8: `TagColorEnum` defines all 9 entries: `Red=0, Purple=1, Green=2, Orange=3, Blue=4, Gray=5, Cyan=6, Magenta=7, Violet=8`
  - [ ] AC-9: Color picker (consumer) can derive all 9 options from the enum
- **Dependencies:** `tasks/003-validation-schemas.md`
- **Output Files:**
  - `shared/enums/TagColorEnum.ts`
  - `shared/schemas/tag.schema.ts`

---

### Task 013-B: Tag Transformer

- **Agent:** API
- **Spec:** `specs/admin-tag-management.md`
- **Section:** AC-8; TypeScript Interfaces → `TagModel`
- **Acceptance Criteria:**
  - [ ] AC-8: `transformTag` accepts a raw `tags` DB row and returns `TagModel`
  - [ ] AC-8: Return shape is `{ id, name, description, color: { name: string, value: number } }`
  - [ ] AC-8: `color.name` is the `TagColorEnum` key derived from the stored integer (e.g., `"Red"`)
  - [ ] AC-8: `color.value` is the raw `smallint` stored in the DB
  - [ ] AC-8: All 9 `TagColorEnum` entries map correctly without fallback errors
- **Dependencies:** `tasks/005-api-transformers.md`, Task 013-A
- **Output Files:**
  - `server/transformers/tag.transformer.ts`

---

### Task 013-C: Admin Tag API Endpoints (6 routes)

- **Agent:** API
- **Spec:** `specs/admin-tag-management.md`
- **Section:** AC-1 through AC-6; AC-12; Edge Cases table
- **Acceptance Criteria:**
  - [ ] AC-1: `GET /api/admin/tag/all` — returns all tags transformed via `transformTag`; each item includes `id`, `name`, `description`, `color: { name, value }`
  - [ ] AC-1: Returns `401`/`403` for unauthenticated or non-admin requests
  - [ ] AC-2: `POST /api/admin/tag/table` — accepts `SearchTableRequest`; returns paginated result via `SearchTable` service; supports text search on `name` and `description`; supports sort by `id`, `name`, `createdAt`
  - [ ] AC-2: Requires `AdminUser` role
  - [ ] AC-3: `POST /api/admin/tag/create` — validates body with `createTagSchema`; rejects duplicate `name` with `422` and `"Tag name already exists"` message; defaults `description` to `"default"` when omitted; returns created tag via `transformTag`
  - [ ] AC-3: Requires `AdminUser` role
  - [ ] AC-4: `PUT /api/admin/tag/[id]` — validates body with `updateTagSchema`; returns `404` when tag not found; rejects duplicate `name` with `422`; returns updated tag via `transformTag`
  - [ ] AC-4: Requires `AdminUser` role
  - [ ] AC-5: `DELETE /api/admin/tag/[id]` — returns `404` when tag not found; deletes row from `tags` table; **does not** cascade-clean `cards.tags` JSONB; returns `{ success: true }`
  - [ ] AC-5: Requires `AdminUser` role
  - [ ] AC-6: `GET /api/admin/tag/[id]` — returns single tag via `transformTag`; returns `404` when not found
  - [ ] AC-6: Requires `AdminUser` role
  - [ ] AC-12: Every admin route is protected by middleware chain in order: `auth` → `active-user` → `admin`
  - [ ] AC-12: Missing/invalid JWT → `401`; disabled account → `403`; non-admin role → `403`
- **Dependencies:** `tasks/002-database-schema.md`, `tasks/004-authentication.md`, `tasks/005-api-transformers.md`, `tasks/006-search-table-service.md`, Task 013-A, Task 013-B
- **Output Files:**
  - `server/api/admin/tag/all.get.ts`
  - `server/api/admin/tag/table.post.ts`
  - `server/api/admin/tag/create.post.ts`
  - `server/api/admin/tag/[id].put.ts`
  - `server/api/admin/tag/[id].delete.ts`
  - `server/api/admin/tag/[id].get.ts`

---

### Task 013-D: Client Read-Only Tag Endpoint

- **Agent:** API
- **Spec:** `specs/admin-tag-management.md`
- **Section:** AC-7; AC-12 (middleware note)
- **Acceptance Criteria:**
  - [ ] AC-7: `GET /api/client/tag/all` — returns all tags via `transformTag` (same shape as AC-1)
  - [ ] AC-7: Protected by `auth` + `active-user` middleware only — **no** `admin` middleware
  - [ ] AC-7: Any authenticated, active user can call this endpoint; non-admin authenticated users receive `200`
  - [ ] AC-12: Missing/invalid JWT → `401`; disabled account → `403`
- **Dependencies:** `tasks/004-authentication.md`, `tasks/005-api-transformers.md`, Task 013-A, Task 013-B
- **Output Files:**
  - `server/api/client/tag/all.get.ts`

---

### Task 013-E: Client-Side Composable + TypeScript Types

- **Agent:** API
- **Spec:** `specs/admin-tag-management.md`
- **Section:** TypeScript Interfaces; API Endpoints Consumed table; State Management
- **Acceptance Criteria:**
  - [ ] `TagModel` interface is defined and exported
  - [ ] `CreateTagRequest` and `UpdateTagRequest` interfaces are defined and exported
  - [ ] `use-admin-tag-api` composable exposes typed wrappers for all 6 admin tag endpoints:
    - `getAllTags()` → `GET /api/admin/tag/all`
    - `getTagTable(body)` → `POST /api/admin/tag/table`
    - `createTag(body)` → `POST /api/admin/tag/create`
    - `updateTag(id, body)` → `PUT /api/admin/tag/[id]`
    - `deleteTag(id)` → `DELETE /api/admin/tag/[id]`
    - `getTag(id)` → `GET /api/admin/tag/[id]`
  - [ ] All composable functions use `$fetch` for mutations (POST/PUT/DELETE) and `useFetch`/`useAsyncData` for reads where appropriate
  - [ ] Explicit return types on all composable functions
- **Dependencies:** Task 013-C, Task 013-D
- **Output Files:**
  - `app/types/tag.types.ts`
  - `app/composables/api/use-admin-tag-api.ts`

---

## Wave 2 — Frontend (Frontend Agent)

### Task 013-F: Tag Chip Component

- **Agent:** Frontend
- **Spec:** `specs/admin-tag-management.md`
- **Section:** AC-10
- **Acceptance Criteria:**
  - [ ] AC-10: `Tag.vue` accepts a `TagModel` prop
  - [ ] AC-10: Renders tag `name` with background/border color derived from `color.name` (or `color.value`)
  - [ ] AC-10: Color rendering covers all 9 `TagColorEnum` variants with distinct, accessible styles:
    - `Red`, `Purple`, `Green`, `Orange`, `Blue`, `Gray`, `Cyan`, `Magenta`, `Violet`
  - [ ] Uses `<script setup lang="ts">` and `defineProps<{ tag: TagModel }>()`
- **Dependencies:** Task 013-E
- **Output Files:**
  - `app/components/ui/Tag.vue`

---

### Task 013-G: Tag Color Picker Component

- **Agent:** Frontend
- **Spec:** `specs/admin-tag-management.md`
- **Section:** AC-9
- **Acceptance Criteria:**
  - [ ] AC-9: Renders all 9 `TagColorEnum` options: `Red`, `Purple`, `Green`, `Orange`, `Blue`, `Gray`, `Cyan`, `Magenta`, `Violet`
  - [ ] AC-9: Each option renders with its actual color swatch for visual identification
  - [ ] AC-9: Emits the selected color's integer value (`TagColorEnum` value) on selection, not the key string
  - [ ] Accepts a `modelValue` prop (current integer value) and emits `update:modelValue` (v-model compatible)
  - [ ] Uses `<script setup lang="ts">` with typed props and emits
- **Dependencies:** Task 013-A, Task 013-E
- **Output Files:**
  - `app/components/ui/TagColorPicker.vue`

---

### Task 013-H: Admin Tag Page + Modals

- **Agent:** Frontend
- **Spec:** `specs/admin-tag-management.md`
- **Section:** AC-11; State Management; Edge Cases table
- **Acceptance Criteria:**
  - [ ] AC-11: `app/pages/tag/index.vue` uses the `default` layout (`definePageMeta({ layout: 'default' })`)
  - [ ] AC-11: Page displays a `Table/Search` component backed by `POST /api/admin/tag/table`
  - [ ] AC-11: Page provides an **Add Tag** button that opens `AddTagModal`
  - [ ] AC-11: Each table row has **Edit** and **Delete** action buttons
  - [ ] AC-11: **Add Tag modal** contains: name input, `TagColorPicker`, description textarea; submits to `POST /api/admin/tag/create` via composable
  - [ ] AC-11: **Edit Tag modal** is pre-populated with existing tag data; submits to `PUT /api/admin/tag/[id]`
  - [ ] AC-11: **Delete confirmation modal** shows the tag name; submits to `DELETE /api/admin/tag/[id]`; includes a warning that card references are not automatically removed
  - [ ] AC-11: On successful create/update/delete, the table data refreshes
  - [ ] AC-11: API validation errors (e.g., duplicate name `422`) are surfaced as inline error messages inside the open modal — not as toast-only
  - [ ] No dedicated Pinia store — local `ref` state for modals (open/close, form values, loading, error)
  - [ ] Displays a `Tag.vue` chip for each tag's color in the table rows
- **Dependencies:** Task 013-E, Task 013-F, Task 013-G
- **Output Files:**
  - `app/pages/tag/index.vue`
  - `app/components/modal/tag/AddTagModal.vue`
  - `app/components/modal/tag/EditTagModal.vue`
  - `app/components/modal/tag/DeleteTagModal.vue`

---

## Wave 3 — Tests (Testing Agent)

### Task 013-I: API Unit Tests

- **Agent:** Testing
- **Spec:** `specs/admin-tag-management.md`
- **Section:** AC-1 through AC-8; AC-12; Edge Cases table
- **Acceptance Criteria:**
  - [ ] `transformTag` unit tests:
    - Maps all 9 `TagColorEnum` values to the correct `color.name` string and `color.value` integer
    - Returns correct `TagModel` shape for a well-formed DB row
  - [ ] `GET /api/admin/tag/all` — happy path returns transformed array; non-admin → `403`; unauthenticated → `401`
  - [ ] `POST /api/admin/tag/table` — happy path returns paginated result; search on `name`/`description` filters correctly; sort by `id`, `name`, `createdAt` works
  - [ ] `POST /api/admin/tag/create` — happy path creates and returns tag; duplicate name → `422` with expected message; `name` > 50 chars → `422`; `description` > 255 chars → `422`; `color` outside 0–8 → `422`; missing `description` defaults to `"default"`
  - [ ] `PUT /api/admin/tag/[id]` — happy path updates and returns tag; non-existent ID → `404`; duplicate name → `422`; empty body → no-op, tag returned unchanged
  - [ ] `DELETE /api/admin/tag/[id]` — happy path returns `{ success: true }`; non-existent ID → `404`; does not touch `cards.tags` JSONB
  - [ ] `GET /api/admin/tag/[id]` — happy path returns transformed tag; non-existent ID → `404`
  - [ ] `GET /api/client/tag/all` — authenticated non-admin user receives `200`; unauthenticated → `401`; disabled user → `403`
  - [ ] AC-12: Disabled account → `403` on all admin routes; non-admin role → `403` on all admin routes
- **Dependencies:** Task 013-A, Task 013-B, Task 013-C, Task 013-D
- **Output Files:**
  - `tests/unit/server/transformers/tag.transformer.test.ts`
  - `tests/unit/server/api/admin/tag/all.get.test.ts`
  - `tests/unit/server/api/admin/tag/table.post.test.ts`
  - `tests/unit/server/api/admin/tag/create.post.test.ts`
  - `tests/unit/server/api/admin/tag/[id].put.test.ts`
  - `tests/unit/server/api/admin/tag/[id].delete.test.ts`
  - `tests/unit/server/api/admin/tag/[id].get.test.ts`
  - `tests/unit/server/api/client/tag/all.get.test.ts`

---

### Task 013-J: E2E Tests — Admin Tag Management Page

- **Agent:** Testing
- **Spec:** `specs/admin-tag-management.md`
- **Section:** AC-11; Edge Cases table
- **Acceptance Criteria:**
  - [ ] Admin can navigate to the tag management page and see the tag table
  - [ ] Admin can open the **Add Tag** modal, fill in name + select a color, submit, and see the new tag appear in the table
  - [ ] Admin can open the **Edit Tag** modal for an existing tag, change the name, submit, and see the updated name in the table
  - [ ] Admin can open the **Delete confirmation** modal, confirm, and see the tag removed from the table
  - [ ] Submitting a duplicate tag name in Add modal shows inline error inside the modal
  - [ ] Submitting a duplicate tag name in Edit modal shows inline error inside the modal
  - [ ] Delete modal displays a warning about card references not being automatically removed
- **Dependencies:** Task 013-A through Task 013-H
- **Output Files:**
  - `tests/e2e/admin-tag-management.spec.ts`
