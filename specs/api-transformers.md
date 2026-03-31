# Spec: API Response Transformers

## Status: Approved

## Purpose

Define and implement a consistent set of transformer functions that convert raw Drizzle ORM query results into clean, typed API response shapes. These transformers act as the data-presentation layer (analogous to Laravel Resources), ensuring every API route returns a stable, well-typed contract regardless of internal schema changes.

A shared `apiResponse` / `apiError` utility wraps all responses in a standard envelope, providing uniform status codes and error payloads across the entire Nitro server.

---

## User Stories

- **As a backend developer**, I want each entity to have a dedicated transformer so I can reshape a raw DB row into a predictable response shape without scattering mapping logic across route handlers.
- **As a frontend developer**, I want every API response to follow the same `{ status, data }` envelope so I can write a single `useFetch` wrapper that handles success and error states uniformly.
- **As a backend developer**, I want card attachments split into `card_attachments` and `comment_attachments` so the client does not need to perform this filtering itself.
- **As a backend developer**, I want tag colors expanded to `{ name, value }` so the client has both the human-readable label and the integer enum value without an extra lookup.
- **As a backend developer**, I want board icons resolved to a `icon_name` string via `BoardIconNameMap` so API consumers receive a ready-to-render Material Symbol name instead of a raw integer.
- **As a backend developer**, I want notification `created_at` formatted as `"Y-m-d H:i:s"` so the client receives a consistent, locale-neutral timestamp string.

---

## Acceptance Criteria

### AC-0 — Standard Response Utilities (`server/utils/response.ts`)

- `apiResponse<T>(data: T, status = 200)` returns `{ status, data }`.
- `apiError(message, status = 400, errors?)` returns `{ status, message, errors }` where `errors` is an optional `Record<string, string[]>`.
- Both functions are auto-imported by Nitro (exported from `server/utils/`).
- No route handler constructs a raw response object — all success and error responses go through these utilities.

---

### AC-1 — `transformUser` (`server/transformers/user.transformer.ts`)

**Input:** a row from the `users` table, plus an optional `cardUser` pivot row from `card_user`.

**Output:**
```
{ id, name, username, avatar, role, telegram, is_disable, status? }
```

- `avatar` is `string | null`.
- `telegram` is `{ id, chat_id?, username? } | null`.
- `role` is the raw integer from the `users` row.
- `is_disable` is a boolean.
- `status` is only present when a `cardUser` pivot is supplied; its value is the `CardUserStatusEnum` integer from that pivot row.
- When no pivot is provided the `status` key must be omitted entirely (not `undefined`, not `null`).

---

### AC-2 — `transformBoard` (`server/transformers/board.transformer.ts`)

**Input:** a row from the `boards` table with eager-loaded `users` and `boardLists` relations.

**Output:**
```
{ id, name, description, icon, icon_name, status, users_count, members[], board_lists[] }
```

- `icon` is the raw integer from the `boards` row.
- `icon_name` is resolved by passing `icon` through `BoardIconNameMap`; the result is a Material Symbols string (e.g., `"dashboard"`). If the key is absent from the map the value falls back to an empty string `""`.
- `users_count` equals the length of the loaded `users` relation array.
- `members` is `users` mapped through `transformUser` (no card pivot context).
- `board_lists` is `boardLists` mapped through `transformBoardList`.

---

### AC-3 — `transformBoardList` (`server/transformers/boardList.transformer.ts`)

**Input:** a row from the `board_lists` table with an eager-loaded `cards` relation.

**Output:**
```
{ id, board_id, name, position, cards[] }
```

- `cards` is the `cards` relation mapped through `transformCard`.
- When the `cards` relation is absent or empty, `cards` is `[]`.

---

### AC-4 — `transformCard` (`server/transformers/card.transformer.ts`)

**Input:** a row from the `cards` table with eager-loaded `users` (with `card_user` pivot), `comments` (with their own `attachment`), and `attachments` relations.

**Output:**
```
{ id, boardlist_id, name, description, position, tags[], users[], comments[], card_attachments[], comment_attachments[] }
```

- `boardlist_id` is `number | null`.
- `description` is `string | null`.
- `tags` is the `tags` relation mapped through `transformTag`, with color fully expanded.
- `users` is the `users` relation mapped through `transformUser`, passing the `card_user` pivot so each user includes `status`.
- `comments` is mapped through `transformComment`.
- `card_attachments` — attachments from the `attachments` relation where the attachment is **not** referenced by any comment's `attachmentId`.
- `comment_attachments` — attachments from the `attachments` relation where the attachment **is** referenced by at least one comment's `attachmentId`.
- The split between `card_attachments` and `comment_attachments` is computed inside the transformer; no caller is expected to pre-filter.

---

### AC-5 — `transformComment` (`server/transformers/comment.transformer.ts`)

**Input:** a row from the `comments` table with eager-loaded `user` and `attachment` relations.

**Output:**
```
{ id, user_id, card_id, body, user_name, user_avatar, attachment }
```

- `user_name` is sourced from the related `user` row's `name` field.
- `user_avatar` is sourced from the related `user` row's `avatar` field (`string | null`).
- `attachment` is `transformAttachment(attachment)` when the relation is present, or `null` when absent.

---

### AC-6 — `transformAttachment` (`server/transformers/attachment.transformer.ts`)

**Input:** a row from the `attachments` table.

**Output:**
```
{ id, card_id, name, path, type }
```

- `type` is the raw integer enum value from the row.
- No relations are resolved; this is a flat mapping.

---

### AC-7 — `transformTag` (`server/transformers/tag.transformer.ts`)

**Input:** a row from the `tags` table.

**Output:**
```
{ id, name, description, color: { name, value } }
```

- `color.value` is the raw integer stored in the `tags` row (e.g., `0`).
- `color.name` is the string key of the corresponding enum entry (e.g., `"Red"`). This is resolved by reverse-mapping the integer against the color enum — not a hardcoded lookup.
- Both `color.name` and `color.value` must always be present.

---

### AC-8 — `transformCardLog` (`server/transformers/cardLog.transformer.ts`)

**Input:** a row from the `card_logs` table with eager-loaded `user` and `card` relations.

**Output:**
```
{ id, user_id, card_id, board_id, type, data, user, card: { id, name } }
```

- `type` is the raw integer from the row.
- `data` is typed as `{ message?: string; changes?: Record<string, unknown>; old_data?: Record<string, unknown> }`.
- `user` is `transformUser(user)` (no card pivot context).
- `card` includes only `{ id, name }` — not the full transformed card.

---

### AC-9 — `transformNotification` (`server/transformers/notification.transformer.ts`)

**Input:** a row from the `notifications` table.

**Output:**
```
{ id, type, data, read_at, created_at }
```

- `id` is a UUID string.
- `data` is the raw JSON string stored in the column (not parsed).
- `read_at` is `string | null`.
- `created_at` is formatted as `"Y-m-d H:i:s"` (e.g., `"2024-01-15 14:30:00"`) — the raw `Date` object from Drizzle must be converted to this format inside the transformer.

---

## Component Tree / File Structure

```
server/
├── transformers/
│   ├── user.transformer.ts          # AC-1
│   ├── board.transformer.ts         # AC-2
│   ├── boardList.transformer.ts     # AC-3
│   ├── card.transformer.ts          # AC-4
│   ├── comment.transformer.ts       # AC-5
│   ├── attachment.transformer.ts    # AC-6
│   ├── tag.transformer.ts           # AC-7
│   ├── cardLog.transformer.ts       # AC-8
│   └── notification.transformer.ts  # AC-9
└── utils/
    └── response.ts                  # AC-0
```

No client-side files are created as part of this spec.

---

## Data Model

### TypeScript Interfaces

```typescript
// ── Response envelope ────────────────────────────────────────────────────────

interface ApiResponse<T> {
  status: number
  data: T
}

interface ApiErrorResponse {
  status: number
  message: string
  errors?: Record<string, string[]>
}

// ── Transformer output shapes ─────────────────────────────────────────────────

interface TransformedUser {
  id: number
  name: string
  username: string
  avatar: string | null
  role: number
  telegram: { id: string; chat_id?: string; username?: string } | null
  is_disable: boolean
  status?: number  // present only in card-user pivot context
}

interface TransformedBoard {
  id: number
  name: string
  description: string
  icon: number
  icon_name: string
  status: number
  users_count: number
  members: TransformedUser[]
  board_lists: TransformedBoardList[]
}

interface TransformedBoardList {
  id: number
  board_id: number
  name: string
  position: number
  cards: TransformedCard[]
}

interface TransformedCard {
  id: number
  boardlist_id: number | null
  name: string
  description: string | null
  position: number
  tags: TransformedTag[]
  users: TransformedUser[]
  comments: TransformedComment[]
  card_attachments: TransformedAttachment[]
  comment_attachments: TransformedAttachment[]
}

interface TransformedComment {
  id: number
  user_id: number
  card_id: number
  body: string
  user_name: string
  user_avatar: string | null
  attachment: TransformedAttachment | null
}

interface TransformedAttachment {
  id: number
  card_id: number
  name: string
  path: string
  type: number
}

interface TransformedTag {
  id: number
  name: string
  description: string
  color: { name: string; value: number }
}

interface TransformedCardLog {
  id: number
  user_id: number
  card_id: number
  board_id: number
  type: number
  data: {
    message?: string
    changes?: Record<string, unknown>
    old_data?: Record<string, unknown>
  }
  user: TransformedUser
  card: { id: number; name: string }
}

interface TransformedNotification {
  id: string           // UUID
  type: string
  data: string         // raw JSON string, not parsed
  read_at: string | null
  created_at: string   // formatted "Y-m-d H:i:s"
}
```

---

## Edge Cases

| Transformer | Edge Case | Expected Behaviour |
|---|---|---|
| `transformUser` | Called without a card pivot | `status` key absent from output object |
| `transformBoard` | `icon` value not found in `BoardIconNameMap` | `icon_name` falls back to `""` |
| `transformBoard` | `users` / `boardLists` relations not loaded | `members` and `board_lists` default to `[]` |
| `transformBoardList` | `cards` relation not loaded | `cards` defaults to `[]` |
| `transformCard` | No attachments on the card | Both `card_attachments` and `comment_attachments` are `[]` |
| `transformCard` | All attachments are linked to comments | `card_attachments` is `[]`; `comment_attachments` contains all |
| `transformCard` | No comments on the card | `comments` is `[]`; attachment split still executes safely |
| `transformComment` | No attachment linked to comment | `attachment` is `null` |
| `transformTag` | Color integer has no reverse-map match | Throw a typed error — a missing color is a data integrity violation, not a graceful fallback |
| `transformCardLog` | `data` column holds an empty object | `data` is `{}` — all optional keys absent |
| `transformNotification` | `read_at` is `null` | `read_at` is `null` in output |
| `transformNotification` | `created_at` is a `Date` object | Must be formatted to `"Y-m-d H:i:s"` string before returning |

---

## Non-Goals

- Transformers do **not** perform database queries; they only reshape data already loaded by the route handler.
- Transformers do **not** apply authorization filtering (e.g., hiding fields based on role); that is handled by route-level middleware.
- Transformers do **not** validate input shapes with Zod; Zod validation is applied to **request** bodies, not DB rows.
- This spec does not cover pagination wrappers — those are a separate utility concern.
- No client-side types or composables are introduced by this spec.

---

## Dependencies

| Dependency | Purpose |
|---|---|
| `shared/enums/BoardIconNameMap` | Maps `icon` integer → Material Symbol string name in `transformBoard` |
| `shared/enums/CardUserStatusEnum` | Provides the pivot status integer type used in `transformUser` |
| Color enum (name TBD in schema spec) | Reverse-mapped in `transformTag` to produce `color.name` |
| `server/database/schema/` | Drizzle row types used as transformer inputs (inferred from schema) |
| `specs/database-schema.md` | Source of truth for all table shapes and relation definitions |
| `specs/validation-schemas.md` | Confirms field types align with Zod schemas used in route handlers |
