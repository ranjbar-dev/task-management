# Spec: SearchTable Service

## Status: Approved

## Purpose

Provide a reusable, server-side data table engine (`server/services/searchTable.ts`) consumed by all admin and client `table` POST endpoints. The service centralises filtering, sorting, and pagination logic so individual route handlers stay thin — they supply a base Drizzle query plus a validated request body, and the service returns a consistent paginated (or full-collection) response.

---

## User Stories

- **As a backend developer**, I want a single function (`handleSearchTable`) that I can call from any `table` endpoint so that filtering, sorting, and pagination never have to be re-implemented per resource.
- **As a frontend developer**, I want the response shape to be identical across every table endpoint so that the shared `Table/Search` UI component works against any resource without per-endpoint customisation.
- **As an admin user**, I want to filter, sort, and paginate boards, users, tags, and notifications from the admin panel.
- **As a client user**, I want to filter and sort card activity log tables for boards I belong to.

---

## Acceptance Criteria

### AC-1 — Service Signature
- [ ] The service exports a single async function: `handleSearchTable(options: SearchTableOptions): Promise<PaginatedResponse | FullResponse>`
- [ ] `SearchTableOptions` accepts a `query` (Drizzle select query) and a `body` object matching the validated `searchTableSchema`

### AC-2 — Filter: `like`
- [ ] Applies a case-insensitive SQL `ILIKE` pattern (`%value%`) to the specified column
- [ ] Works on `string` column types only
- [ ] Multiple `like` filters in the `search` array are AND-composed

### AC-3 — Filter: `equal`
- [ ] Applies a strict equality (`=`) condition to the specified column
- [ ] Accepts any scalar value (string, number, boolean)

### AC-4 — Filter: `date`
- [ ] Accepts a value in the shape `{ from?: string; to?: string }` (ISO 8601 date strings)
- [ ] Applies `>=` when `from` is provided
- [ ] Applies `<=` when `to` is provided
- [ ] Both bounds are optional and may be combined
- [ ] Works on `date` and `timestamp` column types

### AC-5 — Filter: `datetime-bool`
- [ ] `value: true` → column `IS NOT NULL`
- [ ] `value: false` → column `IS NULL`
- [ ] Intended for nullable `timestamp` columns (e.g. `deletedAt`, `verifiedAt`)

### AC-6 — Filter: `in`
- [ ] Accepts an array value and generates SQL `IN (...)`
- [ ] An empty array produces no results (returns empty items list, total = 0)

### AC-7 — Filter: `relation-equal`
- [ ] Applies exact equality on a column from a joined/related table
- [ ] Requires the `query` to already include the relevant `leftJoin` / `innerJoin`

### AC-8 — Filter: `relation-like`
- [ ] Applies case-insensitive ILIKE on a column from a joined/related table
- [ ] Requires the `query` to already include the relevant join

### AC-9 — Filter: `bool`
- [ ] Applies boolean equality (`= true` or `= false`) to the specified column
- [ ] Distinct from `datetime-bool`: operates on actual boolean columns, not nullable timestamps

### AC-10 — Float Conversion
- [ ] Columns of type `amount` / `price` (decimal/numeric in Postgres) are coerced to `float` before comparison when a `like` or `equal` filter targets them

### AC-11 — Sorting
- [ ] Sorts results by the column identified by `sort.key`
- [ ] `sort.direction` is either `'asc'` or `'desc'`
- [ ] If `sort` is omitted, no explicit ORDER BY is applied (Postgres natural order)
- [ ] Sorting is applied after all filters

### AC-12 — Pagination (default paginated mode)
- [ ] Default `per_page` is **15** when omitted
- [ ] Default `page` is **1** when omitted
- [ ] Returns `items`, `pagination.total`, `pagination.per_page`, `pagination.current_page`, `pagination.last_page`
- [ ] `last_page` is `Math.ceil(total / per_page)` (minimum 1)
- [ ] Out-of-range `page` values return an empty `items` array with accurate pagination metadata

### AC-13 — Full-Collection Mode (`all: true`)
- [ ] When `body.all === true`, skips pagination and returns all matching rows
- [ ] Response shape is `{ status: number, data: { items: T[] } }` — no `pagination` key
- [ ] Filters and sorting still apply in full-collection mode

### AC-14 — Response Envelope
- [ ] All responses are wrapped in the standard envelope: `{ status: 200, data: { ... } }`
- [ ] On validation error (invalid body), throw an `H3Error` with status `422`

### AC-15 — Zod Schema (`shared/schemas/searchTable.schema.ts`)
- [ ] Schema validates the full `body` structure before the service processes it
- [ ] Each search item must have `key` (string), `type` (enum of 8 values), and `value` (any)
- [ ] `sort.direction` must be `'asc'` or `'desc'`
- [ ] `per_page` and `page` must be positive integers when provided
- [ ] `all` must be a boolean when provided
- [ ] All top-level fields are optional

### AC-16 — Consumer: Admin Board Table (`POST /api/admin/board/table`)
- [ ] Base query selects from `boards` with `leftJoin(boardUsers)`
- [ ] Passes validated body to `handleSearchTable`
- [ ] Protected by `auth` + `admin` server middleware

### AC-17 — Consumer: Admin User Table (`POST /api/admin/user/table`)
- [ ] Base query selects from `users`
- [ ] Protected by `auth` + `admin` server middleware

### AC-18 — Consumer: Admin Tag Table (`POST /api/admin/tag/table`)
- [ ] Base query selects from `tags`
- [ ] Protected by `auth` + `admin` server middleware

### AC-19 — Consumer: Admin Notification Table (`POST /api/admin/notification/table`)
- [ ] Base query selects from `notifications` ordered by `created_at DESC` by default
- [ ] Protected by `auth` + `admin` server middleware

### AC-20 — Consumer: Board Logs Tables
- [ ] `POST /api/admin/board/logs-table/[id]` — admin access to any board's logs
- [ ] `POST /api/client/board/logs-table/[id]` — client access limited to boards the user belongs to
- [ ] Both endpoints validate board membership / admin role before delegating to `handleSearchTable`

---

## Component Tree / File Structure

```
server/
└── services/
    └── searchTable.ts          # Core service — handleSearchTable()

server/api/
├── admin/
│   ├── board/
│   │   ├── table.post.ts       # AC-16
│   │   └── logs-table/
│   │       └── [id].post.ts    # AC-20 (admin)
│   ├── user/
│   │   └── table.post.ts       # AC-17
│   ├── tag/
│   │   └── table.post.ts       # AC-18
│   └── notification/
│       └── table.post.ts       # AC-19
└── client/
    └── board/
        └── logs-table/
            └── [id].post.ts    # AC-20 (client)

shared/
└── schemas/
    └── searchTable.schema.ts   # AC-15  (also referenced by validation-schemas spec)

shared/
└── types/
    └── api.ts                  # PaginatedResponse<T> / FullResponse<T> interfaces
```

---

## Data Model

### TypeScript Interfaces

```typescript
// server/services/searchTable.ts

type FilterType =
  | 'like'
  | 'equal'
  | 'date'
  | 'datetime-bool'
  | 'in'
  | 'relation-equal'
  | 'relation-like'
  | 'bool'

interface SearchFilter {
  key: string
  type: FilterType
  value: unknown
}

interface SortOptions {
  key: string
  direction: 'asc' | 'desc'
}

interface SearchTableBody {
  search?:   SearchFilter[]
  sort?:     SortOptions
  per_page?: number
  page?:     number
  all?:      boolean
}

interface SearchTableOptions {
  query: DrizzleSelectQuery   // any Drizzle .select().from(...) chain (with optional joins)
  body:  SearchTableBody
}
```

```typescript
// shared/types/api.ts

interface PaginationMeta {
  total:        number
  per_page:     number
  current_page: number
  last_page:    number
}

interface PaginatedResponse<T = unknown> {
  status: number
  data: {
    items:      T[]
    pagination: PaginationMeta
  }
}

interface FullResponse<T = unknown> {
  status: number
  data: {
    items: T[]
  }
}
```

```typescript
// shared/schemas/searchTable.schema.ts

import { z } from 'zod'

export const searchTableSchema = z.object({
  search: z.array(
    z.object({
      key:   z.string(),
      type:  z.enum(['like', 'equal', 'date', 'datetime-bool', 'in', 'relation-equal', 'relation-like', 'bool']),
      value: z.any(),
    }),
  ).optional(),
  sort: z.object({
    key:       z.string(),
    direction: z.enum(['asc', 'desc']),
  }).optional(),
  per_page: z.number().int().positive().optional(),
  page:     z.number().int().positive().optional(),
  all:      z.boolean().optional(),
})

export type SearchTableBody = z.infer<typeof searchTableSchema>
```

---

## Example Usage

```typescript
// server/api/admin/board/table.post.ts
// Spec: specs/search-table-service.md

import { handleSearchTable } from '~/server/services/searchTable'
import { searchTableSchema } from '~~/shared/schemas/searchTable.schema'

export default defineEventHandler(async (event) => {
  const body = searchTableSchema.parse(await readBody(event))

  const baseQuery = db
    .select()
    .from(boards)
    .leftJoin(boardUsers, eq(boards.id, boardUsers.boardId))

  return handleSearchTable({ query: baseQuery, body })
})
```

---

## Edge Cases

| Scenario | Expected Behaviour |
|---|---|
| `search` array is empty `[]` | No WHERE clauses added; all rows returned (subject to pagination) |
| `value` for `in` filter is `[]` | Returns `{ items: [], pagination: { total: 0, ... } }` |
| `page` exceeds `last_page` | Returns `{ items: [] }` with correct `total` and `last_page` |
| `per_page` is very large (e.g. 10 000) | Service applies it as-is; callers should enforce a max via their own Zod refinement if needed |
| `all: true` combined with `sort` | Sort is still applied; pagination is skipped |
| `all: true` combined with `search` | Filters are still applied; all matching rows returned |
| `sort.key` references a non-existent column | Drizzle will throw at query-build time; the error propagates as a 500 |
| `date` filter with `from > to` | Returns empty result set (valid SQL, zero rows match) |
| `datetime-bool` on a non-nullable column | Filter behaves as `IS NOT NULL` always (column is never null); callers must ensure the column is nullable |
| `relation-equal` / `relation-like` without prior join | Drizzle/Postgres will throw a column-not-found error at query time |
| Multiple filters of the same type on different columns | All are AND-composed into the WHERE clause |

---

## Non-Goals

- **Client-side pagination** — The service is server-side only; the frontend paginates by sending new requests.
- **Full-text search** (e.g. Postgres `tsvector`) — The `like` filter uses simple ILIKE, not FTS indices.
- **Cursor-based pagination** — Only offset/page-number pagination is supported.
- **Field selection / projection** — The caller's base query defines which columns are selected; the service does not add or remove columns.
- **Authorization / row-level security** — Each consuming endpoint is responsible for applying its own access checks before calling `handleSearchTable`.
- **Response caching** — No Redis caching is applied inside the service; endpoints may add caching independently.
- **Custom aggregations** — COUNT, SUM, AVG etc. are not part of this service.

---

## Dependencies

| Dependency | Role |
|---|---|
| `drizzle-orm` | Query builder — all SQL is constructed via Drizzle operators (`like`, `eq`, `inArray`, `isNull`, `isNotNull`, `gte`, `lte`, `asc`, `desc`) |
| `zod` | Request body validation via `searchTableSchema` (defined in `shared/schemas/searchTable.schema.ts`) |
| `shared/schemas/searchTable.schema.ts` | Shared Zod schema — also documented in `specs/validation-schemas.md` |
| `shared/types/api.ts` | `PaginatedResponse<T>` and `FullResponse<T>` response types |
| `server/database/index.ts` | Drizzle `db` client instance — not used inside the service itself, but required by every consumer |
| `server/middleware/auth.ts` | All admin consumers require an authenticated session |
| `server/middleware/admin.ts` | All `admin/` route consumers require the admin role |
