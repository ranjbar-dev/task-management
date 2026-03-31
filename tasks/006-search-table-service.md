# Tasks: SearchTable Service
**Spec:** `specs/search-table-service.md`
**Status:** pending

---

## Overview

Implement the reusable server-side `handleSearchTable` service (`server/services/searchTable.ts`) that centralises filtering, sorting, and pagination for all `table` POST endpoints across admin and client API routes. The service accepts a Drizzle base query and a validated request body and returns a consistent paginated or full-collection response. Alongside the core service, this task covers the shared Zod schema, shared TypeScript interfaces, all six consumer route handlers, and a full unit test suite.

Wave 1 builds the core service and all its consumers (API Agent). Wave 2 is not applicable — the spec does not describe any new frontend component (the existing `Table/Search` UI components consume the consistent response shape but are not scoped here). Wave 3 writes the unit tests (Testing Agent).

---

## Wave 1 — Data Layer (API Agent)

### Task 006-A: Zod Schema & Shared TypeScript Interfaces

- **Agent:** API
- **Spec:** `specs/search-table-service.md`
- **Section:** AC-15 — Zod Schema; Data Model → TypeScript Interfaces (`PaginationMeta`, `PaginatedResponse<T>`, `FullResponse<T>`)
- **Acceptance Criteria:**
  - [ ] AC-15: `shared/schemas/searchTable.schema.ts` exists and exports `searchTableSchema` and the inferred `SearchTableBody` type.
  - [ ] AC-15: Each item in the `search` array is validated to have `key` (string), `type` (enum of exactly 8 values: `'like' | 'equal' | 'date' | 'datetime-bool' | 'in' | 'relation-equal' | 'relation-like' | 'bool'`), and `value` (`z.any()`).
  - [ ] AC-15: `sort.direction` is validated as `z.enum(['asc', 'desc'])`.
  - [ ] AC-15: `per_page` and `page` are `z.number().int().positive()` when provided.
  - [ ] AC-15: `all` is `z.boolean()` when provided.
  - [ ] AC-15: All top-level fields (`search`, `sort`, `per_page`, `page`, `all`) are optional.
  - [ ] `shared/types/api.ts` exports `PaginationMeta`, `PaginatedResponse<T>`, and `FullResponse<T>` interfaces matching the Data Model in the spec exactly.
  - [ ] No default exports — only named exports in both files.
- **Dependencies:** `tasks/001-infrastructure-setup.md`, `tasks/002-database-schema.md`, `tasks/003-validation-schemas.md`
- **Output Files:**
  - `shared/schemas/searchTable.schema.ts`
  - `shared/types/api.ts`

---

### Task 006-B: Core `handleSearchTable` Service

- **Agent:** API
- **Spec:** `specs/search-table-service.md`
- **Section:** AC-1 through AC-14; Data Model → `SearchTableOptions`; Edge Cases table
- **Acceptance Criteria:**
  - [ ] AC-1: File exports a single async function `handleSearchTable(options: SearchTableOptions): Promise<PaginatedResponse | FullResponse>`.
  - [ ] AC-1: `SearchTableOptions` has `query` (Drizzle select query chain) and `body` (`SearchTableBody` from `searchTableSchema`).
  - [ ] AC-2 (`like`): Applies case-insensitive `ILIKE '%value%'` to the target column; only valid for string columns; multiple `like` filters are AND-composed.
  - [ ] AC-3 (`equal`): Applies strict SQL `=` equality; accepts string, number, or boolean scalar values.
  - [ ] AC-4 (`date`): Accepts `{ from?: string; to?: string }` (ISO 8601); applies `>=` for `from`, `<=` for `to`; both bounds are optional and may be combined; works on `date` and `timestamp` columns.
  - [ ] AC-5 (`datetime-bool`): `value: true` → `IS NOT NULL`; `value: false` → `IS NULL`; intended for nullable timestamp columns.
  - [ ] AC-6 (`in`): Generates SQL `IN (...)`; an empty array immediately returns `{ items: [], pagination: { total: 0, per_page, current_page: page, last_page: 1 } }` without executing the base query.
  - [ ] AC-7 (`relation-equal`): Applies exact equality on a column from a joined table; relies on caller having included the join in the base query.
  - [ ] AC-8 (`relation-like`): Applies ILIKE on a column from a joined table; relies on caller having included the join in the base query.
  - [ ] AC-9 (`bool`): Applies `= true` or `= false` to an actual boolean column (distinct from `datetime-bool`).
  - [ ] AC-10: When a `like` or `equal` filter targets a column of type `amount`/`price` (decimal/numeric), the value is coerced to `float` before comparison.
  - [ ] AC-11: Sorts results by `sort.key` in `sort.direction` (`'asc'` or `'desc'`); sorting is applied after all filters; when `sort` is omitted no explicit `ORDER BY` is added.
  - [ ] AC-12: Default `per_page` is 15 when omitted; default `page` is 1 when omitted; returns `items`, `pagination.total`, `pagination.per_page`, `pagination.current_page`, `pagination.last_page`; `last_page = Math.ceil(total / per_page)` with a minimum of 1; out-of-range `page` returns empty `items` with accurate pagination metadata.
  - [ ] AC-13: When `body.all === true`, skips pagination and returns `{ status: 200, data: { items: T[] } }` — no `pagination` key; filters and sorting still apply.
  - [ ] AC-14: All responses are wrapped in `{ status: 200, data: { ... } }`; on invalid body, throws `H3Error` with status 422.
  - [ ] Edge case: empty `search` array `[]` adds no WHERE clauses; all rows returned subject to pagination.
  - [ ] Edge case: `page` exceeding `last_page` returns `{ items: [] }` with correct `total` and `last_page`.
  - [ ] Edge case: `all: true` combined with `sort` still applies the sort before returning.
  - [ ] Edge case: multiple filters of the same type on different columns are all AND-composed.
- **Dependencies:** Task 006-A, `tasks/001-infrastructure-setup.md`, `tasks/002-database-schema.md`, `tasks/003-validation-schemas.md`
- **Output Files:**
  - `server/services/searchTable.ts`

---

### Task 006-C: Admin Consumer Route Handlers

- **Agent:** API
- **Spec:** `specs/search-table-service.md`
- **Section:** AC-16, AC-17, AC-18, AC-19; Example Usage; Component Tree / File Structure
- **Acceptance Criteria:**
  - [ ] AC-16: `POST /api/admin/board/table` — base query selects from `boards` with `leftJoin(boardUsers, eq(boards.id, boardUsers.boardId))`; body validated with `searchTableSchema`; result from `handleSearchTable` returned directly; protected by `auth` + `admin` server middleware.
  - [ ] AC-17: `POST /api/admin/user/table` — base query selects from `users`; body validated with `searchTableSchema`; protected by `auth` + `admin` server middleware.
  - [ ] AC-18: `POST /api/admin/tag/table` — base query selects from `tags`; body validated with `searchTableSchema`; protected by `auth` + `admin` server middleware.
  - [ ] AC-19: `POST /api/admin/notification/table` — base query selects from `notifications`; notifications ordered by `created_at DESC` by default in the base query; body validated with `searchTableSchema`; protected by `auth` + `admin` server middleware.
  - [ ] Each handler is a thin wrapper: validates body with `searchTableSchema.parse(await readBody(event))`, builds the base query, calls `handleSearchTable({ query, body })`, and returns the result.
  - [ ] Each file includes the comment `// Spec: specs/search-table-service.md` at the top.
- **Dependencies:** Task 006-A, Task 006-B, `tasks/002-database-schema.md`
- **Output Files:**
  - `server/api/admin/board/table.post.ts`
  - `server/api/admin/user/table.post.ts`
  - `server/api/admin/tag/table.post.ts`
  - `server/api/admin/notification/table.post.ts`

---

### Task 006-D: Board Logs Consumer Route Handlers (Admin + Client)

- **Agent:** API
- **Spec:** `specs/search-table-service.md`
- **Section:** AC-20; Component Tree / File Structure
- **Acceptance Criteria:**
  - [ ] AC-20: `POST /api/admin/board/logs-table/[id]` — admin has access to any board's logs; validates board exists; body validated with `searchTableSchema`; delegates to `handleSearchTable`; protected by `auth` + `admin` server middleware.
  - [ ] AC-20: `POST /api/client/board/logs-table/[id]` — client access is limited to boards the user belongs to; validates board membership before calling `handleSearchTable`; returns 403 if the requesting user is not a member of the board; protected by `auth` server middleware.
  - [ ] Both handlers validate the `[id]` route param as a positive integer; returns 404 if the board does not exist.
  - [ ] Each file includes the comment `// Spec: specs/search-table-service.md` at the top.
- **Dependencies:** Task 006-A, Task 006-B, `tasks/002-database-schema.md`
- **Output Files:**
  - `server/api/admin/board/logs-table/[id].post.ts`
  - `server/api/client/board/logs-table/[id].post.ts`

---

## Wave 2 — Frontend (Frontend Agent)

_Not applicable._ The spec explicitly scopes this feature to the server-side service and consuming API endpoints. The frontend `Table/Search` UI component already consumes the consistent response envelope and is not modified by this feature.

---

## Wave 3 — Tests (Testing Agent)

### Task 006-E: Unit Tests — `handleSearchTable` Core Service

- **Agent:** Testing
- **Spec:** `specs/search-table-service.md`
- **Section:** AC-1 through AC-14; Edge Cases table
- **Acceptance Criteria:**
  - [ ] Happy path: paginated response with `search`, `sort`, `per_page`, `page` returns correct `items` and `pagination` metadata.
  - [ ] Happy path: `all: true` returns all matching rows with no `pagination` key.
  - [ ] Filter `like`: ILIKE pattern applied; multiple filters AND-composed.
  - [ ] Filter `equal`: strict equality applied; boolean and numeric values accepted.
  - [ ] Filter `date`: `from` only, `to` only, and both bounds tested; empty result for `from > to`.
  - [ ] Filter `datetime-bool`: `true` → `IS NOT NULL`; `false` → `IS NULL`.
  - [ ] Filter `in`: array with values returns matching rows; empty array returns `items: []` with `total: 0`.
  - [ ] Filter `relation-equal`: applies equality on a joined column.
  - [ ] Filter `relation-like`: applies ILIKE on a joined column.
  - [ ] Filter `bool`: `true` and `false` equality on a boolean column.
  - [ ] AC-10: amount/price columns are coerced to float before comparison.
  - [ ] Sorting: `asc` and `desc` ordering are applied after filters; omitting `sort` produces no ORDER BY.
  - [ ] Pagination defaults: `per_page` defaults to 15; `page` defaults to 1.
  - [ ] Out-of-range `page` returns `items: []` with accurate `total` and `last_page`.
  - [ ] `last_page` is always at least 1 even when `total` is 0.
  - [ ] Empty `search` array adds no WHERE clauses.
  - [ ] Invalid body (missing required `key` field, invalid `type` enum) causes `handleSearchTable` to throw an H3Error with status 422.
- **Dependencies:** Task 006-A, Task 006-B
- **Output Files:**
  - `tests/unit/server/services/searchTable.test.ts`

---

### Task 006-F: Unit Tests — Zod Schema

- **Agent:** Testing
- **Spec:** `specs/search-table-service.md`
- **Section:** AC-15
- **Acceptance Criteria:**
  - [ ] Valid body with all fields passes `searchTableSchema.parse()` without error.
  - [ ] Completely empty body `{}` passes (all fields optional).
  - [ ] Invalid `type` value in a search item (e.g. `'fuzzy'`) fails with a ZodError.
  - [ ] Invalid `sort.direction` (e.g. `'random'`) fails with a ZodError.
  - [ ] Negative `per_page` or `page` value fails with a ZodError.
  - [ ] `per_page: 0` fails with a ZodError (must be positive).
  - [ ] `all: "true"` (string) fails with a ZodError (must be boolean).
  - [ ] Inferred `SearchTableBody` type is assignable from a fully-populated valid object at compile time.
- **Dependencies:** Task 006-A
- **Output Files:**
  - `tests/unit/shared/schemas/searchTable.schema.test.ts`

---

### Task 006-G: Unit Tests — Consumer Route Handlers

- **Agent:** Testing
- **Spec:** `specs/search-table-service.md`
- **Section:** AC-16 through AC-20
- **Acceptance Criteria:**
  - [ ] AC-16: Admin board table handler returns paginated board data; non-admin request is rejected with 403.
  - [ ] AC-17: Admin user table handler returns paginated user data; non-admin request is rejected with 403.
  - [ ] AC-18: Admin tag table handler returns paginated tag data; non-admin request is rejected with 403.
  - [ ] AC-19: Admin notification table handler returns paginated notification data ordered by `created_at DESC` by default; non-admin request is rejected with 403.
  - [ ] AC-20 (admin logs): Admin board logs handler returns logs for any board; non-admin request rejected with 403; non-existent board returns 404.
  - [ ] AC-20 (client logs): Client board logs handler returns logs when the requesting user is a board member; non-member request is rejected with 403; non-existent board returns 404.
  - [ ] Invalid request body (fails `searchTableSchema`) returns 422 for all handlers.
  - [ ] Unauthenticated request is rejected with 401 for all handlers.
- **Dependencies:** Task 006-A, Task 006-B, Task 006-C, Task 006-D
- **Output Files:**
  - `tests/unit/server/api/admin/board/table.test.ts`
  - `tests/unit/server/api/admin/user/table.test.ts`
  - `tests/unit/server/api/admin/tag/table.test.ts`
  - `tests/unit/server/api/admin/notification/table.test.ts`
  - `tests/unit/server/api/admin/board/logs-table.test.ts`
  - `tests/unit/server/api/client/board/logs-table.test.ts`
