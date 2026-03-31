# Spec: Admin Tag Management
## Status: Approved

---

## Purpose

Provide administrators with full CRUD control over tags used throughout the task management system. Tags are globally defined (name, color, description) and attached to cards as JSONB arrays of IDs. This spec covers all 6 admin-facing tag endpoints, the client read-only tag endpoint, the tag transformer, and the frontend tag management page with modal workflows.

---

## User Stories

- **As an admin**, I want to create tags with a name, one of 9 preset colors, and an optional description so cards can be labeled consistently.
- **As an admin**, I want to view all tags in a paginated, searchable table so I can manage a large tag library efficiently.
- **As an admin**, I want to edit a tag's name, color, or description so I can correct mistakes or rebrand labels.
- **As an admin**, I want to delete a tag so obsolete labels no longer appear as options.
- **As an admin**, I want to view the details of a single tag so I can inspect its current configuration.
- **As any authenticated user**, I want to read all available tags so I can assign them to cards.

---

## Acceptance Criteria

### AC-1: List All Tags (`GET /api/admin/tag/all`)
- Returns an array of all tags transformed via `transformTag`.
- Each item includes `id`, `name`, `description`, and `color: { name: string, value: number }`.
- Color is expanded — e.g., `{ name: "Red", value: 0 }` not a raw integer.
- Requires `AdminUser` role; unauthenticated or non-admin requests receive `401`/`403`.

### AC-2: Paginated Tag Table (`POST /api/admin/tag/table`)
- Accepts a `SearchTableRequest` body (filters, sort, pagination).
- Returns a paginated result set via the `SearchTable` service.
- Supports text search on `name` and `description` fields.
- Sort columns include at minimum: `id`, `name`, `createdAt`.
- Requires `AdminUser` role.

### AC-3: Create Tag (`POST /api/admin/tag/create`)
- Validates request body against `createTagSchema`: `name` (required, 1–50 chars), `color` (required, integer 0–8), `description` (optional, max 255 chars).
- Rejects duplicate `name` with `422` and a message indicating the name is already taken.
- Stores `color` as a `smallint` (`TagColorEnum` value).
- `description` defaults to `"default"` when omitted.
- Returns the created tag via `transformTag`.
- Requires `AdminUser` role.

### AC-4: Update Tag (`PUT /api/admin/tag/[id]`)
- Validates request body against `updateTagSchema` (all fields optional/partial).
- Returns `404` when the tag `id` does not exist.
- `name` uniqueness constraint must still hold on update; duplicate name returns `422`.
- Returns the updated tag via `transformTag`.
- Requires `AdminUser` role.

### AC-5: Delete Tag (`DELETE /api/admin/tag/[id]`)
- Returns `404` when the tag `id` does not exist.
- Deletes the tag row from the `tags` table.
- **Does not** cascade-clean `cards.tags` JSONB arrays — stale IDs remain until `syncCardTags` runs.
- Returns a success acknowledgement (e.g., `{ success: true }`).
- Requires `AdminUser` role.

### AC-6: Get Tag Details (`GET /api/admin/tag/[id]`)
- Returns a single tag transformed via `transformTag`.
- Returns `404` when the tag `id` does not exist.
- Requires `AdminUser` role.

### AC-7: Client Read-Only Tag List (`GET /api/client/tag/all`)
- Returns an array of all tags via `transformTag` (same shape as AC-1).
- Requires only `auth` + `active-user` middleware — **no admin role required**.
- Any authenticated, active user can read tags for card-assignment purposes.

### AC-8: Tag Transformer (`transformTag`)
- Accepts a raw `tags` DB row.
- Returns `{ id, name, description, color: { name: string, value: number } }`.
- `color.name` is the `TagColorEnum` key (e.g., `"Violet"`) derived from the stored integer value.
- `color.value` is the raw `smallint` stored in the DB.
- All 9 color entries in `TagColorEnum` must map correctly.

### AC-9: Tag Color Picker (Frontend)
- Color picker must offer all 9 `TagColorEnum` options: Red, Purple, Green, Orange, Blue, Gray, Cyan, Magenta, Violet.
- Selected color is submitted as its integer value (`TagColorEnum` value).
- Each color option renders with its actual color swatch for visual identification.

### AC-10: Tag Chip Component (`app/components/ui/Tag.vue`)
- Accepts a `TagModel` prop.
- Renders the tag `name` with a background/border color derived from `color.name` (or `color.value`).
- Color rendering covers all 9 `TagColorEnum` variants with distinct, accessible styles.

### AC-11: Admin Tag Page (`app/pages/tag/index.vue`)
- Uses the `default` layout.
- Displays a `Table/Search` component backed by `POST /api/admin/tag/table`.
- Provides an **Add Tag** button that opens the Add Tag modal.
- Each table row has **Edit** and **Delete** action buttons.
- **Add Tag modal**: form with name input, color picker, description textarea; submits to `POST /api/admin/tag/create`.
- **Edit Tag modal**: pre-populated form; submits to `PUT /api/admin/tag/[id]`.
- **Delete confirmation modal**: shows tag name; submits to `DELETE /api/admin/tag/[id]`; warns that card references are not automatically removed.
- On successful create/update/delete the table refreshes.
- Validation errors from the API (e.g., duplicate name) are surfaced inline in the modal.

### AC-12: Middleware Chain (All Admin Routes)
- Every `server/api/admin/tag/*` handler is protected by, in order:
  1. `auth` middleware — validates JWT, attaches user to event context.
  2. `active-user` middleware — rejects disabled/suspended accounts.
  3. `admin` middleware — requires `role === UserRoleEnum.AdminUser`.
- Missing or invalid JWT → `401`.
- Disabled account → `403`.
- Non-admin role → `403`.

---

## Component Tree / File Structure

```
server/
└── api/
    ├── admin/
    │   └── tag/
    │       ├── all.get.ts          # AC-1: list all tags
    │       ├── table.post.ts       # AC-2: paginated table
    │       ├── create.post.ts      # AC-3: create tag
    │       ├── [id].put.ts         # AC-4: update tag
    │       ├── [id].delete.ts      # AC-5: delete tag
    │       └── [id].get.ts         # AC-6: get tag details
    └── client/
        └── tag/
            └── all.get.ts          # AC-7: client read-only list

app/
├── pages/
│   └── tag/
│       └── index.vue               # Admin tag management page (AC-11)
└── components/
    └── ui/
        └── Tag.vue                 # Tag chip display component (AC-10)

shared/
└── schemas/
    └── tag.schema.ts               # createTagSchema, updateTagSchema (Zod)

server/
└── transformers/
    └── tag.transformer.ts          # transformTag function (AC-8)
```

---

## Data Model

### Database Table — `tags`

| Column        | Type                          | Constraints                        |
|---------------|-------------------------------|------------------------------------|
| `id`          | `serial`                      | Primary key                        |
| `name`        | `varchar(50)`                 | Not null, unique                   |
| `color`       | `smallint`                    | Not null — stores `TagColorEnum` value |
| `description` | `varchar(255)`                | Not null, default `'default'`      |
| `createdAt`   | `timestamp with time zone`    | Not null, default `now()`          |
| `updatedAt`   | `timestamp with time zone`    | Not null, default `now()`          |

### TagColorEnum

| Key       | Value |
|-----------|-------|
| `Red`     | `0`   |
| `Purple`  | `1`   |
| `Green`   | `2`   |
| `Orange`  | `3`   |
| `Blue`    | `4`   |
| `Gray`    | `5`   |
| `Cyan`    | `6`   |
| `Magenta` | `7`   |
| `Violet`  | `8`   |

### Tags on Cards (Relationship)

Tags are **not** stored via a junction table. Instead, `cards.tags` is a JSONB column holding an array of tag `id` integers:

```
cards.tags = [1, 4, 7]   // array of tag IDs
```

- Tags are assigned/removed from cards via `POST /api/client/card/tags/[id]`.
- Deleting a tag does **not** retroactively clean stale IDs from `cards.tags` — a `syncCardTags` utility must be invoked separately to remove orphaned references.
- The admin and client tag endpoints operate on the `tags` table only; card JSONB cleanup is out of scope for this spec.

---

### API Endpoints Consumed

| Method | Endpoint | Request | Response |
|--------|----------|---------|----------|
| `GET` | `/api/admin/tag/all` | — | `TagModel[]` |
| `POST` | `/api/admin/tag/table` | `SearchTableRequest` | `SearchTableResponse<TagModel>` |
| `POST` | `/api/admin/tag/create` | `CreateTagRequest` | `TagModel` |
| `PUT` | `/api/admin/tag/[id]` | `UpdateTagRequest` | `TagModel` |
| `DELETE` | `/api/admin/tag/[id]` | — | `{ success: true }` |
| `GET` | `/api/admin/tag/[id]` | — | `TagModel` |
| `GET` | `/api/client/tag/all` | — | `TagModel[]` |

---

### TypeScript Interfaces

```typescript
// Transformer output (used everywhere tags are consumed)
interface TagModel {
  id: number
  name: string
  description: string
  color: {
    name: string   // TagColorEnum key, e.g. "Red"
    value: number  // TagColorEnum integer, e.g. 0
  }
}

// POST /api/admin/tag/create request body
interface CreateTagRequest {
  name: string          // required, 1–50 chars
  color: number         // required, TagColorEnum value 0–8
  description?: string  // optional, max 255 chars; defaults to "default"
}

// PUT /api/admin/tag/[id] request body
interface UpdateTagRequest {
  name?: string
  color?: number
  description?: string
}

// Shared Zod schemas (shared/schemas/tag.schema.ts)
// createTagSchema  → validates CreateTagRequest
// updateTagSchema  → createTagSchema.partial()
```

---

## State Management

- No dedicated Pinia store required for tag management — the admin page manages local state via `useAsyncData` / `$fetch`.
- The `Table/Search` component handles its own pagination and filter state internally.
- Modals use local `ref` state (open/close, form values, loading, error).
- After any mutation (create/update/delete), the table data is refreshed by re-triggering `POST /api/admin/tag/table`.
- Client-side tag lists (for card assignment) may be cached in a shared composable or lightweight store that other features (card detail) can consume; design of that composable is deferred to the card management spec.

---

## Edge Cases

| Scenario | Expected Behaviour |
|----------|--------------------|
| Create tag with duplicate `name` | `422` response; modal shows inline error "Tag name already exists" |
| Update tag `name` to one used by another tag | `422` same inline error |
| Delete tag that is referenced by 1+ cards | Delete succeeds; stale IDs remain in `cards.tags` until `syncCardTags` runs; UI warns user |
| `GET /api/admin/tag/[id]` with non-existent ID | `404` |
| `PUT /api/admin/tag/[id]` with empty body | No-op update; tag returned unchanged (all fields optional) |
| `color` value outside 0–8 | `422` validation error from `createTagSchema` / `updateTagSchema` |
| `name` exceeds 50 chars | `422` validation error |
| `description` exceeds 255 chars | `422` validation error |
| Non-admin authenticated user hits admin route | `403` |
| Unauthenticated request | `401` |
| Disabled user hits any tag endpoint | `403` from `active-user` middleware |

---

## Non-Goals

- Automatic cleanup of `cards.tags` JSONB arrays when a tag is deleted (handled by `syncCardTags`, out of scope).
- Tag reordering or prioritization.
- Per-board tag scoping — tags are global.
- Bulk create/delete operations.
- Tag usage analytics (how many cards reference a tag).
- Custom hex color input — only the 9 `TagColorEnum` presets are supported.

---

## Dependencies

| Dependency | Role |
|------------|------|
| `shared/schemas/tag.schema.ts` | Zod validation for create and update requests |
| `shared/enums/TagColorEnum.ts` | 9 color constants used by server and client |
| `server/transformers/tag.transformer.ts` | `transformTag()` — normalizes DB row to `TagModel` |
| `server/middleware/auth.ts` | JWT validation on all tag routes |
| `server/middleware/active-user.ts` | Blocks disabled users |
| `server/middleware/admin.ts` | Enforces `AdminUser` role on admin routes |
| `SearchTable` service | Powers paginated `table.post.ts` endpoint (see `search-table-service.md`) |
| `app/components/ui/Tag.vue` | Tag chip rendering on the admin page and card views |
| `cards.tags` (JSONB) | Read-only relationship context; card tag sync is a separate concern |
| `POST /api/client/card/tags/[id]` | Card-side tag assignment; referenced for context only |
