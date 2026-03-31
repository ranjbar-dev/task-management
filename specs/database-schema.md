# Spec: Database Schema

## Status: Approved

## Purpose
Define the complete PostgreSQL database layer for the task management application. This spec covers all 11 Drizzle ORM table schemas, their column definitions and constraints, inter-table relations, TypeScript enums shared across server and client, and the ERD. It exists so that a backend developer can scaffold the database from scratch — running drizzle-kit migrations and the seed script — with a full understanding of every constraint, default, cascade rule, soft-delete pattern, and post-write side effect.

## User Stories
- As a backend developer, I want a single reference for every table schema so that I can implement the Drizzle schemas without consulting scattered source files.
- As a backend developer, I want all enum definitions documented so that I can use the same numeric values consistently across server routes and client code.
- As a backend developer, I want cascade and soft-delete rules documented so that I can write correct queries that respect referential integrity.
- As a backend developer, I want all post-write observer hooks documented alongside each schema so that I never miss a side effect when implementing a route handler.
- As a backend developer, I want the ERD documented so that I can reason about query join paths before writing code.

## Acceptance Criteria

### Users
- [ ] AC-1: `users` table has columns: `id` (serial PK), `name` (varchar 50 NOT NULL), `username` (varchar 50 NOT NULL UNIQUE), `password` (varchar 255 NOT NULL), `avatar` (varchar 255 nullable), `telegram` (jsonb nullable), `notification` (jsonb nullable), `role` (smallint NOT NULL DEFAULT 0), `is_disable` (boolean NOT NULL DEFAULT false), `created_at` (timestamptz NOT NULL DEFAULT now()), `updated_at` (timestamptz NOT NULL DEFAULT now())
- [ ] AC-2: `telegram` jsonb column is typed as `{ id: string; chat_id?: string; username?: string }`
- [ ] AC-3: `notification` jsonb column is typed as `Record<string, boolean>` with 12 preference keys
- [ ] AC-4: On user creation, `initializeUserDefaults()` is called: sets `telegram.id` to a random 12-character token and sets all 12 notification preference keys to `true`

### Boards
- [ ] AC-5: `boards` table has columns: `id` (serial PK), `user_id` (integer NOT NULL FK → users.id), `name` (varchar 50 NOT NULL), `description` (text NOT NULL), `status` (smallint NOT NULL DEFAULT 0), `icon` (smallint NOT NULL DEFAULT 0), `created_at` (timestamptz NOT NULL DEFAULT now()), `updated_at` (timestamptz NOT NULL DEFAULT now())
- [ ] AC-6: After board create → `broadcastToCentrifugo()` is called with event `board_create`
- [ ] AC-7: After board update → `broadcastToCentrifugo()` is called with event `board_update`
- [ ] AC-8: After board delete → `broadcastToCentrifugo()` is called with event `board_delete`

### Board Users
- [ ] AC-9: `board_user` table has columns: `id` (serial PK), `user_id` (integer NOT NULL FK → users.id CASCADE DELETE), `board_id` (integer NOT NULL FK → boards.id CASCADE DELETE), `created_at`, `updated_at`
- [ ] AC-10: Deleting a user cascade-deletes all `board_user` rows for that user
- [ ] AC-11: Deleting a board cascade-deletes all `board_user` rows for that board

### Board Lists
- [ ] AC-12: `board_lists` table has columns: `id` (serial PK), `board_id` (integer NOT NULL FK → boards.id CASCADE DELETE), `name` (varchar 50 NOT NULL), `position` (integer NOT NULL), `created_at`, `updated_at`
- [ ] AC-13: After list create → `broadcastToCentrifugo()` is called with event `list_create` to the board channel
- [ ] AC-14: After list update → `broadcastToCentrifugo()` is called with event `list_update`
- [ ] AC-15: After list delete → `broadcastToCentrifugo()` is called with event `list_delete`

### Cards
- [ ] AC-16: `cards` table has columns: `id` (serial PK), `board_list_id` (integer nullable FK → board_lists.id SET NULL on delete), `name` (varchar 255 NOT NULL), `description` (text nullable), `position` (smallint NOT NULL), `tags` (jsonb NOT NULL DEFAULT []), `deleted_at` (timestamptz nullable), `created_at`, `updated_at`
- [ ] AC-17: `deleted_at` IS NULL for all active card queries — soft delete implemented via `softDeleteCard()` setting `deleted_at = now()` and `restoreCard()` setting `deleted_at = null`
- [ ] AC-18: All card queries in route handlers apply Drizzle `isNull(cards.deletedAt)` filter
- [ ] AC-19: `getCombinedActivity(cardId)` merges `card_logs` (type=1 comments, type=2 attachments) into a single chronological array
- [ ] AC-20: After card create → CardLog (type=0) is written + `card_create` broadcast
- [ ] AC-21: After card update → CardLog is written + `card_update` broadcast
- [ ] AC-22: After card soft-delete → CardLog is written + `card_delete` broadcast
- [ ] AC-23: Card member changes broadcast `card_add_users` and `card_remove_users` events

### Card Users
- [ ] AC-24: `card_user` table has columns: `id` (serial PK), `user_id` (integer NOT NULL FK → users.id CASCADE DELETE), `card_id` (integer NOT NULL FK → cards.id CASCADE DELETE), `status` (smallint NOT NULL DEFAULT 0), `created_at`, `updated_at`
- [ ] AC-25: `status` accepts values 0 (None), 1 (Pending), 2 (Doing), 3 (OnHold), 4 (Done) from `CardUserStatusEnum`

### Card Logs
- [ ] AC-26: `card_logs` table has columns: `id` (serial PK), `user_id` (integer NOT NULL FK → users.id), `card_id` (integer NOT NULL FK → cards.id), `board_id` (integer NOT NULL FK → boards.id), `type` (smallint NOT NULL), `data` (jsonb NOT NULL DEFAULT {}), `created_at`, `updated_at`
- [ ] AC-27: `type` values: 0 = card event, 1 = comment event, 2 = attachment event
- [ ] AC-28: `data` jsonb is typed as `{ message?: string; changes?: Record<string, any>; old_data?: Record<string, any> }`
- [ ] AC-29: After card log is created → `log_create` is broadcast to the card channel

### Comments
- [ ] AC-30: `comments` table has columns: `id` (serial PK), `user_id` (integer NOT NULL FK → users.id CASCADE DELETE), `card_id` (integer NOT NULL FK → cards.id CASCADE DELETE), `attachment_id` (integer nullable FK → attachments.id), `body` (text NOT NULL), `pin` (boolean NOT NULL DEFAULT false), `created_at`, `updated_at`
- [ ] AC-31: After comment create → CardLog (type=1) is written + `comment_create` broadcast
- [ ] AC-32: After comment update → CardLog is written + `comment_update` broadcast
- [ ] AC-33: After comment delete → CardLog is written + `comment_delete` broadcast

### Attachments
- [ ] AC-34: `attachments` table has columns: `id` (serial PK), `user_id` (integer NOT NULL FK → users.id CASCADE DELETE), `card_id` (integer NOT NULL FK → cards.id CASCADE DELETE), `name` (varchar 50 NOT NULL), `type` (smallint NOT NULL), `path` (varchar 255 NOT NULL), `created_at`, `updated_at`
- [ ] AC-35: `type` accepts values 0 (Other), 1 (Image), 2 (Video), 3 (Audio), 4 (PDF), 5 (Excel) from `FileTypeEnum`
- [ ] AC-36: After attachment create → CardLog (type=2) is written + `attachment_create` broadcast
- [ ] AC-37: After attachment update → `attachment_update` broadcast
- [ ] AC-38: After attachment delete → `attachment_delete` broadcast + physical file is removed from disk

### Tags
- [ ] AC-39: `tags` table has columns: `id` (serial PK), `name` (varchar 50 NOT NULL UNIQUE), `color` (smallint NOT NULL), `description` (varchar 255 NOT NULL DEFAULT 'default'), `created_at`, `updated_at`
- [ ] AC-40: `color` accepts values 0–8 from `TagColorEnum`: Red(0), Purple(1), Green(2), Orange(3), Blue(4), Gray(5), Cyan(6), Magenta(7), Violet(8)
- [ ] AC-41: Card-tag associations are stored as a JSONB array of tag IDs in `cards.tags` (no junction table)

### Notifications
- [ ] AC-42: `notifications` table has columns: `id` (uuid PK DEFAULT gen_random_uuid()), `type` (varchar 255 NOT NULL), `notifiable_id` (integer NOT NULL FK → users.id CASCADE DELETE), `data` (text NOT NULL), `read_at` (timestamptz nullable), `created_at`, `updated_at`
- [ ] AC-43: `type` stores a notification class name string (e.g., `'CreateNewBoardNotification'`)
- [ ] AC-44: `data` stores a JSON-stringified payload (text, not jsonb)
- [ ] AC-45: `read_at` is NULL for unread notifications

### Drizzle Relations
- [ ] AC-46: All Drizzle relations are defined in `server/database/schema/relations.ts` — no relations defined inline in table schema files
- [ ] AC-47: `users` has `many(boardUsers)`, `many(cardUsers)`, `many(cardLogs)`, `many(comments)`, `many(attachments)`, `many(notifications)`, `many(boards)` (as creator)
- [ ] AC-48: `boards` has `one(users)` as creator, `many(boardUsers)`, `many(boardLists)`, `many(cardLogs)`
- [ ] AC-49: `cards` has `one(boardLists)` as list, `many(cardUsers)`, `many(cardLogs)`, `many(comments)`, `many(attachments)`
- [ ] AC-50: `comments` has `one(attachments)` as optional linked attachment

### Schema Index
- [ ] AC-51: `server/database/schema/index.ts` re-exports all 11 table schemas and the relations file

### Enums
- [ ] AC-52: All enums are defined as `const` objects with `as const` (not TypeScript `enum` keyword) in `shared/enums/`
- [ ] AC-53: `UserRoleEnum`: `{ NormalUser: 0, AdminUser: 1 }`
- [ ] AC-54: `BoardStatusEnum`: `{ Deactive: 0, Active: 1 }`
- [ ] AC-55: `BoardIconEnum` has 15 entries (0–14) mapping to icon names; `BoardIconNameMap` maps numeric values to Material Symbols icon name strings
- [ ] AC-56: `CardUserStatusEnum`: `{ None: 0, Pending: 1, Doing: 2, OnHold: 3, Done: 4 }`
- [ ] AC-57: `FileTypeEnum`: `{ Other: 0, Image: 1, Video: 2, Audio: 3, PDF: 4, Excel: 5 }`
- [ ] AC-58: `TagColorEnum`: `{ Red: 0, Purple: 1, Green: 2, Orange: 3, Blue: 4, Gray: 5, Cyan: 6, Magenta: 7, Violet: 8 }`

## Component Tree / File Structure

```
server/database/
├── schema/
│   ├── users.ts           # users table
│   ├── boards.ts          # boards table
│   ├── boardUsers.ts      # board_user junction table
│   ├── boardLists.ts      # board_lists table
│   ├── cards.ts           # cards table (soft delete)
│   ├── cardUsers.ts       # card_user pivot table
│   ├── cardLogs.ts        # card_logs audit table
│   ├── comments.ts        # comments table
│   ├── attachments.ts     # attachments table
│   ├── tags.ts            # tags table
│   ├── notifications.ts   # notifications table
│   ├── relations.ts       # all Drizzle relations
│   └── index.ts           # re-exports all schemas
├── migrations/            # generated by drizzle-kit generate
├── seed.ts                # database seeder
└── index.ts               # Drizzle client instance (db)

shared/
├── enums/
│   ├── UserRoleEnum.ts
│   ├── BoardStatusEnum.ts
│   ├── BoardIconEnum.ts
│   ├── CardUserStatusEnum.ts
│   ├── FileTypeEnum.ts
│   └── TagColorEnum.ts
└── types/
    ├── models.ts          # TypeScript model interfaces
    ├── enums.ts           # re-exported enum types
    └── api.ts             # API response/error types
```

## Data Model

### TypeScript Enums (`shared/enums/`)

```typescript
// shared/enums/UserRoleEnum.ts
export const UserRoleEnum = { NormalUser: 0, AdminUser: 1 } as const
export type UserRole = (typeof UserRoleEnum)[keyof typeof UserRoleEnum]

// shared/enums/BoardStatusEnum.ts
export const BoardStatusEnum = { Deactive: 0, Active: 1 } as const
export type BoardStatus = (typeof BoardStatusEnum)[keyof typeof BoardStatusEnum]

// shared/enums/BoardIconEnum.ts
export const BoardIconEnum = {
  bolt: 0, box: 1, brush: 2, camera: 3, crown: 4,
  database: 5, euro: 6, folder: 7, home: 8, map: 9,
  mobile: 10, rocket: 11, star: 12, store: 13, tag: 14,
} as const
export type BoardIcon = (typeof BoardIconEnum)[keyof typeof BoardIconEnum]

export const BoardIconNameMap: Record<BoardIcon, string> = {
  0: 'bolt',           // Material Symbol
  1: 'inventory_2',
  2: 'brush',
  3: 'photo_camera',
  4: 'crown',
  5: 'database',
  6: 'euro',
  7: 'folder',
  8: 'home',
  9: 'map',
  10: 'smartphone',
  11: 'rocket_launch',
  12: 'star',
  13: 'store',
  14: 'sell',
}

// shared/enums/CardUserStatusEnum.ts
export const CardUserStatusEnum = {
  None: 0, Pending: 1, Doing: 2, OnHold: 3, Done: 4,
} as const
export type CardUserStatus = (typeof CardUserStatusEnum)[keyof typeof CardUserStatusEnum]

// shared/enums/FileTypeEnum.ts
export const FileTypeEnum = {
  Other: 0, Image: 1, Video: 2, Audio: 3, PDF: 4, Excel: 5,
} as const
export type FileType = (typeof FileTypeEnum)[keyof typeof FileTypeEnum]

// shared/enums/TagColorEnum.ts
export const TagColorEnum = {
  Red: 0, Purple: 1, Green: 2, Orange: 3, Blue: 4,
  Gray: 5, Cyan: 6, Magenta: 7, Violet: 8,
} as const
export type TagColor = (typeof TagColorEnum)[keyof typeof TagColorEnum]
```

### Drizzle Schemas (`server/database/schema/`)

```typescript
// server/database/schema/users.ts
export const users = pgTable('users', {
  id:           serial('id').primaryKey(),
  name:         varchar('name', { length: 50 }).notNull(),
  username:     varchar('username', { length: 50 }).notNull().unique(),
  password:     varchar('password', { length: 255 }).notNull(),
  avatar:       varchar('avatar', { length: 255 }),
  telegram:     jsonb('telegram').$type<{ id: string; chat_id?: string; username?: string }>(),
  notification: jsonb('notification').$type<Record<string, boolean>>(),
  role:         smallint('role').notNull().default(0),        // UserRoleEnum
  isDisable:    boolean('is_disable').notNull().default(false),
  createdAt:    timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:    timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
// Observer: On create → initializeUserDefaults() sets telegram.id (12-char random token)
//           and all 12 notification preference keys to true.
```

```typescript
// server/database/schema/boards.ts
export const boards = pgTable('boards', {
  id:          serial('id').primaryKey(),
  userId:      integer('user_id').notNull().references(() => users.id),
  name:        varchar('name', { length: 50 }).notNull(),
  description: text('description').notNull(),
  status:      smallint('status').notNull().default(0),       // BoardStatusEnum
  icon:        smallint('icon').notNull().default(0),          // BoardIconEnum
  createdAt:   timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:   timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
// Observer: After create  → broadcastToCentrifugo('board_create')
//           After update  → broadcastToCentrifugo('board_update')
//           After delete  → broadcastToCentrifugo('board_delete')
```

```typescript
// server/database/schema/boardUsers.ts
export const boardUsers = pgTable('board_user', {
  id:        serial('id').primaryKey(),
  userId:    integer('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  boardId:   integer('board_id').notNull().references(() => boards.id, { onDelete: 'cascade' }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
// Cascade: Deleting user or board removes all matching board_user rows.
```

```typescript
// server/database/schema/boardLists.ts
export const boardLists = pgTable('board_lists', {
  id:        serial('id').primaryKey(),
  boardId:   integer('board_id').notNull().references(() => boards.id, { onDelete: 'cascade' }),
  name:      varchar('name', { length: 50 }).notNull(),
  position:  integer('position').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
// Observer: After create → broadcastToCentrifugo('list_create', boardChannel)
//           After update → broadcastToCentrifugo('list_update', boardChannel)
//           After delete → broadcastToCentrifugo('list_delete', boardChannel)
```

```typescript
// server/database/schema/cards.ts
export const cards = pgTable('cards', {
  id:          serial('id').primaryKey(),
  boardListId: integer('board_list_id').references(() => boardLists.id, { onDelete: 'set null' }),
  name:        varchar('name', { length: 255 }).notNull(),
  description: text('description'),
  position:    smallint('position').notNull(),
  tags:        jsonb('tags').$type<number[]>().notNull().default([]),
  deletedAt:   timestamp('deleted_at', { withTimezone: true }),   // soft delete
  createdAt:   timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:   timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
// Soft delete: softDeleteCard()  → sets deletedAt = now()
//              restoreCard()      → sets deletedAt = null
// All read queries must apply: isNull(cards.deletedAt)
// Observer: After create      → createCardLog(type=0) + broadcastToCentrifugo('card_create')
//           After update      → createCardLog(type=0) + broadcastToCentrifugo('card_update')
//           After soft-delete → createCardLog(type=0) + broadcastToCentrifugo('card_delete')
//           Member assignment → broadcastToCentrifugo('card_add_users' | 'card_remove_users')
// Utility:  getCombinedActivity(cardId) → merged chronological array of comments + logs
```

```typescript
// server/database/schema/cardUsers.ts
export const cardUsers = pgTable('card_user', {
  id:        serial('id').primaryKey(),
  userId:    integer('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  cardId:    integer('card_id').notNull().references(() => cards.id, { onDelete: 'cascade' }),
  status:    smallint('status').notNull().default(0),    // CardUserStatusEnum
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
// status: None(0) | Pending(1) | Doing(2) | OnHold(3) | Done(4)
// Cascade: Deleting user or card removes all matching card_user rows.
```

```typescript
// server/database/schema/cardLogs.ts
export const cardLogs = pgTable('card_logs', {
  id:        serial('id').primaryKey(),
  userId:    integer('user_id').notNull().references(() => users.id),
  cardId:    integer('card_id').notNull().references(() => cards.id),
  boardId:   integer('board_id').notNull().references(() => boards.id),
  type:      smallint('type').notNull(),    // 0=card, 1=comment, 2=attachment
  data:      jsonb('data').$type<{
    message?: string
    changes?: Record<string, any>
    old_data?: Record<string, any>
  }>().notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
// Observer: After create → broadcastToCentrifugo('log_create', cardChannel)
```

```typescript
// server/database/schema/comments.ts
export const comments = pgTable('comments', {
  id:           serial('id').primaryKey(),
  userId:       integer('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  cardId:       integer('card_id').notNull().references(() => cards.id, { onDelete: 'cascade' }),
  attachmentId: integer('attachment_id').references(() => attachments.id),
  body:         text('body').notNull(),
  pin:          boolean('pin').notNull().default(false),
  createdAt:    timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:    timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
// Observer: After create → createCardLog(type=1) + broadcastToCentrifugo('comment_create')
//           After update → createCardLog(type=1) + broadcastToCentrifugo('comment_update')
//           After delete → createCardLog(type=1) + broadcastToCentrifugo('comment_delete')
```

```typescript
// server/database/schema/attachments.ts
export const attachments = pgTable('attachments', {
  id:        serial('id').primaryKey(),
  userId:    integer('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  cardId:    integer('card_id').notNull().references(() => cards.id, { onDelete: 'cascade' }),
  name:      varchar('name', { length: 50 }).notNull(),
  type:      smallint('type').notNull(),    // FileTypeEnum
  path:      varchar('path', { length: 255 }).notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
// type: Other(0) | Image(1) | Video(2) | Audio(3) | PDF(4) | Excel(5)
// Observer: After create → createCardLog(type=2) + broadcastToCentrifugo('attachment_create')
//           After update → broadcastToCentrifugo('attachment_update')
//           After delete → broadcastToCentrifugo('attachment_delete') + removeFileFromDisk(path)
```

```typescript
// server/database/schema/tags.ts
export const tags = pgTable('tags', {
  id:          serial('id').primaryKey(),
  name:        varchar('name', { length: 50 }).notNull().unique(),
  color:       smallint('color').notNull(),    // TagColorEnum
  description: varchar('description', { length: 255 }).notNull().default('default'),
  createdAt:   timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:   timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
// color: Red(0) | Purple(1) | Green(2) | Orange(3) | Blue(4) | Gray(5) | Cyan(6) | Magenta(7) | Violet(8)
// Card-tag associations stored in cards.tags as jsonb number[] — no junction table.
```

```typescript
// server/database/schema/notifications.ts
export const notifications = pgTable('notifications', {
  id:           uuid('id').primaryKey().defaultRandom(),
  type:         varchar('type', { length: 255 }).notNull(),
  notifiableId: integer('notifiable_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  data:         text('data').notNull(),    // JSON stringified payload
  readAt:       timestamp('read_at', { withTimezone: true }),
  createdAt:    timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:    timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
// type: notification class name string, e.g. 'CreateNewBoardNotification'
// data: JSON.stringify(payload) — stored as text, not jsonb
// readAt: null = unread
// Simplified from polymorphic to direct FK: notifiable_id → users.id
```

### Drizzle Relations (`server/database/schema/relations.ts`)

```typescript
export const usersRelations = relations(users, ({ many }) => ({
  boards:        many(boardUsers),          // boards the user is a member of
  cards:         many(cardUsers),           // cards assigned to the user
  logs:          many(cardLogs),
  comments:      many(comments),
  attachments:   many(attachments),
  notifications: many(notifications),
  createdBoards: many(boards),             // boards the user created
}))

export const boardsRelations = relations(boards, ({ one, many }) => ({
  creator:    one(users, { fields: [boards.userId], references: [users.id] }),
  users:      many(boardUsers),
  boardLists: many(boardLists),
  logs:       many(cardLogs),
}))

export const boardUsersRelations = relations(boardUsers, ({ one }) => ({
  user:  one(users,  { fields: [boardUsers.userId],  references: [users.id] }),
  board: one(boards, { fields: [boardUsers.boardId], references: [boards.id] }),
}))

export const boardListsRelations = relations(boardLists, ({ one, many }) => ({
  board: one(boards, { fields: [boardLists.boardId], references: [boards.id] }),
  cards: many(cards),
}))

export const cardsRelations = relations(cards, ({ one, many }) => ({
  list:        one(boardLists, { fields: [cards.boardListId], references: [boardLists.id] }),
  users:       many(cardUsers),
  logs:        many(cardLogs),
  comments:    many(comments),
  attachments: many(attachments),
}))

export const cardUsersRelations = relations(cardUsers, ({ one }) => ({
  user: one(users, { fields: [cardUsers.userId], references: [users.id] }),
  card: one(cards, { fields: [cardUsers.cardId], references: [cards.id] }),
}))

export const cardLogsRelations = relations(cardLogs, ({ one }) => ({
  user:  one(users,  { fields: [cardLogs.userId],  references: [users.id] }),
  card:  one(cards,  { fields: [cardLogs.cardId],  references: [cards.id] }),
  board: one(boards, { fields: [cardLogs.boardId], references: [boards.id] }),
}))

export const commentsRelations = relations(comments, ({ one }) => ({
  card:       one(cards,       { fields: [comments.cardId],       references: [cards.id] }),
  user:       one(users,       { fields: [comments.userId],       references: [users.id] }),
  attachment: one(attachments, { fields: [comments.attachmentId], references: [attachments.id] }),
}))

export const attachmentsRelations = relations(attachments, ({ one, many }) => ({
  card:     one(cards,  { fields: [attachments.cardId],  references: [cards.id] }),
  user:     one(users,  { fields: [attachments.userId],  references: [users.id] }),
  comments: many(comments),
}))

export const notificationsRelations = relations(notifications, ({ one }) => ({
  user: one(users, { fields: [notifications.notifiableId], references: [users.id] }),
}))
```

### TypeScript Model Interfaces (`shared/types/models.ts`)

```typescript
interface UserModel {
  id: number
  name: string
  username: string
  password: string
  avatar: string | null
  telegram: { id: string; chat_id?: string; username?: string } | null
  notification: Record<string, boolean> | null
  role: UserRole
  is_disable: boolean
}

interface BoardModel {
  id: number
  user_id: number
  name: string
  description: string
  status: BoardStatus
  icon: BoardIcon
  icon_name?: string
  members: BoardUserModel[]
  board_lists: BoardListModel[]
}

interface BoardListModel {
  id: number
  board_id: number
  name: string
  position: number
  cards: CardModel[]
}

interface CardModel {
  id: number
  board_list_id: number | null
  name: string
  description: string | null
  position: number
  tags: number[]
  users: CardUserModel[]
  attachments: AttachmentModel[]
  comments: CommentModel[]
}

interface CardUserModel {
  id: number
  user_id: number
  card_id: number
  status: CardUserStatus
}

interface CardLogModel {
  id: number
  user_id: number
  card_id: number
  board_id: number
  type: 0 | 1 | 2
  data: { message?: string; changes?: Record<string, unknown>; old_data?: Record<string, unknown> }
}

interface CommentModel {
  id: number
  user_id: number
  user_name: string
  user?: UserModel
  card_id: number
  user_avatar: string | null
  attachment_id: number | null
  attachment?: AttachmentModel | null
  body: string
  pin: boolean
}

interface AttachmentModel {
  id: number
  user_id: number
  card_id: number
  name: string
  type: FileType
  path: string
}

interface TagModel {
  id: number
  name: string
  color: { name: string; value: TagColor }
  description: string
}

interface NotificationModel {
  id: string            // uuid
  type: string
  notifiable: UserModel
  data: string          // JSON stringified
  read_at: string | null
}

interface BoardUserModel {
  id: number
  user_id: number
  board_id: number
}
```

### Entity Relationship Diagram

```
users ──────────────────────────────────────────────────────────────────────────┐
  │                                                                              │
  │ M:M (via board_user)                                                         │
  ├──board_user──── boards                                                       │
  │                   │                                                          │
  │                   │ 1:N                                                      │
  │                   └── board_lists                                            │
  │                             │                                                │
  │                             │ 1:N                                            │
  │                             └── cards ──M:M (via card_user)──────── users   │
  │                                   │                                          │
  │                                   │ 1:N                                      │
  │ 1:N ─────────────────────── card_logs (board_id FK →boards)                 │
  │                                   │                                          │
  │ 1:N ─────────────────────── comments ──N:1──── attachments                  │
  │                                   │                  │                       │
  │ 1:N ─────────────────────── attachments             1:N                     │
  │                                                   comments                  │
  │                                                                              │
  └── notifications (notifiable_id → users.id)                                  │
                                                                                 │
tags ── stored as jsonb number[] in cards.tags (no junction table)              │
                                                                                 │
cards.board_list_id → board_lists.id  (SET NULL on list delete)                 │
card_logs.board_id  → boards.id                                                 │
```

## Edge Cases

- **Soft Delete Visibility:** Hard-deleted cards (via cascade from a deleted list) are gone permanently. Only application-level soft-deletes set `deleted_at`. Queries must always apply `isNull(cards.deletedAt)` unless explicitly fetching deleted records.
- **Orphaned Cards:** When a `board_list` is deleted, `cards.board_list_id` is set to `NULL` (SET NULL). These cards become orphaned and are not visible in any list. A periodic cleanup or restore flow should handle them.
- **Tag References:** Tags are stored as a JSONB array of IDs in `cards.tags`. If a tag is deleted from the `tags` table, stale IDs will remain in `cards.tags`. Callers must handle missing tag IDs gracefully (e.g., filter out non-existent IDs when transforming).
- **Notification `data` as Text:** Unlike other JSONB columns, `notifications.data` is stored as `text` (JSON-stringified). Always `JSON.parse(data)` before use; always `JSON.stringify(payload)` before insert.
- **`telegram.id` Token:** The `telegram.id` field in the `users.telegram` jsonb is a random 12-character token generated at user creation — not the Telegram user ID. The actual Telegram user ID and chat ID are stored as `chat_id` after the user connects their Telegram account.
- **Duplicate `board_user` Rows:** There is no UNIQUE constraint on `(user_id, board_id)` in `board_user`. Application logic must prevent duplicate membership before inserting.
- **`card_user` Status Default:** New card members default to `status = 0` (None). Status changes are explicit updates, not automatic.
- **`updated_at` Maintenance:** Drizzle does not auto-update `updated_at` on row changes. Route handlers are responsible for setting `updatedAt: new Date()` on every UPDATE query.

## Non-Goals

- This spec does not cover the Drizzle migration workflow (`drizzle-kit generate` / `drizzle-kit migrate`) — see `infrastructure-setup.md`.
- This spec does not cover the database seeder (`server/database/seed.ts`) implementation details.
- This spec does not cover the Drizzle client instance configuration (`server/database/index.ts`) — see `infrastructure-setup.md`.
- This spec does not define Zod validation schemas — see `validation-schemas.md`.
- This spec does not define API route handlers that use these schemas — each domain has its own spec.
- This spec does not cover full-text search or PostgreSQL indexes beyond the constraints listed.
- This spec does not cover the notification preference key names (the 12 keys) — see `notification-system.md`.

## Dependencies

- **PostgreSQL 16** — target database engine
- **Drizzle ORM** (`drizzle-orm/pg-core`) — `pgTable`, `serial`, `integer`, `varchar`, `text`, `smallint`, `boolean`, `jsonb`, `uuid`, `timestamp`, `relations`
- **drizzle-kit** — migration generation (`drizzle-kit generate`, `drizzle-kit migrate`)
- **`infrastructure-setup.md`** — provides `DATABASE_URL`, Docker PostgreSQL service, `drizzle.config.ts`, and the Drizzle client instance
- **`notification-system.md`** — defines the 12 notification preference keys stored in `users.notification`
- **`realtime-centrifugo.md`** — provides the `broadcastToCentrifugo()` utility called by observer hooks
- **`audit-trail.md`** — provides `createCardLog()` and `getCombinedActivity()` called by observer hooks
