# Tasks: Database Schema
**Spec:** `specs/database-schema.md`
**Status:** pending

## Overview

Implement the complete PostgreSQL database layer for the task management application. This covers 6 shared TypeScript enum files, 3 shared type interface files, 11 Drizzle ORM table schemas, a consolidated relations file, a schema index re-export, and unit/integration tests that verify migrations, FK constraints, and observer-hook contracts. All schema work is server-side (API Agent); shared enums and type models are included here because they are foundational to both server routes and client code. No UI is produced by this task.

---

## Wave 1 — Data Layer (API Agent)

### Task 002-A: Shared Enums
- **Agent:** API
- **Spec:** `specs/database-schema.md`
- **Section:** Enums (AC-52 – AC-58); Data Model → TypeScript Enums
- **Acceptance Criteria:**
  - [ ] AC-52: All enums are `const` objects with `as const` — no TypeScript `enum` keyword used
  - [ ] AC-53: `UserRoleEnum` exports `{ NormalUser: 0, AdminUser: 1 }` and `UserRole` type
  - [ ] AC-54: `BoardStatusEnum` exports `{ Deactive: 0, Active: 1 }` and `BoardStatus` type
  - [ ] AC-55: `BoardIconEnum` has exactly 15 entries (0–14); `BoardIconNameMap` maps each numeric value to its Material Symbols icon name string
  - [ ] AC-56: `CardUserStatusEnum` exports `{ None: 0, Pending: 1, Doing: 2, OnHold: 3, Done: 4 }` and `CardUserStatus` type
  - [ ] AC-57: `FileTypeEnum` exports `{ Other: 0, Image: 1, Video: 2, Audio: 3, PDF: 4, Excel: 5 }` and `FileType` type
  - [ ] AC-58: `TagColorEnum` exports `{ Red: 0, Purple: 1, Green: 2, Orange: 3, Blue: 4, Gray: 5, Cyan: 6, Magenta: 7, Violet: 8 }` and `TagColor` type
- **Dependencies:** `tasks/001-infrastructure-setup.md`
- **Output Files:**
  - `shared/enums/UserRoleEnum.ts`
  - `shared/enums/BoardStatusEnum.ts`
  - `shared/enums/BoardIconEnum.ts`
  - `shared/enums/CardUserStatusEnum.ts`
  - `shared/enums/FileTypeEnum.ts`
  - `shared/enums/TagColorEnum.ts`

---

### Task 002-B: Shared TypeScript Model Interfaces
- **Agent:** API
- **Spec:** `specs/database-schema.md`
- **Section:** Data Model → TypeScript Model Interfaces; Component Tree (`shared/types/`)
- **Acceptance Criteria:**
  - [ ] `shared/types/models.ts` defines interfaces: `UserModel`, `BoardModel`, `BoardListModel`, `CardModel`, `CardUserModel`, `CardLogModel`, `CommentModel`, `AttachmentModel`, `TagModel`, `NotificationModel`, `BoardUserModel` — exactly as specified in the Data Model section
  - [ ] `shared/types/enums.ts` re-exports all enum value types (`UserRole`, `BoardStatus`, `BoardIcon`, `CardUserStatus`, `FileType`, `TagColor`) from `shared/enums/`
  - [ ] `shared/types/api.ts` defines the standard API response and error shape types used across server routes and client composables
  - [ ] All types use imported enum types from `shared/enums/` (e.g., `role: UserRole`, `status: BoardStatus`)
  - [ ] No `any` — use `unknown` with type guards where truly dynamic shapes appear
- **Dependencies:** `Task 002-A`
- **Output Files:**
  - `shared/types/models.ts`
  - `shared/types/enums.ts`
  - `shared/types/api.ts`

---

### Task 002-C: Users Schema
- **Agent:** API
- **Spec:** `specs/database-schema.md`
- **Section:** Users (AC-1 – AC-4); Data Model → `users.ts`
- **Acceptance Criteria:**
  - [ ] AC-1: `users` table has all 11 columns with the exact types, constraints, and defaults specified: `id` (serial PK), `name` (varchar 50 NOT NULL), `username` (varchar 50 NOT NULL UNIQUE), `password` (varchar 255 NOT NULL), `avatar` (varchar 255 nullable), `telegram` (jsonb nullable), `notification` (jsonb nullable), `role` (smallint NOT NULL DEFAULT 0), `is_disable` (boolean NOT NULL DEFAULT false), `created_at` (timestamptz NOT NULL DEFAULT now()), `updated_at` (timestamptz NOT NULL DEFAULT now())
  - [ ] AC-2: `telegram` jsonb column is typed as `{ id: string; chat_id?: string; username?: string }` using Drizzle `.$type<>()`
  - [ ] AC-3: `notification` jsonb column is typed as `Record<string, boolean>` using Drizzle `.$type<>()`
  - [ ] AC-4: Observer comment documents that on user creation `initializeUserDefaults()` must be called: sets `telegram.id` to a random 12-character token and sets all notification preference keys to `true`
- **Dependencies:** `tasks/001-infrastructure-setup.md`
- **Output Files:**
  - `server/database/schema/users.ts`

---

### Task 002-D: Board Domain Schemas
- **Agent:** API
- **Spec:** `specs/database-schema.md`
- **Section:** Boards (AC-5 – AC-8), Board Users (AC-9 – AC-11), Board Lists (AC-12 – AC-15); Data Model → `boards.ts`, `boardUsers.ts`, `boardLists.ts`
- **Acceptance Criteria:**
  - [ ] AC-5: `boards` table has all 8 columns with exact types and defaults: `id` (serial PK), `user_id` (integer NOT NULL FK → users.id), `name` (varchar 50 NOT NULL), `description` (text NOT NULL), `status` (smallint NOT NULL DEFAULT 0), `icon` (smallint NOT NULL DEFAULT 0), `created_at`, `updated_at`
  - [ ] AC-6: Observer comment documents `broadcastToCentrifugo('board_create')` after board insert
  - [ ] AC-7: Observer comment documents `broadcastToCentrifugo('board_update')` after board update
  - [ ] AC-8: Observer comment documents `broadcastToCentrifugo('board_delete')` after board delete
  - [ ] AC-9: `board_user` table has all 5 columns: `id` (serial PK), `user_id` (FK → users.id CASCADE DELETE), `board_id` (FK → boards.id CASCADE DELETE), `created_at`, `updated_at`
  - [ ] AC-10: `user_id` FK on `board_user` has `onDelete: 'cascade'`
  - [ ] AC-11: `board_id` FK on `board_user` has `onDelete: 'cascade'`
  - [ ] AC-12: `board_lists` table has all 6 columns: `id` (serial PK), `board_id` (FK → boards.id CASCADE DELETE), `name` (varchar 50 NOT NULL), `position` (integer NOT NULL), `created_at`, `updated_at`
  - [ ] AC-13: Observer comment documents `broadcastToCentrifugo('list_create', boardChannel)` after list insert
  - [ ] AC-14: Observer comment documents `broadcastToCentrifugo('list_update', boardChannel)` after list update
  - [ ] AC-15: Observer comment documents `broadcastToCentrifugo('list_delete', boardChannel)` after list delete
- **Dependencies:** `tasks/001-infrastructure-setup.md`, `Task 002-C`
- **Output Files:**
  - `server/database/schema/boards.ts`
  - `server/database/schema/boardUsers.ts`
  - `server/database/schema/boardLists.ts`

---

### Task 002-E: Card Domain Schemas
- **Agent:** API
- **Spec:** `specs/database-schema.md`
- **Section:** Cards (AC-16 – AC-23), Card Users (AC-24 – AC-25), Card Logs (AC-26 – AC-29); Data Model → `cards.ts`, `cardUsers.ts`, `cardLogs.ts`
- **Acceptance Criteria:**
  - [ ] AC-16: `cards` table has all 9 columns: `id` (serial PK), `board_list_id` (integer nullable FK → board_lists.id SET NULL on delete), `name` (varchar 255 NOT NULL), `description` (text nullable), `position` (smallint NOT NULL), `tags` (jsonb NOT NULL DEFAULT []), `deleted_at` (timestamptz nullable), `created_at`, `updated_at`
  - [ ] AC-17: Observer comment documents `softDeleteCard()` (sets `deletedAt = now()`) and `restoreCard()` (sets `deletedAt = null`)
  - [ ] AC-18: Comment in schema file states that all read queries must apply `isNull(cards.deletedAt)` filter
  - [ ] AC-19: Observer comment documents `getCombinedActivity(cardId)` merging `card_logs` (type=1 comments, type=2 attachments) into a single chronological array
  - [ ] AC-20: Observer comment documents `createCardLog(type=0)` + `broadcastToCentrifugo('card_create')` after card insert
  - [ ] AC-21: Observer comment documents `createCardLog(type=0)` + `broadcastToCentrifugo('card_update')` after card update
  - [ ] AC-22: Observer comment documents `createCardLog(type=0)` + `broadcastToCentrifugo('card_delete')` after card soft-delete
  - [ ] AC-23: Observer comment documents `broadcastToCentrifugo('card_add_users')` and `broadcastToCentrifugo('card_remove_users')` on member changes
  - [ ] AC-24: `card_user` table has all 6 columns: `id` (serial PK), `user_id` (FK → users.id CASCADE DELETE), `card_id` (FK → cards.id CASCADE DELETE), `status` (smallint NOT NULL DEFAULT 0), `created_at`, `updated_at`
  - [ ] AC-25: Comment in `cardUsers.ts` documents `CardUserStatusEnum` values: None(0), Pending(1), Doing(2), OnHold(3), Done(4)
  - [ ] AC-26: `card_logs` table has all 8 columns: `id` (serial PK), `user_id` (FK → users.id), `card_id` (FK → cards.id), `board_id` (FK → boards.id), `type` (smallint NOT NULL), `data` (jsonb NOT NULL DEFAULT {}), `created_at`, `updated_at`
  - [ ] AC-27: Comment in `cardLogs.ts` documents type values: 0 = card event, 1 = comment event, 2 = attachment event
  - [ ] AC-28: `data` jsonb is typed via `.$type<{ message?: string; changes?: Record<string, any>; old_data?: Record<string, any> }>()`
  - [ ] AC-29: Observer comment documents `broadcastToCentrifugo('log_create', cardChannel)` after card log insert
- **Dependencies:** `tasks/001-infrastructure-setup.md`, `Task 002-C`, `Task 002-D`
- **Output Files:**
  - `server/database/schema/cards.ts`
  - `server/database/schema/cardUsers.ts`
  - `server/database/schema/cardLogs.ts`

---

### Task 002-F: Comment & Attachment Schemas
- **Agent:** API
- **Spec:** `specs/database-schema.md`
- **Section:** Comments (AC-30 – AC-33), Attachments (AC-34 – AC-38); Data Model → `comments.ts`, `attachments.ts`
- **Acceptance Criteria:**
  - [ ] AC-30: `comments` table has all 8 columns: `id` (serial PK), `user_id` (FK → users.id CASCADE DELETE), `card_id` (FK → cards.id CASCADE DELETE), `attachment_id` (integer nullable FK → attachments.id), `body` (text NOT NULL), `pin` (boolean NOT NULL DEFAULT false), `created_at`, `updated_at`
  - [ ] AC-31: Observer comment documents `createCardLog(type=1)` + `broadcastToCentrifugo('comment_create')` after comment insert
  - [ ] AC-32: Observer comment documents `createCardLog(type=1)` + `broadcastToCentrifugo('comment_update')` after comment update
  - [ ] AC-33: Observer comment documents `createCardLog(type=1)` + `broadcastToCentrifugo('comment_delete')` after comment delete
  - [ ] AC-34: `attachments` table has all 8 columns: `id` (serial PK), `user_id` (FK → users.id CASCADE DELETE), `card_id` (FK → cards.id CASCADE DELETE), `name` (varchar 50 NOT NULL), `type` (smallint NOT NULL), `path` (varchar 255 NOT NULL), `created_at`, `updated_at`
  - [ ] AC-35: Comment in `attachments.ts` documents `FileTypeEnum` values: Other(0), Image(1), Video(2), Audio(3), PDF(4), Excel(5)
  - [ ] AC-36: Observer comment documents `createCardLog(type=2)` + `broadcastToCentrifugo('attachment_create')` after attachment insert
  - [ ] AC-37: Observer comment documents `broadcastToCentrifugo('attachment_update')` after attachment update
  - [ ] AC-38: Observer comment documents `broadcastToCentrifugo('attachment_delete')` + `removeFileFromDisk(path)` after attachment delete
- **Dependencies:** `tasks/001-infrastructure-setup.md`, `Task 002-C`, `Task 002-E`
- **Output Files:**
  - `server/database/schema/comments.ts`
  - `server/database/schema/attachments.ts`

---

### Task 002-G: Tag & Notification Schemas
- **Agent:** API
- **Spec:** `specs/database-schema.md`
- **Section:** Tags (AC-39 – AC-41), Notifications (AC-42 – AC-45); Data Model → `tags.ts`, `notifications.ts`
- **Acceptance Criteria:**
  - [ ] AC-39: `tags` table has all 6 columns: `id` (serial PK), `name` (varchar 50 NOT NULL UNIQUE), `color` (smallint NOT NULL), `description` (varchar 255 NOT NULL DEFAULT 'default'), `created_at`, `updated_at`
  - [ ] AC-40: Comment in `tags.ts` documents `TagColorEnum` values: Red(0)–Violet(8)
  - [ ] AC-41: Comment in `tags.ts` states that card-tag associations are stored as a JSONB `number[]` in `cards.tags` — no junction table exists
  - [ ] AC-42: `notifications` table has all 7 columns: `id` (uuid PK DEFAULT gen_random_uuid()), `type` (varchar 255 NOT NULL), `notifiable_id` (integer NOT NULL FK → users.id CASCADE DELETE), `data` (text NOT NULL), `read_at` (timestamptz nullable), `created_at`, `updated_at`
  - [ ] AC-43: Comment in `notifications.ts` states `type` stores a notification class name string (e.g., `'CreateNewBoardNotification'`)
  - [ ] AC-44: Comment in `notifications.ts` states `data` stores a JSON-stringified payload as `text` — always `JSON.parse()` before use, always `JSON.stringify()` before insert
  - [ ] AC-45: Comment in `notifications.ts` states `read_at` is NULL for unread notifications
- **Dependencies:** `tasks/001-infrastructure-setup.md`, `Task 002-C`
- **Output Files:**
  - `server/database/schema/tags.ts`
  - `server/database/schema/notifications.ts`

---

### Task 002-H: Drizzle Relations
- **Agent:** API
- **Spec:** `specs/database-schema.md`
- **Section:** Drizzle Relations (AC-46 – AC-50); Data Model → `relations.ts`
- **Acceptance Criteria:**
  - [ ] AC-46: All Drizzle relations are defined exclusively in `server/database/schema/relations.ts` — no `relations()` calls inside individual table schema files
  - [ ] AC-47: `usersRelations` defines `many(boardUsers)`, `many(cardUsers)`, `many(cardLogs)`, `many(comments)`, `many(attachments)`, `many(notifications)`, `many(boards)` (as `createdBoards`)
  - [ ] AC-48: `boardsRelations` defines `one(users)` as `creator` (via `boards.userId → users.id`), `many(boardUsers)`, `many(boardLists)`, `many(cardLogs)`
  - [ ] AC-49: `cardsRelations` defines `one(boardLists)` as `list` (via `cards.boardListId → boardLists.id`), `many(cardUsers)`, `many(cardLogs)`, `many(comments)`, `many(attachments)`
  - [ ] AC-50: `commentsRelations` defines `one(attachments)` as optional linked attachment (via `comments.attachmentId → attachments.id`)
  - [ ] Relations for `boardUsers`, `boardLists`, `cardUsers`, `cardLogs`, `attachments`, `notifications` are also defined, matching the Data Model section exactly
- **Dependencies:** `Task 002-C`, `Task 002-D`, `Task 002-E`, `Task 002-F`, `Task 002-G`
- **Output Files:**
  - `server/database/schema/relations.ts`

---

### Task 002-I: Schema Index
- **Agent:** API
- **Spec:** `specs/database-schema.md`
- **Section:** Schema Index (AC-51); Component Tree → `schema/index.ts`
- **Acceptance Criteria:**
  - [ ] AC-51: `server/database/schema/index.ts` re-exports all 11 table schemas (`users`, `boards`, `boardUsers`, `boardLists`, `cards`, `cardUsers`, `cardLogs`, `comments`, `attachments`, `tags`, `notifications`) and the `relations.ts` file via named exports
  - [ ] All imports in `relations.ts` and consuming files use the `index.ts` barrel — no deep imports from individual schema files outside the `schema/` directory
- **Dependencies:** `Task 002-H` (which transitively requires 002-C through 002-G)
- **Output Files:**
  - `server/database/schema/index.ts`

---

## Wave 3 — Tests (Testing Agent)

### Task 002-J: Database Schema Tests
- **Agent:** Testing
- **Spec:** `specs/database-schema.md`
- **Section:** All ACs
- **Acceptance Criteria:**
  - [ ] Schema migrations generated by `drizzle-kit generate` run against a test PostgreSQL instance without errors
  - [ ] All 11 tables are created with the correct column types and NOT NULL constraints (verifiable via `information_schema.columns`)
  - [ ] UNIQUE constraints are present on `users.username` and `tags.name`
  - [ ] CASCADE DELETE on `board_user.user_id`, `board_user.board_id`, `board_lists.board_id`, `card_user.user_id`, `card_user.card_id`, `comments.user_id`, `comments.card_id`, `attachments.user_id`, `attachments.card_id`, `notifications.notifiable_id`
  - [ ] SET NULL on `cards.board_list_id` when the referenced `board_list` row is deleted
  - [ ] `notifications.id` column type is UUID; `users.id` is serial integer
  - [ ] `cards.deleted_at` is nullable; inserting a card sets `deleted_at` to NULL by default
  - [ ] All 6 enum files export the correct `const` objects and derived types without TypeScript compilation errors
  - [ ] `shared/types/models.ts` compiles without errors and all interfaces reference the correct enum types from `shared/enums/`
  - [ ] `server/database/schema/index.ts` re-exports are resolvable — no missing module errors
- **Dependencies:** `Task 002-A`, `Task 002-B`, `Task 002-C`, `Task 002-D`, `Task 002-E`, `Task 002-F`, `Task 002-G`, `Task 002-H`, `Task 002-I`
- **Output Files:**
  - `tests/unit/database/schema.test.ts`
  - `tests/unit/database/enums.test.ts`
