# Tasks: Zod Validation Schemas
**Spec:** `specs/validation-schemas.md`
**Status:** pending

---

## Overview

Implement the complete shared Zod validation schema layer in `shared/schemas/`. Nine schema files cover all API request bodies across auth, board, boardList, card, comment, attachment, tag, user, and searchTable domains. Each file exports named schema constants plus inferred TypeScript types. A server-side ZodError handler transforms `ZodError` exceptions into standardised HTTP 422 responses. All schemas are consumed by Nitro route handlers (server-side) and optionally by client-side form composables via the Nuxt 4 `shared/` auto-import boundary.

---

## Wave 1 — Shared Schemas & Error Handler (API Agent)

### Task 003-A: Auth & Centrifugo Channel Schemas

- **Agent:** API
- **Spec:** `specs/validation-schemas.md`
- **Section:** `auth.schema.ts` ACs; General ACs; Data Model (inferred types for `loginSchema`, `centrifugoChannelSchema`); Usage in Route Handlers
- **Acceptance Criteria:**
  - [ ] All schema files live under `shared/schemas/` and are importable from both server (`~~/shared/schemas/`) and client code.
  - [ ] Every schema file exports named schema constants (no default exports).
  - [ ] All schemas use Zod (`z`) — no other validation library is used.
  - [ ] `loginSchema` — `username`: required string, minimum 1 character.
  - [ ] `loginSchema` — `password`: required string, minimum 8 characters.
  - [ ] `centrifugoChannelSchema` — `channel`: required string, minimum 1 character.
  - [ ] `LoginInput` and `CentrifugoChannelInput` types are exported as `z.infer<>` of their respective schemas.
  - [ ] Server route handlers import schemas from `~~/shared/schemas/<entity>.schema` and pass `readBody(event)` output to `schema.parse(body)`.
  - [ ] Validated data (return value of `.parse()`) is used exclusively downstream — raw `body` is never used after validation.
- **Dependencies:** `tasks/002-database-schema.md`
- **Output Files:**
  - `shared/schemas/auth.schema.ts`

---

### Task 003-B: Board & BoardList Schemas

- **Agent:** API
- **Spec:** `specs/validation-schemas.md`
- **Section:** `board.schema.ts` ACs; `boardList.schema.ts` ACs; Enum Coercion Behaviour (`status`, `icon`); Data Model (inferred types); Edge Cases (`.partial()` fields, position integers)
- **Acceptance Criteria:**
  - [ ] `createBoardSchema` — `name`: required string, min 1 char, max 50 chars.
  - [ ] `createBoardSchema` — `description`: required string, minimum 1 character.
  - [ ] `createBoardSchema` — `status`: optional `z.coerce.number().int()`, range 0–1 inclusive, maps to `BoardStatusEnum`.
  - [ ] `createBoardSchema` — `icon`: optional coerced integer, range 0–14 inclusive, maps to `BoardIconEnum`.
  - [ ] `updateBoardSchema` — all fields from `createBoardSchema` made optional via `.partial()`.
  - [ ] `syncBoardMembersSchema` — `user_ids`: required array of integers (`z.number().int()`).
  - [ ] `createBoardListSchema` — `name`: required string, min 1 char, max 50 chars.
  - [ ] `createBoardListSchema` — `position`: required integer (`z.number().int()`).
  - [ ] `updateBoardListSchema` — all fields from `createBoardListSchema` made optional via `.partial()`.
  - [ ] `updatePositionSchema` — `items`: required array of objects, each with `id` (integer) and `position` (integer).
  - [ ] `z.coerce.number().int()` fields accept both `number` and numeric `string` inputs (e.g., `"1"` → `1`).
  - [ ] `z.coerce.number()` conversion of a non-numeric string (e.g., `"abc"`) yields `NaN`, which fails `.int()` and returns a 422, not a 500.
  - [ ] `.partial()` schemas still enforce original field constraints when a field is supplied (e.g., `name: ""` still fails `.min(1)`).
  - [ ] `position` fields use `z.number().int()` and reject float values.
  - [ ] Inferred types exported: `CreateBoardInput`, `UpdateBoardInput`, `SyncBoardMembersInput`, `CreateBoardListInput`, `UpdateBoardListInput`, `UpdatePositionInput`.
- **Dependencies:** `tasks/002-database-schema.md`
- **Output Files:**
  - `shared/schemas/board.schema.ts`
  - `shared/schemas/boardList.schema.ts`

---

### Task 003-C: Card Schemas

- **Agent:** API
- **Spec:** `specs/validation-schemas.md`
- **Section:** `card.schema.ts` ACs; Enum Coercion Behaviour (`status` → `CardUserStatusEnum`); Data Model (inferred types); Edge Cases (position integers, array sizes)
- **Acceptance Criteria:**
  - [ ] `createCardSchema` — `name`: required string, min 1 char, max 255 chars.
  - [ ] `createCardSchema` — `description`: optional string.
  - [ ] `createCardSchema` — `position`: required integer.
  - [ ] `createCardSchema` — `user_ids`: optional array of integers.
  - [ ] `createCardSchema` — `tag_ids`: optional array of integers.
  - [ ] `quickCreateCardSchema` — `name`: required string, min 1 char, max 255 chars; no other fields accepted.
  - [ ] `updateCardSchema` — `name`: optional string, min 1 char, max 255 chars.
  - [ ] `updateCardSchema` — `description`: optional string.
  - [ ] `updateCardSchema` — `board_list_id`: optional integer.
  - [ ] `updateCardSchema` — `position`: optional integer.
  - [ ] `syncCardMembersSchema` — `user_ids`: required array of integers.
  - [ ] `syncCardTagsSchema` — `tag_ids`: required array of integers.
  - [ ] `updateCardStatusSchema` — `user_id`: required integer.
  - [ ] `updateCardStatusSchema` — `status`: required coerced integer, range 0–4 inclusive, maps to `CardUserStatusEnum`.
  - [ ] `updateCardPositionSchema` — `board_list_id`: required integer.
  - [ ] `updateCardPositionSchema` — `items`: required array of objects, each with `id` (integer) and `position` (integer).
  - [ ] `z.coerce.number().int()` on `status` accepts numeric string inputs; rejects non-numeric strings with 422.
  - [ ] Inferred types exported: `CreateCardInput`, `QuickCreateCardInput`, `UpdateCardInput`, `SyncCardMembersInput`, `SyncCardTagsInput`, `UpdateCardStatusInput`, `UpdateCardPositionInput`.
- **Dependencies:** `tasks/002-database-schema.md`
- **Output Files:**
  - `shared/schemas/card.schema.ts`

---

### Task 003-D: Comment & Attachment Schemas

- **Agent:** API
- **Spec:** `specs/validation-schemas.md`
- **Section:** `comment.schema.ts` ACs; `attachment.schema.ts` ACs; Enum Coercion Behaviour (`type`); Data Model (inferred types)
- **Acceptance Criteria:**
  - [ ] `createCommentSchema` — `body`: required string, minimum 1 character.
  - [ ] `createCommentSchema` — `attachment_id`: optional integer.
  - [ ] `updateCommentSchema` — `body`: optional string, minimum 1 character.
  - [ ] `updateCommentSchema` — `pin`: optional boolean.
  - [ ] `createAttachmentSchema` — `name`: required string, min 1 char, max 50 chars.
  - [ ] `createAttachmentSchema` — `type`: required coerced integer, range 0–5 inclusive.
  - [ ] `updateAttachmentSchema` — all fields from `createAttachmentSchema` made optional via `.partial()`.
  - [ ] `.optional()` fields accept `undefined` but not `null` — no `.nullable()` used unless explicitly required.
  - [ ] Inferred types exported: `CreateCommentInput`, `UpdateCommentInput`, `CreateAttachmentInput`, `UpdateAttachmentInput`.
- **Dependencies:** `tasks/002-database-schema.md`
- **Output Files:**
  - `shared/schemas/comment.schema.ts`
  - `shared/schemas/attachment.schema.ts`

---

### Task 003-E: Tag & User Schemas

- **Agent:** API
- **Spec:** `specs/validation-schemas.md`
- **Section:** `tag.schema.ts` ACs; `user.schema.ts` ACs; Enum Coercion Behaviour (`color` → `TagColorEnum`, `role` → `UserRoleEnum`, `is_disable`); Data Model (inferred types); Edge Cases (`updatePasswordSchema` refinement order)
- **Acceptance Criteria:**
  - [ ] `createTagSchema` — `name`: required string, min 1 char, max 50 chars.
  - [ ] `createTagSchema` — `color`: required coerced integer, range 0–8 inclusive, maps to `TagColorEnum`.
  - [ ] `createTagSchema` — `description`: optional string, maximum 255 characters.
  - [ ] `updateTagSchema` — all fields from `createTagSchema` made optional via `.partial()`.
  - [ ] `createUserSchema` — `name`: required string, min 1 char, max 50 chars.
  - [ ] `createUserSchema` — `username`: required string, min 1 char, max 50 chars.
  - [ ] `createUserSchema` — `password`: required string, minimum 8 characters.
  - [ ] `createUserSchema` — `role`: optional coerced integer, range 0–1 inclusive, maps to `UserRoleEnum`.
  - [ ] `createUserSchema` — `is_disable`: optional coerced boolean (`z.coerce.boolean()`).
  - [ ] `updateUserSchema` — `name`: optional string, min 1 char, max 50 chars.
  - [ ] `updateUserSchema` — `username`: optional string, min 1 char, max 50 chars.
  - [ ] `updateUserSchema` — `role`: optional coerced integer, range 0–1 inclusive.
  - [ ] `updateUserSchema` — `is_disable`: optional coerced boolean.
  - [ ] `updatePasswordSchema` — `current_password`: required string, minimum 8 characters.
  - [ ] `updatePasswordSchema` — `new_password`: required string, minimum 8 characters.
  - [ ] `updatePasswordSchema` — `confirm_password`: required string, minimum 8 characters.
  - [ ] `updatePasswordSchema` — cross-field `.refine()`: `new_password` must equal `confirm_password`; on mismatch raises `ZodError` targeting the `confirm_password` path with message `"Passwords do not match"`.
  - [ ] `updatePasswordSchema` — `.refine()` runs after all individual field validations; both min-length and cross-field checks can fail simultaneously.
  - [ ] `updateProfileSchema` — `name`: optional string, min 1 char, max 50 chars.
  - [ ] `updateProfileSchema` — `username`: optional string, min 1 char, max 50 chars.
  - [ ] `z.coerce.number().int()` on `role` and `color` accepts numeric string inputs; rejects non-numeric strings with 422.
  - [ ] Inferred types exported: `CreateTagInput`, `UpdateTagInput`, `CreateUserInput`, `UpdateUserInput`, `UpdatePasswordInput`, `UpdateProfileInput`.
- **Dependencies:** `tasks/002-database-schema.md`
- **Output Files:**
  - `shared/schemas/tag.schema.ts`
  - `shared/schemas/user.schema.ts`

---

### Task 003-F: SearchTable Schema

- **Agent:** API
- **Spec:** `specs/validation-schemas.md`
- **Section:** `searchTable.schema.ts` ACs; Data Model (`SearchTableInput`); Edge Cases (`z.any()` and downstream sanitisation)
- **Acceptance Criteria:**
  - [ ] `searchTableSchema` — `search`: optional array of filter objects, each containing `key` (string), `type` (`z.enum` of `like | equal | date | datetime-bool | in | relation-equal | relation-like | bool`), and `value` (`z.any()`).
  - [ ] `searchTableSchema` — `sort`: optional object with `key` (string) and `direction` (`z.enum` of `asc | desc`).
  - [ ] `searchTableSchema` — `per_page`: optional positive integer (`z.number().int().positive()`).
  - [ ] `searchTableSchema` — `page`: optional positive integer.
  - [ ] `searchTableSchema` — `all`: optional boolean.
  - [ ] `value` field in search filters uses `z.any()` — no runtime type check is applied; downstream query builders are responsible for sanitising this value to prevent injection.
  - [ ] `SearchTableInput` inferred type is exported via `z.infer<typeof searchTableSchema>`.
- **Dependencies:** `tasks/002-database-schema.md`
- **Output Files:**
  - `shared/schemas/searchTable.schema.ts`

---

### Task 003-G: ZodError 422 Server Error Handler

- **Agent:** API
- **Spec:** `specs/validation-schemas.md`
- **Section:** General ACs (422 response format); Usage in Route Handlers ACs; Dependencies (`server/utils/`)
- **Acceptance Criteria:**
  - [ ] On a `ZodError`, the server returns HTTP `422` with the body:
    ```json
    {
      "status": 422,
      "errors": {
        "<field>": ["<message>"]
      }
    }
    ```
  - [ ] The `errors` object keys correspond to failing field paths (dot-notation for nested fields, e.g., `"items.0.position"`).
  - [ ] A `ZodError` thrown by `.parse()` inside any route handler is caught by this central handler — individual route handlers do not need `try/catch` for `ZodError`.
  - [ ] Coercion failure from a non-numeric string (e.g., `"abc"` into `z.coerce.number().int()`) is caught and returns a 422, not a 500.
  - [ ] The handler does not expose raw Zod internals beyond the `errors` map.
- **Dependencies:** `tasks/002-database-schema.md`
- **Output Files:**
  - `server/utils/validation-error.ts`

---

## Wave 2 — Frontend Integration

> **Skipped.** The spec explicitly states that client-side form state management (Pinia, `useForm`) and response schema validation are out of scope. The schemas in `shared/schemas/` are made available to client code via the Nuxt 4 `shared/` auto-import boundary without additional frontend work. No Wave 2 tasks are required.

---

## Wave 3 — Tests (Testing Agent)

### Task 003-H: Validation Schema Unit Tests

- **Agent:** Testing
- **Spec:** `specs/validation-schemas.md`
- **Section:** All ACs
- **Acceptance Criteria:**
  - [ ] `auth.schema.ts` — `loginSchema` parses valid `{ username, password }` input correctly.
  - [ ] `auth.schema.ts` — `loginSchema` rejects empty `username` (fails `.min(1)`).
  - [ ] `auth.schema.ts` — `loginSchema` rejects `password` shorter than 8 characters.
  - [ ] `auth.schema.ts` — `centrifugoChannelSchema` parses valid channel string; rejects empty string.
  - [ ] `board.schema.ts` — `createBoardSchema` parses valid input; rejects `name: ""`, `name` exceeding 50 chars, `description: ""`, `status` outside 0–1, `icon` outside 0–14.
  - [ ] `board.schema.ts` — `updateBoardSchema` accepts a fully partial payload; still rejects an explicitly supplied `name: ""`.
  - [ ] `board.schema.ts` — `createBoardSchema` coerces `status: "1"` (string) to `1` (integer).
  - [ ] `board.schema.ts` — `createBoardSchema` rejects `status: "abc"` with 422 (not 500).
  - [ ] `board.schema.ts` — `syncBoardMembersSchema` rejects non-integer array elements.
  - [ ] `boardList.schema.ts` — `updatePositionSchema` rejects items with float `position` values.
  - [ ] `card.schema.ts` — `createCardSchema` parses valid full input and minimal input (only required fields).
  - [ ] `card.schema.ts` — `quickCreateCardSchema` rejects extra fields (strict if applicable) and enforces `name` constraints.
  - [ ] `card.schema.ts` — `updateCardStatusSchema` coerces `status: "3"` to `3`; rejects `status: 5` (out of range 0–4).
  - [ ] `comment.schema.ts` — `updateCommentSchema` accepts `pin: true`; rejects `body: ""`.
  - [ ] `attachment.schema.ts` — `createAttachmentSchema` coerces `type: "2"` to `2`; rejects `type: 6` (out of range 0–5).
  - [ ] `tag.schema.ts` — `createTagSchema` coerces `color: "5"` to `5`; rejects `color: 9` (out of range 0–8); rejects `description` exceeding 255 chars.
  - [ ] `user.schema.ts` — `createUserSchema` coerces `role: "1"` to `1`; coerces `is_disable: "true"` to boolean.
  - [ ] `user.schema.ts` — `updatePasswordSchema` passes when `new_password === confirm_password`.
  - [ ] `user.schema.ts` — `updatePasswordSchema` fails with `ZodError` targeting `confirm_password` path and message `"Passwords do not match"` when passwords differ.
  - [ ] `user.schema.ts` — `updatePasswordSchema` simultaneously reports min-length failures and cross-field mismatch when both apply.
  - [ ] `searchTable.schema.ts` — `searchTableSchema` parses a valid fully-populated payload.
  - [ ] `searchTable.schema.ts` — `searchTableSchema` rejects an invalid `type` enum value in a search filter item.
  - [ ] `searchTable.schema.ts` — `searchTableSchema` rejects `direction` values outside `asc | desc`.
  - [ ] `searchTable.schema.ts` — `searchTableSchema` rejects non-positive `per_page` and `page` values.
  - [ ] All schemas — `.optional()` fields accept `undefined` and reject `null` (no `.nullable()` used).
  - [ ] All `.min(1)` string fields reject empty strings `""`.
  - [ ] `server/utils/validation-error.ts` — formats a `ZodError` into the `{ status: 422, errors: { "<field>": ["<message>"] } }` response shape, including dot-notation nested paths.
- **Dependencies:** Task 003-A, Task 003-B, Task 003-C, Task 003-D, Task 003-E, Task 003-F, Task 003-G
- **Output Files:**
  - `tests/unit/shared/schemas/auth.schema.test.ts`
  - `tests/unit/shared/schemas/board.schema.test.ts`
  - `tests/unit/shared/schemas/boardList.schema.test.ts`
  - `tests/unit/shared/schemas/card.schema.test.ts`
  - `tests/unit/shared/schemas/comment.schema.test.ts`
  - `tests/unit/shared/schemas/attachment.schema.test.ts`
  - `tests/unit/shared/schemas/tag.schema.test.ts`
  - `tests/unit/shared/schemas/user.schema.test.ts`
  - `tests/unit/shared/schemas/searchTable.schema.test.ts`
  - `tests/unit/server/utils/validation-error.test.ts`

---

## Dependency Graph

```
tasks/002-database-schema.md
         │
         ├── 003-A (auth.schema.ts)
         ├── 003-B (board.schema.ts, boardList.schema.ts)
         ├── 003-C (card.schema.ts)
         ├── 003-D (comment.schema.ts, attachment.schema.ts)
         ├── 003-E (tag.schema.ts, user.schema.ts)
         ├── 003-F (searchTable.schema.ts)
         └── 003-G (server/utils/validation-error.ts)
                  │
                  └── 003-H (all unit tests)
```

> Tasks 003-A through 003-G can be executed in parallel (no inter-schema dependencies). Task 003-H requires all Wave 1 tasks to be complete.
