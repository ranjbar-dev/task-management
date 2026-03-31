# Spec: Zod Validation Schemas

## Status: Approved

---

## Purpose

Define and implement a shared Zod validation schema layer (`shared/schemas/`) that enforces all API request body constraints across the full stack. These schemas serve as the single source of truth for input validation — consumed by Nitro server route handlers (server-side `parse()`) and optionally by client-side form composables. They also establish the contract for error response shape returned to callers on validation failure.

---

## User Stories

- **As an API developer**, I want every route handler to validate its request body against a well-typed Zod schema so that invalid data is rejected before reaching business logic.
- **As a frontend developer**, I want to import the same Zod schemas to validate form input on the client so that validation rules are never duplicated.
- **As a developer**, I want enum fields (role, status, color, icon) to accept numeric strings from form inputs and be coerced to integers automatically.
- **As an API consumer**, I want consistent 422 error responses with field-keyed error arrays so that I can map errors directly to form fields.
- **As a developer**, I want the `updatePasswordSchema` to cross-validate `new_password` and `confirm_password` so that mismatches are caught at the schema level.
- **As a developer**, I want a reusable `searchTableSchema` that covers pagination, sorting, and multi-type filtering for all admin list endpoints.

---

## Acceptance Criteria

### General

- [ ] All schema files live under `shared/schemas/` and are importable from both server (`~~/shared/schemas/`) and client code.
- [ ] Every schema file exports named schema constants (no default exports).
- [ ] All schemas use Zod (`z`) — no other validation library is used.
- [ ] On a `ZodError`, the server route handler returns HTTP `422` with the body:
  ```json
  {
    "status": 422,
    "errors": {
      "<field>": ["<message>"]
    }
  }
  ```
- [ ] The `errors` object keys correspond to the failing field paths (dot-notation for nested fields).

---

### `auth.schema.ts`

**`loginSchema`**
- [ ] `username`: required string, minimum 1 character.
- [ ] `password`: required string, minimum 8 characters.

**`centrifugoChannelSchema`**
- [ ] `channel`: required string, minimum 1 character.

---

### `board.schema.ts`

**`createBoardSchema`**
- [ ] `name`: required string, minimum 1 character, maximum 50 characters.
- [ ] `description`: required string, minimum 1 character.
- [ ] `status`: optional coerced integer (`z.coerce.number().int()`), range 0–1 inclusive, maps to `BoardStatusEnum`.
- [ ] `icon`: optional coerced integer, range 0–14 inclusive, maps to `BoardIconEnum`.

**`updateBoardSchema`**
- [ ] All fields from `createBoardSchema` made optional via `.partial()`.

**`syncBoardMembersSchema`**
- [ ] `user_ids`: required array of integers (each element `z.number().int()`).

---

### `boardList.schema.ts`

**`createBoardListSchema`**
- [ ] `name`: required string, minimum 1 character, maximum 50 characters.
- [ ] `position`: required integer (`z.number().int()`).

**`updateBoardListSchema`**
- [ ] All fields from `createBoardListSchema` made optional via `.partial()`.

**`updatePositionSchema`**
- [ ] `items`: required array of objects, each containing:
  - `id`: integer.
  - `position`: integer.

---

### `card.schema.ts`

**`createCardSchema`**
- [ ] `name`: required string, minimum 1 character, maximum 255 characters.
- [ ] `description`: optional string.
- [ ] `position`: required integer.
- [ ] `user_ids`: optional array of integers.
- [ ] `tag_ids`: optional array of integers.

**`quickCreateCardSchema`**
- [ ] `name`: required string, minimum 1 character, maximum 255 characters.
- [ ] No other fields accepted.

**`updateCardSchema`**
- [ ] `name`: optional string, minimum 1 character, maximum 255 characters.
- [ ] `description`: optional string.
- [ ] `board_list_id`: optional integer.
- [ ] `position`: optional integer.

**`syncCardMembersSchema`**
- [ ] `user_ids`: required array of integers.

**`syncCardTagsSchema`**
- [ ] `tag_ids`: required array of integers.

**`updateCardStatusSchema`**
- [ ] `user_id`: required integer.
- [ ] `status`: required coerced integer, range 0–4 inclusive, maps to `CardUserStatusEnum`.

**`updateCardPositionSchema`**
- [ ] `board_list_id`: required integer.
- [ ] `items`: required array of objects, each containing:
  - `id`: integer.
  - `position`: integer.

---

### `comment.schema.ts`

**`createCommentSchema`**
- [ ] `body`: required string, minimum 1 character.
- [ ] `attachment_id`: optional integer.

**`updateCommentSchema`**
- [ ] `body`: optional string, minimum 1 character.
- [ ] `pin`: optional boolean.

---

### `attachment.schema.ts`

**`createAttachmentSchema`**
- [ ] `name`: required string, minimum 1 character, maximum 50 characters.
- [ ] `type`: required coerced integer, range 0–5 inclusive.

**`updateAttachmentSchema`**
- [ ] All fields from `createAttachmentSchema` made optional via `.partial()`.

---

### `tag.schema.ts`

**`createTagSchema`**
- [ ] `name`: required string, minimum 1 character, maximum 50 characters.
- [ ] `color`: required coerced integer, range 0–8 inclusive, maps to `TagColorEnum`.
- [ ] `description`: optional string, maximum 255 characters.

**`updateTagSchema`**
- [ ] All fields from `createTagSchema` made optional via `.partial()`.

---

### `user.schema.ts`

**`createUserSchema`**
- [ ] `name`: required string, minimum 1 character, maximum 50 characters.
- [ ] `username`: required string, minimum 1 character, maximum 50 characters.
- [ ] `password`: required string, minimum 8 characters.
- [ ] `role`: optional coerced integer, range 0–1 inclusive, maps to `UserRoleEnum`.
- [ ] `is_disable`: optional coerced boolean (`z.coerce.boolean()`).

**`updateUserSchema`**
- [ ] `name`: optional string, minimum 1 character, maximum 50 characters.
- [ ] `username`: optional string, minimum 1 character, maximum 50 characters.
- [ ] `role`: optional coerced integer, range 0–1 inclusive.
- [ ] `is_disable`: optional coerced boolean.

**`updatePasswordSchema`**
- [ ] `current_password`: required string, minimum 8 characters.
- [ ] `new_password`: required string, minimum 8 characters.
- [ ] `confirm_password`: required string, minimum 8 characters.
- [ ] Cross-field refinement: `new_password` must equal `confirm_password`; on mismatch, a `ZodError` is raised targeting the `confirm_password` path with the message `"Passwords do not match"`.

**`updateProfileSchema`**
- [ ] `name`: optional string, minimum 1 character, maximum 50 characters.
- [ ] `username`: optional string, minimum 1 character, maximum 50 characters.

---

### `searchTable.schema.ts`

**`searchTableSchema`**
- [ ] `search`: optional array of filter objects, each containing:
  - `key`: string.
  - `type`: one of `like | equal | date | datetime-bool | in | relation-equal | relation-like | bool` (enforced via `z.enum`).
  - `value`: any (`z.any()`).
- [ ] `sort`: optional object containing:
  - `key`: string.
  - `direction`: one of `asc | desc`.
- [ ] `per_page`: optional positive integer (`z.number().int().positive()`).
- [ ] `page`: optional positive integer.
- [ ] `all`: optional boolean.

---

### Enum Coercion Behaviour

- [ ] Fields declared with `z.coerce.number().int()` accept both `number` and numeric `string` inputs (e.g., `"1"` is coerced to `1`). This supports HTML form submissions and query-string parameters.
- [ ] The following fields use coercion and map to their respective enums:

  | Field | Schema(s) | Enum |
  |-------|-----------|------|
  | `status` | `createBoardSchema`, `updateCardStatusSchema` | `BoardStatusEnum`, `CardUserStatusEnum` |
  | `icon` | `createBoardSchema` | `BoardIconEnum` |
  | `role` | `createUserSchema`, `updateUserSchema` | `UserRoleEnum` |
  | `color` | `createTagSchema` | `TagColorEnum` |
  | `type` | `createAttachmentSchema` | _(attachment type enum)_ |
  | `is_disable` | `createUserSchema`, `updateUserSchema` | _(boolean coercion from string)_ |

---

### Usage in Route Handlers

- [ ] Server route handlers import schemas from `~~/shared/schemas/<entity>.schema`.
- [ ] Request body is read via `readBody(event)` and passed to `schema.parse(body)`.
- [ ] A `ZodError` thrown by `.parse()` is caught by a server error handler and transformed into the 422 error response format documented above.
- [ ] Validated data (the return value of `.parse()`) is used exclusively downstream — raw `body` is never used after validation.

---

## Component Tree / File Structure

```
shared/
└── schemas/
    ├── auth.schema.ts          # loginSchema, centrifugoChannelSchema
    ├── board.schema.ts         # createBoardSchema, updateBoardSchema, syncBoardMembersSchema
    ├── boardList.schema.ts     # createBoardListSchema, updateBoardListSchema, updatePositionSchema
    ├── card.schema.ts          # createCardSchema, quickCreateCardSchema, updateCardSchema,
    │                           # syncCardMembersSchema, syncCardTagsSchema,
    │                           # updateCardStatusSchema, updateCardPositionSchema
    ├── comment.schema.ts       # createCommentSchema, updateCommentSchema
    ├── attachment.schema.ts    # createAttachmentSchema, updateAttachmentSchema
    ├── tag.schema.ts           # createTagSchema, updateTagSchema
    ├── user.schema.ts          # createUserSchema, updateUserSchema,
    │                           # updatePasswordSchema, updateProfileSchema
    └── searchTable.schema.ts   # searchTableSchema
```

These files are in `shared/` — the Nuxt 4 auto-import boundary shared between `server/` and `app/`. No build-time aliasing is required beyond `~~/shared/`.

---

## Data Model (TypeScript Interfaces & Inferred Types)

Each Zod schema exposes an inferred TypeScript type via `z.infer<>`. These types should be re-exported from each schema file for use in transformers, services, and composables.

```typescript
// Pattern for every schema file
import { z } from 'zod'

export const createBoardSchema = z.object({ /* ... */ })
export type CreateBoardInput = z.infer<typeof createBoardSchema>
```

| Schema | Inferred Type |
|--------|---------------|
| `loginSchema` | `LoginInput` |
| `centrifugoChannelSchema` | `CentrifugoChannelInput` |
| `createBoardSchema` | `CreateBoardInput` |
| `updateBoardSchema` | `UpdateBoardInput` |
| `syncBoardMembersSchema` | `SyncBoardMembersInput` |
| `createBoardListSchema` | `CreateBoardListInput` |
| `updateBoardListSchema` | `UpdateBoardListInput` |
| `updatePositionSchema` | `UpdatePositionInput` |
| `createCardSchema` | `CreateCardInput` |
| `quickCreateCardSchema` | `QuickCreateCardInput` |
| `updateCardSchema` | `UpdateCardInput` |
| `syncCardMembersSchema` | `SyncCardMembersInput` |
| `syncCardTagsSchema` | `SyncCardTagsInput` |
| `updateCardStatusSchema` | `UpdateCardStatusInput` |
| `updateCardPositionSchema` | `UpdateCardPositionInput` |
| `createCommentSchema` | `CreateCommentInput` |
| `updateCommentSchema` | `UpdateCommentInput` |
| `createAttachmentSchema` | `CreateAttachmentInput` |
| `updateAttachmentSchema` | `UpdateAttachmentInput` |
| `createTagSchema` | `CreateTagInput` |
| `updateTagSchema` | `UpdateTagInput` |
| `createUserSchema` | `CreateUserInput` |
| `updateUserSchema` | `UpdateUserInput` |
| `updatePasswordSchema` | `UpdatePasswordInput` |
| `updateProfileSchema` | `UpdateProfileInput` |
| `searchTableSchema` | `SearchTableInput` |

---

## Edge Cases

- **Empty string vs. missing field**: Zod `.min(1)` rejects empty strings (`""`). Routes must not silently pass empty strings through.
- **`null` vs. `undefined`**: Zod's `.optional()` accepts `undefined` but not `null`. Fields that may be `null` in the database must not be passed in request bodies unless explicitly typed with `.nullable()`.
- **`updatePasswordSchema` refinement order**: The `.refine()` runs after all individual field validations; both min-length and cross-field checks can fail simultaneously.
- **Coercion from non-numeric strings**: `z.coerce.number()` converts `"abc"` to `NaN`, which then fails `.int()`. The 422 error must still be returned, not a 500.
- **`z.any()` in `searchTableSchema`**: The `value` field has no runtime type check — downstream query builders are responsible for sanitising this value to prevent injection.
- **`updateBoardSchema` / `updateTagSchema` / `updateBoardListSchema` via `.partial()`**: All fields become optional, but if a field *is* supplied it must still satisfy its original constraints (e.g., a supplied `name: ""` still fails `.min(1)`).
- **Position integers**: `z.number().int()` rejects floats. Clients must send integer positions, not `1.5`.
- **Large `user_ids` / `tag_ids` arrays**: No maximum array length is enforced by the schema. The service layer must handle bulk sync limits.

---

## Non-Goals

- This spec does not cover **database-level uniqueness** validation (e.g., unique username). That is enforced at the service layer.
- This spec does not define **response schemas** (output shape validation). See the transformers spec.
- This spec does not cover **file upload validation** (MIME type, size). That is handled separately in the attachment upload route.
- This spec does not define the enum value constants themselves. Those live in `shared/enums/`.
- Client-side form state management (Pinia, `useForm`) is out of scope — schemas are provided as primitives for composables to consume.
- Rate limiting and authentication are out of scope.

---

## Dependencies

| Dependency | Version / Source | Notes |
|------------|------------------|-------|
| `zod` | `^3.x` | Peer dependency; already declared in `package.json` |
| `shared/enums/` | Internal | Enum numeric ranges must stay in sync with schema `.min()` / `.max()` bounds |
| `server/utils/` | Internal | A shared error handler utility must catch `ZodError` and format the 422 response |
| Nuxt 4 `shared/` layer | Framework | Enables auto-import of shared modules in both `app/` and `server/` |
