# Tasks: API Response Transformers
**Spec:** `specs/api-transformers.md`
**Status:** pending

## Overview

Implement the complete data-presentation layer for the Nitro server: a standard response envelope utility and nine entity transformer functions that convert raw Drizzle ORM query results into clean, typed API response shapes. All work is server-side only — this spec introduces no client-side files. Wave 2 is skipped.

Execution order within Wave 1 is sequential: leaf transformers first (user, attachment, tag), then composed transformers that call them (comment → card + boardList → board), then standalone transformers with no intra-spec dependencies (cardLog, notification). Response utilities are a prerequisite for nothing in this spec but must be completed first as they underpin every route handler in the project.

---

## Wave 1 — Data Layer (API Agent)

### Task 005-A: Standard Response Utilities
- **Agent:** API
- **Spec:** `specs/api-transformers.md`
- **Section:** AC-0 — Standard Response Utilities
- **Acceptance Criteria:**
  - [ ] AC-0a: `apiResponse<T>(data: T, status = 200)` returns `{ status, data }` typed as `ApiResponse<T>`
  - [ ] AC-0b: `apiError(message, status = 400, errors?)` returns `{ status, message, errors }` where `errors` is `Record<string, string[]> | undefined`
  - [ ] AC-0c: Both functions are exported from `server/utils/response.ts` so Nitro auto-imports them
  - [ ] AC-0d: No route handler constructs a raw response object — all success and error responses go through these utilities (enforced by convention; document in JSDoc)
- **Dependencies:** `tasks/001-infrastructure-setup.md`
- **Output Files:**
  - `server/utils/response.ts`

---

### Task 005-B: Leaf Transformers — User, Attachment, Tag
- **Agent:** API
- **Spec:** `specs/api-transformers.md`
- **Section:** AC-1 (`transformUser`), AC-6 (`transformAttachment`), AC-7 (`transformTag`)
- **Acceptance Criteria:**
  - [ ] AC-1a: `transformUser(row, cardUserPivot?)` returns `{ id, name, username, avatar, role, telegram, is_disable }` when no pivot is supplied
  - [ ] AC-1b: When a `cardUser` pivot row is provided, the output additionally includes `status` (the `CardUserStatusEnum` integer from the pivot)
  - [ ] AC-1c: When no pivot is provided, the `status` key is **omitted entirely** — not `undefined`, not `null`
  - [ ] AC-1d: `avatar` is `string | null`; `telegram` is `{ id, chat_id?, username? } | null`; `is_disable` is boolean
  - [ ] AC-6a: `transformAttachment(row)` returns `{ id, card_id, name, path, type }` — flat mapping, no relations resolved
  - [ ] AC-6b: `type` is the raw integer enum value from the row
  - [ ] AC-7a: `transformTag(row)` returns `{ id, name, description, color: { name, value } }`
  - [ ] AC-7b: `color.value` is the raw integer stored in the `tags` row
  - [ ] AC-7c: `color.name` is the string key of the corresponding `TagColorEnum` entry, resolved by reverse-mapping the integer — not a hardcoded lookup
  - [ ] AC-7d: If the integer has no reverse-map match, a typed error is thrown — this is a data integrity violation, not a graceful fallback
- **Dependencies:** `tasks/005-api-transformers.md` Task 005-A, `tasks/002-database-schema.md`
- **Output Files:**
  - `server/transformers/user.transformer.ts`
  - `server/transformers/attachment.transformer.ts`
  - `server/transformers/tag.transformer.ts`

---

### Task 005-C: Comment Transformer
- **Agent:** API
- **Spec:** `specs/api-transformers.md`
- **Section:** AC-5 (`transformComment`)
- **Acceptance Criteria:**
  - [ ] AC-5a: `transformComment(row)` returns `{ id, user_id, card_id, body, user_name, user_avatar, attachment }`
  - [ ] AC-5b: `user_name` is sourced from the eager-loaded `user` relation's `name` field
  - [ ] AC-5c: `user_avatar` is sourced from the eager-loaded `user` relation's `avatar` field (`string | null`)
  - [ ] AC-5d: `attachment` is `transformAttachment(row.attachment)` when the relation is present, or `null` when absent
- **Dependencies:** Task 005-B
- **Output Files:**
  - `server/transformers/comment.transformer.ts`

---

### Task 005-D: Card and BoardList Transformers
- **Agent:** API
- **Spec:** `specs/api-transformers.md`
- **Section:** AC-4 (`transformCard`), AC-3 (`transformBoardList`)
- **Acceptance Criteria:**
  - [ ] AC-4a: `transformCard(row)` returns `{ id, boardlist_id, name, description, position, tags[], users[], comments[], card_attachments[], comment_attachments[] }`
  - [ ] AC-4b: `boardlist_id` is `number | null`; `description` is `string | null`
  - [ ] AC-4c: `tags` is the `tags` relation mapped through `transformTag`
  - [ ] AC-4d: `users` is the `users` relation mapped through `transformUser`, passing each user's `card_user` pivot row so every user in the output includes `status`
  - [ ] AC-4e: `comments` is the `comments` relation mapped through `transformComment`
  - [ ] AC-4f: `card_attachments` — attachments from the `attachments` relation where the attachment's `id` is **not** referenced by any comment's `attachmentId`
  - [ ] AC-4g: `comment_attachments` — attachments from the `attachments` relation where the attachment's `id` **is** referenced by at least one comment's `attachmentId`
  - [ ] AC-4h: The card_attachments / comment_attachments split is computed entirely inside the transformer; callers do not pre-filter
  - [ ] AC-4i: When `attachments` is absent or empty, both `card_attachments` and `comment_attachments` are `[]`
  - [ ] AC-4j: When all attachments are linked to comments, `card_attachments` is `[]`
  - [ ] AC-3a: `transformBoardList(row)` returns `{ id, board_id, name, position, cards[] }`
  - [ ] AC-3b: `cards` is the `cards` relation mapped through `transformCard`
  - [ ] AC-3c: When the `cards` relation is absent or empty, `cards` defaults to `[]`
- **Dependencies:** Task 005-B, Task 005-C
- **Output Files:**
  - `server/transformers/card.transformer.ts`
  - `server/transformers/boardList.transformer.ts`

---

### Task 005-E: Board Transformer
- **Agent:** API
- **Spec:** `specs/api-transformers.md`
- **Section:** AC-2 (`transformBoard`)
- **Acceptance Criteria:**
  - [ ] AC-2a: `transformBoard(row)` returns `{ id, name, description, icon, icon_name, status, users_count, members[], board_lists[] }`
  - [ ] AC-2b: `icon` is the raw integer from the `boards` row
  - [ ] AC-2c: `icon_name` is resolved by passing `icon` through `BoardIconNameMap`; falls back to `""` if the key is absent
  - [ ] AC-2d: `users_count` equals the length of the loaded `users` relation array
  - [ ] AC-2e: `members` is the `users` relation mapped through `transformUser` (no card pivot context)
  - [ ] AC-2f: `board_lists` is the `boardLists` relation mapped through `transformBoardList`
  - [ ] AC-2g: When `users` or `boardLists` relations are not loaded, `members` and `board_lists` default to `[]`
- **Dependencies:** Task 005-B, Task 005-D
- **Output Files:**
  - `server/transformers/board.transformer.ts`

---

### Task 005-F: Standalone Transformers — CardLog and Notification
- **Agent:** API
- **Spec:** `specs/api-transformers.md`
- **Section:** AC-8 (`transformCardLog`), AC-9 (`transformNotification`)
- **Acceptance Criteria:**
  - [ ] AC-8a: `transformCardLog(row)` returns `{ id, user_id, card_id, board_id, type, data, user, card: { id, name } }`
  - [ ] AC-8b: `type` is the raw integer from the row
  - [ ] AC-8c: `data` is typed as `{ message?: string; changes?: Record<string, unknown>; old_data?: Record<string, unknown> }`; when the column holds an empty object, `data` is `{}`
  - [ ] AC-8d: `user` is `transformUser(row.user)` with no card pivot context
  - [ ] AC-8e: `card` includes only `{ id, name }` — not the full `TransformedCard` shape
  - [ ] AC-9a: `transformNotification(row)` returns `{ id, type, data, read_at, created_at }`
  - [ ] AC-9b: `id` is a UUID string
  - [ ] AC-9c: `data` is the raw JSON string stored in the column — **not** parsed
  - [ ] AC-9d: `read_at` is `string | null`; when the DB value is `null`, the output is `null`
  - [ ] AC-9e: `created_at` is formatted as `"Y-m-d H:i:s"` (e.g., `"2024-01-15 14:30:00"`) — the raw `Date` object from Drizzle is converted to this format inside the transformer
- **Dependencies:** Task 005-B
- **Output Files:**
  - `server/transformers/cardLog.transformer.ts`
  - `server/transformers/notification.transformer.ts`

---

## Wave 2 — Frontend (Frontend Agent)

_Skipped. This spec introduces no client-side files (`shared/`, `app/`, or `tests/e2e/`). Transformers are server-only pure functions._

---

## Wave 3 — Tests (Testing Agent)

### Task 005-G: Response Utility and Transformer Unit Tests
- **Agent:** Testing
- **Spec:** `specs/api-transformers.md`
- **Section:** All ACs; Edge Cases table
- **Acceptance Criteria:**
  - [ ] `apiResponse` returns correct `{ status, data }` shape with default and explicit status codes
  - [ ] `apiError` returns correct `{ status, message, errors }` shape; `errors` is omitted when not provided
  - [ ] `transformUser` without pivot: output has no `status` key (strict key absence check, not `undefined` check)
  - [ ] `transformUser` with pivot: output includes `status` with the correct integer
  - [ ] `transformBoard` with `icon` absent from `BoardIconNameMap`: `icon_name` is `""`
  - [ ] `transformBoard` with missing `users` / `boardLists` relations: `members` and `board_lists` are `[]`
  - [ ] `transformBoardList` with missing `cards` relation: `cards` is `[]`
  - [ ] `transformCard` with no attachments: both `card_attachments` and `comment_attachments` are `[]`
  - [ ] `transformCard` with all attachments linked to comments: `card_attachments` is `[]`, `comment_attachments` contains all attachments
  - [ ] `transformCard` with no comments: `comments` is `[]`; attachment split executes safely
  - [ ] `transformComment` with no attachment: `attachment` is `null`
  - [ ] `transformTag` with a valid color integer: `color.name` and `color.value` are both present
  - [ ] `transformTag` with an unmapped color integer: throws a typed error
  - [ ] `transformCardLog` with an empty `data` object: output `data` is `{}`
  - [ ] `transformNotification` with `read_at` null: output `read_at` is `null`
  - [ ] `transformNotification` with a `Date` object for `created_at`: output is `"Y-m-d H:i:s"` formatted string
- **Dependencies:** Task 005-A, Task 005-B, Task 005-C, Task 005-D, Task 005-E, Task 005-F
- **Output Files:**
  - `tests/unit/server/utils/response.test.ts`
  - `tests/unit/server/transformers/user.transformer.test.ts`
  - `tests/unit/server/transformers/attachment.transformer.test.ts`
  - `tests/unit/server/transformers/tag.transformer.test.ts`
  - `tests/unit/server/transformers/comment.transformer.test.ts`
  - `tests/unit/server/transformers/card.transformer.test.ts`
  - `tests/unit/server/transformers/boardList.transformer.test.ts`
  - `tests/unit/server/transformers/board.transformer.test.ts`
  - `tests/unit/server/transformers/cardLog.transformer.test.ts`
  - `tests/unit/server/transformers/notification.transformer.test.ts`
