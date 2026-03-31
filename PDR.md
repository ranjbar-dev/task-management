# Product Design Review (PDR)

## Task Management Application — Complete Feature Catalog

### Nuxt 4 Full-Stack Edition

---

## 1. Project Overview

A full-stack Kanban-style task management application with real-time collaboration, Telegram bot integration, and an admin dashboard. The entire application — both API server and SPA client — runs as a single unified Nuxt 4 project, with Nitro handling server-side API routes and the Nuxt SPA serving the frontend.

### 1.1 Tech Stack

| Layer | Technology |
|-------|-----------|
| **Runtime** | Nuxt 4 (Nitro server + Vue 3 SPA client) |
| **Server Engine** | Nitro (built into Nuxt 4) — serves API routes + static SPA |
| **Language** | TypeScript (full-stack) |
| **Database** | PostgreSQL 16 |
| **ORM** | Drizzle ORM (with drizzle-kit for migrations) |
| **Validation** | Zod |
| **Authentication** | JWT via `jose` library (HS256) |
| **Real-Time** | Centrifugo WebSocket server |
| **Notifications** | Custom NotificationService (PostgreSQL + Telegram Bot API) |
| **Queue / Jobs** | BullMQ (Redis-backed) |
| **Cache** | Redis (shared with BullMQ) |
| **Frontend Framework** | Nuxt 4 (Vue 3, SPA mode — SSR disabled) |
| **State Management** | Pinia |
| **CSS Framework** | Tailwind CSS 4 (dark theme, custom violet/purple palette) |
| **Animations** | GSAP 3.13 (ScrollTrigger, ScrollToPlugin) |
| **Drag & Drop** | vue-draggable-plus v4 |
| **Icons** | vue-iconsax v2 + @nuxt/icon + Material Symbols |
| **Toasts** | vue-sonner |
| **Date Picker** | vue3-persian-datetime-picker |
| **Image Viewer** | nuxt-easy-lightbox / vue-easy-lightbox |
| **Docker** | Development + Production compose files |

### 1.2 Architecture Overview

```
┌────────────────────────────────────────────────────┐
│                  Nuxt 4 Application                │
│                                                    │
│  ┌──────────────────┐    ┌──────────────────────┐  │
│  │   SPA Client     │    │   Nitro Server       │  │
│  │   (app/)         │    │   (server/)          │  │
│  │                  │    │                      │  │
│  │  Vue 3 + Pinia   │◄──►  API Routes          │  │
│  │  Composables     │    │  Middleware          │  │
│  │  Components      │    │  Services            │  │
│  │  Pages/Layouts   │    │  Drizzle ORM         │  │
│  └──────────────────┘    └──────────┬───────────┘  │
│                                     │              │
│  ┌──────────────────┐               │              │
│  │  Shared (shared/)│               │              │
│  │  Types, Enums,   │               │              │
│  │  Zod Schemas     │               │              │
│  └──────────────────┘               │              │
└─────────────────────────────────────┼──────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────┐
          │                           │                       │
    ┌─────▼─────┐            ┌────────▼───────┐       ┌──────▼──────┐
    │ PostgreSQL │            │   Centrifugo   │       │    Redis    │
    │  Database  │            │  WebSocket     │       │  (Queue +   │
    │            │            │  Server        │       │   Cache)    │
    └────────────┘            └────────────────┘       └─────────────┘
```

### 1.3 Project Structure

```
task-management/
├── nuxt.config.ts                    # Nuxt 4 configuration
├── package.json
├── tsconfig.json
├── tailwind.config.js
├── drizzle.config.ts                 # Drizzle ORM configuration
├── docker/
│   ├── development/
│   └── production/
├── compose.dev.yaml
├── compose.prod.yaml
├── shared/                           # Shared between server & client
│   ├── enums/
│   │   ├── BoardIconEnum.ts
│   │   ├── BoardStatusEnum.ts
│   │   ├── CardUserStatusEnum.ts
│   │   ├── FileTypeEnum.ts
│   │   ├── TagColorEnum.ts
│   │   └── UserRoleEnum.ts
│   ├── schemas/                      # Zod validation schemas
│   │   ├── auth.schema.ts
│   │   ├── board.schema.ts
│   │   ├── boardList.schema.ts
│   │   ├── card.schema.ts
│   │   ├── comment.schema.ts
│   │   ├── attachment.schema.ts
│   │   ├── tag.schema.ts
│   │   ├── user.schema.ts
│   │   └── searchTable.schema.ts
│   └── types/                        # Shared TypeScript types
│       ├── api.ts                    # Response<T>, PaginatedResponse<T>
│       ├── models.ts                 # All model interfaces
│       └── enums.ts                  # Re-exports from enums/
├── server/                           # Nitro server (API backend)
│   ├── api/
│   │   ├── auth/
│   │   │   ├── login.post.ts
│   │   │   ├── logout.post.ts
│   │   │   ├── refresh.post.ts
│   │   │   └── centrifugo/
│   │   │       └── channel.post.ts
│   │   ├── admin/
│   │   │   ├── board/
│   │   │   │   ├── all.get.ts
│   │   │   │   ├── table.post.ts
│   │   │   │   ├── create.post.ts
│   │   │   │   ├── [id].put.ts
│   │   │   │   ├── [id].delete.ts
│   │   │   │   ├── overview/[id].get.ts
│   │   │   │   ├── logs-table/[id].post.ts
│   │   │   │   ├── members/[id].post.ts
│   │   │   │   └── icons.get.ts
│   │   │   ├── user/
│   │   │   │   ├── all.get.ts
│   │   │   │   ├── table.post.ts
│   │   │   │   ├── create.post.ts
│   │   │   │   ├── [id].put.ts
│   │   │   │   ├── [id].delete.ts
│   │   │   │   ├── [id].get.ts
│   │   │   │   └── update-password/[id].post.ts
│   │   │   ├── tag/
│   │   │   │   ├── all.get.ts
│   │   │   │   ├── table.post.ts
│   │   │   │   ├── create.post.ts
│   │   │   │   ├── [id].put.ts
│   │   │   │   ├── [id].delete.ts
│   │   │   │   └── [id].get.ts
│   │   │   └── notification/
│   │   │       ├── all.get.ts
│   │   │       └── table.post.ts
│   │   └── client/
│   │       ├── board/
│   │       │   ├── all.get.ts
│   │       │   ├── show/[id].get.ts
│   │       │   ├── overview/[id].get.ts
│   │       │   └── logs-table/[id].post.ts
│   │       ├── board-list/
│   │       │   ├── [boardId].get.ts
│   │       │   ├── show/[id].get.ts
│   │       │   ├── create/[boardId].post.ts
│   │       │   ├── [id].put.ts
│   │       │   ├── [id].delete.ts
│   │       │   └── update-position/[id].post.ts
│   │       ├── card/
│   │       │   ├── create/[boardListId].post.ts
│   │       │   ├── quick-create/[boardListId].post.ts
│   │       │   ├── duplicate/[id].post.ts
│   │       │   ├── show/[id].get.ts
│   │       │   ├── [id].put.ts
│   │       │   ├── [id].delete.ts
│   │       │   ├── update-position/[id].post.ts
│   │       │   ├── members/[id].post.ts
│   │       │   ├── [cardId]/delete-member/[userId].post.ts
│   │       │   ├── tags/[id].post.ts
│   │       │   └── update-status/[id].post.ts
│   │       ├── comment/
│   │       │   ├── create/[cardId].post.ts
│   │       │   ├── show/[id].get.ts
│   │       │   ├── [id].put.ts
│   │       │   └── [id].delete.ts
│   │       ├── attachment/
│   │       │   ├── file-types.get.ts
│   │       │   ├── create/[cardId].post.ts
│   │       │   ├── [id].put.ts
│   │       │   └── [id].delete.ts
│   │       ├── tag/
│   │       │   └── all.get.ts
│   │       ├── user/
│   │       │   ├── all.get.ts
│   │       │   ├── show/[id].get.ts
│   │       │   ├── [id].put.ts
│   │       │   ├── update-avatar/[id].post.ts
│   │       │   ├── delete-avatar/[id].post.ts
│   │       │   └── update-password/[id].post.ts
│   │       └── telegram/
│   │           ├── connect.get.ts
│   │           └── disconnect.post.ts
│   ├── middleware/
│   │   ├── auth.ts                   # JWT validation, loads user into event.context
│   │   ├── admin.ts                  # Role check — requires AdminUser
│   │   └── active-user.ts            # Blocks disabled users (is_disable)
│   ├── database/
│   │   ├── schema/
│   │   │   ├── users.ts
│   │   │   ├── boards.ts
│   │   │   ├── boardLists.ts
│   │   │   ├── cards.ts
│   │   │   ├── cardUsers.ts
│   │   │   ├── cardLogs.ts
│   │   │   ├── comments.ts
│   │   │   ├── attachments.ts
│   │   │   ├── tags.ts
│   │   │   ├── notifications.ts
│   │   │   ├── boardUsers.ts         # Junction table
│   │   │   ├── relations.ts          # All Drizzle relation definitions
│   │   │   └── index.ts              # Re-exports all schemas
│   │   ├── migrations/               # Generated by drizzle-kit
│   │   ├── seed.ts                   # Database seeder
│   │   └── index.ts                  # Drizzle client instance (db)
│   ├── services/
│   │   ├── searchTable.ts            # Reusable paginated table search engine
│   │   ├── centrifugo.ts             # Centrifugo HTTP API client
│   │   ├── notification.ts           # NotificationService — database + Telegram
│   │   ├── telegram.ts               # Telegram Bot API HTTP client
│   │   └── fileStorage.ts            # File upload/delete/path utilities
│   ├── jobs/
│   │   ├── queue.ts                  # BullMQ queue + worker setup
│   │   ├── notification.job.ts       # Async notification dispatch job
│   │   └── telegramRegister.job.ts   # Telegram bot registration polling job
│   ├── transformers/                 # Response transformers (replaces Laravel Resources)
│   │   ├── user.transformer.ts
│   │   ├── board.transformer.ts
│   │   ├── boardList.transformer.ts
│   │   ├── card.transformer.ts
│   │   ├── comment.transformer.ts
│   │   ├── attachment.transformer.ts
│   │   ├── tag.transformer.ts
│   │   ├── cardLog.transformer.ts
│   │   └── notification.transformer.ts
│   ├── utils/
│   │   ├── jwt.ts                    # JWT sign/verify/refresh via jose
│   │   ├── password.ts               # bcrypt hash/verify
│   │   ├── response.ts               # Standard API response wrapper
│   │   ├── cardLog.ts                # CardLog creation utility
│   │   └── broadcast.ts              # Centrifugo broadcast helper
│   └── plugins/
│       └── bullmq.ts                 # BullMQ worker initialization on server start
├── app/                              # Nuxt SPA client (unchanged from original)
│   ├── app.vue
│   ├── error.vue
│   ├── assets/
│   ├── components/
│   ├── composables/
│   ├── layouts/
│   ├── middleware/
│   ├── pages/
│   ├── plugins/
│   ├── stores/
│   ├── types/
│   └── utils/
└── public/
    ├── robots.txt
    └── images/
```

### 1.4 Migration Map — Laravel → Nuxt 4

| Laravel Concept | Nuxt 4 Equivalent | Location |
|----------------|-------------------|----------|
| Eloquent Model | Drizzle `pgTable()` schema | `server/database/schema/` |
| Eloquent Relationships | Drizzle `relations()` | `server/database/schema/relations.ts` |
| Eloquent Observer | Service utility functions called in route handlers | `server/utils/`, `server/services/` |
| Migration | `drizzle-kit generate` + `drizzle-kit migrate` | `server/database/migrations/` |
| FormRequest | Zod schema + `zodParse()` helper | `shared/schemas/` |
| API Resource | Transformer function | `server/transformers/` |
| Controller | Nitro route handler (`defineEventHandler`) | `server/api/` |
| Middleware | Nitro server middleware | `server/middleware/` |
| Route (web/api) | File-based routing (`[method].ts`) | `server/api/` |
| Event + Listener | Direct function call / BullMQ job | `server/services/`, `server/jobs/` |
| Notification | `NotificationService.send()` | `server/services/notification.ts` |
| Job (Queue) | BullMQ Job | `server/jobs/` |
| Config (`config/`) | `runtimeConfig` in `nuxt.config.ts` + `.env` | `nuxt.config.ts` |
| Service Provider | Nitro plugin | `server/plugins/` |
| Broadcasting | `broadcastToCentrifugo()` utility | `server/utils/broadcast.ts` |
| Seeder | `server/database/seed.ts` script | `server/database/seed.ts` |
| Storage / Filesystem | `server/services/fileStorage.ts` | `server/services/fileStorage.ts` |
| CORS config | Nitro `routeRules` | `nuxt.config.ts` |

---

## 2. Data Model & Entity Relationships

### 2.1 Drizzle Schemas

All schemas use `drizzle-orm/pg-core`. Primary keys use `serial` (auto-increment integer) for simplicity. JSON fields use PostgreSQL `jsonb` for better indexing and query support.

#### Users

```typescript
// server/database/schema/users.ts
import { pgTable, serial, varchar, boolean, smallint, jsonb, timestamp } from 'drizzle-orm/pg-core'

export const users = pgTable('users', {
  id:           serial('id').primaryKey(),
  name:         varchar('name', { length: 50 }).notNull(),
  username:     varchar('username', { length: 50 }).notNull().unique(),
  password:     varchar('password', { length: 255 }).notNull(),
  avatar:       varchar('avatar', { length: 255 }),
  telegram:     jsonb('telegram').$type<{ id: string; chat_id?: string; username?: string }>(),
  notification: jsonb('notification').$type<Record<string, boolean>>(),
  role:         smallint('role').notNull().default(0),     // UserRoleEnum
  isDisable:    boolean('is_disable').notNull().default(false),
  createdAt:    timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:    timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

**Hooks (replaces UserObserver):**
- On user creation → initialize `telegram.id` (random 12-char token) and all 12 notification preferences to `true`
- Implemented as a utility function `initializeUserDefaults()` called in the create route handler

---

#### Boards

```typescript
// server/database/schema/boards.ts
import { pgTable, serial, varchar, text, smallint, integer, timestamp } from 'drizzle-orm/pg-core'

export const boards = pgTable('boards', {
  id:          serial('id').primaryKey(),
  userId:      integer('user_id').notNull().references(() => users.id),
  name:        varchar('name', { length: 50 }).notNull(),
  description: text('description').notNull(),
  status:      smallint('status').notNull().default(0),   // BoardStatusEnum
  icon:        smallint('icon').notNull().default(0),      // BoardIconEnum
  createdAt:   timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:   timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

**Hooks (replaces BoardObserver):**
- After create/update/delete → call `broadcastToCentrifugo()` with `board_create`, `board_update`, `board_delete`
- Called inline in route handlers

---

#### Board Users (Junction Table)

```typescript
// server/database/schema/boardUsers.ts
import { pgTable, serial, integer, timestamp } from 'drizzle-orm/pg-core'

export const boardUsers = pgTable('board_user', {
  id:        serial('id').primaryKey(),
  userId:    integer('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  boardId:   integer('board_id').notNull().references(() => boards.id, { onDelete: 'cascade' }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

---

#### Board Lists

```typescript
// server/database/schema/boardLists.ts
import { pgTable, serial, integer, varchar, timestamp } from 'drizzle-orm/pg-core'

export const boardLists = pgTable('board_lists', {
  id:        serial('id').primaryKey(),
  boardId:   integer('board_id').notNull().references(() => boards.id, { onDelete: 'cascade' }),
  name:      varchar('name', { length: 50 }).notNull(),
  position:  integer('position').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

**Hooks (replaces BoardListObserver):**
- After create/update/delete → `broadcastToCentrifugo()` with `list_create`, `list_update`, `list_delete` to board channel

---

#### Cards

```typescript
// server/database/schema/cards.ts
import { pgTable, serial, integer, varchar, text, smallint, jsonb, timestamp } from 'drizzle-orm/pg-core'

export const cards = pgTable('cards', {
  id:          serial('id').primaryKey(),
  boardListId: integer('board_list_id').references(() => boardLists.id, { onDelete: 'set null' }),
  name:        varchar('name', { length: 255 }).notNull(),
  description: text('description'),
  position:    smallint('position').notNull(),
  tags:        jsonb('tags').$type<number[]>().notNull().default([]),
  deletedAt:   timestamp('deleted_at', { withTimezone: true }),    // soft delete
  createdAt:   timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:   timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

**Hooks (replaces CardObserver):**
- After create → create `CardLog` (type=0), broadcast `card_create` to board channel
- After update → create `CardLog`, broadcast `card_update` to card channel
- After delete (soft) → create `CardLog`, broadcast `card_delete` to board channel
- Broadcasts for user sync: `card_add_users`, `card_remove_users`

**Soft Deletes:**
- Implemented via `deletedAt` column — all card queries filter `WHERE deleted_at IS NULL` using a Drizzle `isNull()` condition
- Utility function `softDeleteCard()` sets `deletedAt = now()` instead of removing the row
- Restore capability via `restoreCard()` sets `deletedAt = null`

**Combined Activity:**
- `getCombinedActivity(cardId)` utility merges comments + logs into a single chronologically sorted array

---

#### Card Users (Pivot)

```typescript
// server/database/schema/cardUsers.ts
import { pgTable, serial, integer, smallint, timestamp } from 'drizzle-orm/pg-core'

export const cardUsers = pgTable('card_user', {
  id:        serial('id').primaryKey(),
  userId:    integer('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  cardId:    integer('card_id').notNull().references(() => cards.id, { onDelete: 'cascade' }),
  status:    smallint('status').notNull().default(0),    // CardUserStatusEnum
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

**Status Values:** None (0), Pending (1), Doing (2), OnHold (3), Done (4)

---

#### Card Logs

```typescript
// server/database/schema/cardLogs.ts
import { pgTable, serial, integer, smallint, jsonb, timestamp } from 'drizzle-orm/pg-core'

export const cardLogs = pgTable('card_logs', {
  id:        serial('id').primaryKey(),
  userId:    integer('user_id').notNull().references(() => users.id),
  cardId:    integer('card_id').notNull().references(() => cards.id),
  boardId:   integer('board_id').notNull().references(() => boards.id),
  type:      smallint('type').notNull(),    // 0=card, 1=comment, 2=attachment
  data:      jsonb('data').$type<{ message?: string; changes?: Record<string, any>; old_data?: Record<string, any> }>().notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

**Hooks (replaces CardLogObserver):**
- After creation → broadcast `log_create` to card channel

---

#### Comments

```typescript
// server/database/schema/comments.ts
import { pgTable, serial, integer, text, boolean, timestamp } from 'drizzle-orm/pg-core'

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
```

**Hooks (replaces CommentObserver):**
- After create → create `CardLog` (type=1), broadcast `comment_create`
- After update → create `CardLog`, broadcast `comment_update`
- After delete → create `CardLog`, broadcast `comment_delete`

---

#### Attachments

```typescript
// server/database/schema/attachments.ts
import { pgTable, serial, integer, varchar, smallint, timestamp } from 'drizzle-orm/pg-core'

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
```

**File Types:** Other (0), Image (1), Video (2), Audio (3), PDF (4), Excel (5)

**Hooks (replaces AttachmentObserver):**
- After create → create `CardLog` (type=2), broadcast `attachment_create`
- After update → create `CardLog`, broadcast `attachment_update`
- After delete → create `CardLog`, broadcast `attachment_delete`; remove file from disk

---

#### Tags

```typescript
// server/database/schema/tags.ts
import { pgTable, serial, varchar, smallint, timestamp } from 'drizzle-orm/pg-core'

export const tags = pgTable('tags', {
  id:          serial('id').primaryKey(),
  name:        varchar('name', { length: 50 }).notNull().unique(),
  color:       smallint('color').notNull(),    // TagColorEnum
  description: varchar('description', { length: 255 }).notNull().default('default'),
  createdAt:   timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:   timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

**Colors:** Red (0), Purple (1), Green (2), Orange (3), Blue (4), Gray (5), Cyan (6), Magenta (7), Violet (8)

---

#### Notifications

```typescript
// server/database/schema/notifications.ts
import { pgTable, uuid, varchar, integer, text, timestamp } from 'drizzle-orm/pg-core'

export const notifications = pgTable('notifications', {
  id:             uuid('id').primaryKey().defaultRandom(),
  type:           varchar('type', { length: 255 }).notNull(),
  notifiableId:   integer('notifiable_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  data:           text('data').notNull(),             // JSON stringified payload
  readAt:         timestamp('read_at', { withTimezone: true }),
  createdAt:      timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:      timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

> **Note:** The original Laravel schema used polymorphic `notifiable_type` + `notifiable_id`. Since notifications always target users in this application, the schema is simplified to a direct `notifiable_id → users.id` FK. The `type` field stores the notification class name string (e.g., `'CreateNewBoardNotification'`).

---

### 2.2 Drizzle Relations

```typescript
// server/database/schema/relations.ts
import { relations } from 'drizzle-orm'

export const usersRelations = relations(users, ({ many }) => ({
  boards:          many(boardUsers),
  cards:           many(cardUsers),
  logs:            many(cardLogs),
  comments:        many(comments),
  attachments:     many(attachments),
  notifications:   many(notifications),
  createdBoards:   many(boards),
}))

export const boardsRelations = relations(boards, ({ one, many }) => ({
  creator:    one(users, { fields: [boards.userId], references: [users.id] }),
  users:      many(boardUsers),
  boardLists: many(boardLists),
  logs:       many(cardLogs),
}))

export const boardUsersRelations = relations(boardUsers, ({ one }) => ({
  user:  one(users, { fields: [boardUsers.userId], references: [users.id] }),
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
  user:  one(users, { fields: [cardLogs.userId], references: [users.id] }),
  card:  one(cards, { fields: [cardLogs.cardId], references: [cards.id] }),
  board: one(boards, { fields: [cardLogs.boardId], references: [boards.id] }),
}))

export const commentsRelations = relations(comments, ({ one }) => ({
  card:       one(cards, { fields: [comments.cardId], references: [cards.id] }),
  user:       one(users, { fields: [comments.userId], references: [users.id] }),
  attachment: one(attachments, { fields: [comments.attachmentId], references: [attachments.id] }),
}))

export const attachmentsRelations = relations(attachments, ({ one, many }) => ({
  card:     one(cards, { fields: [attachments.cardId], references: [cards.id] }),
  user:     one(users, { fields: [attachments.userId], references: [users.id] }),
  comments: many(comments),
}))

export const notificationsRelations = relations(notifications, ({ one }) => ({
  user: one(users, { fields: [notifications.notifiableId], references: [users.id] }),
}))
```

---

### 2.3 Enums (TypeScript)

All enums are defined in `shared/enums/` and shared between server and client.

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

// Maps icon enum value → Material Symbols icon name
export const BoardIconNameMap: Record<BoardIcon, string> = {
  0: 'bolt', 1: 'inventory_2', 2: 'brush', 3: 'photo_camera',
  4: 'crown', 5: 'database', 6: 'euro', 7: 'folder',
  8: 'home', 9: 'map', 10: 'smartphone', 11: 'rocket_launch',
  12: 'star', 13: 'store', 14: 'sell',
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

---

### 2.4 Entity Relationship Diagram

```
User ──M:M──board_user──M:M── Board
 │                              │
 │ 1:N                          │ 1:N
 ├─ CardLog                     ├─ BoardList
 ├─ Comment                     │    │ 1:N
 └─ Attachment                  │    └─ Card ──M:M──card_user──M:M── User
                                │         │
                                │         ├─ 1:N CardLog
                                │         ├─ 1:N Comment ──N:1── Attachment
                                │         └─ 1:N Attachment
                                │
                                └─ 1:N CardLog

Tag ── stored as JSONB array of tag IDs in cards.tags
Notification ── FK notifiable_id → users.id
```

---

## 3. Authentication & Authorization

### 3.1 JWT Authentication

| Endpoint | Method | Nitro Route File | Description |
|----------|--------|------------------|-------------|
| `/api/auth/login` | POST | `server/api/auth/login.post.ts` | Authenticates user, returns JWT token + user data |
| `/api/auth/logout` | POST | `server/api/auth/logout.post.ts` | Invalidates JWT token (blacklist in Redis) |
| `/api/auth/refresh` | POST | `server/api/auth/refresh.post.ts` | Issues new JWT token |

**JWT Implementation (`server/utils/jwt.ts`):**

```typescript
import { SignJWT, jwtVerify } from 'jose'

// Configuration (from runtimeConfig)
// Algorithm: HS256 (symmetric)
// TTL: Effectively never expires (very large value)
// Refresh TTL: 2 weeks (20160 minutes)
// Secret: JWT_SECRET environment variable

async function signToken(userId: number): Promise<string>
async function verifyToken(token: string): Promise<JWTPayload>
async function refreshToken(oldToken: string): Promise<string>
```

**Token Blacklisting:**
- Blacklisted tokens stored in Redis with TTL matching token expiry
- `blacklistToken(jti)` adds token ID to Redis set
- `isBlacklisted(jti)` checks before validating
- Enables logout/revocation support

**Frontend Auth Flow (unchanged):**
1. Login → store token + user in localStorage + Pinia `authStore`
2. On app load → plugin `0.auth.client.ts` restores token from localStorage, calls `refreshToken()` + `fetchUser()`
3. Route middleware `auth.ts` checks `authStore.isLoggedIn` — redirects to `/auth/log-in` if unauthenticated
4. API client (`useApi`) attaches `Authorization: Bearer {token}` to all requests
5. 401 responses → auto-logout + redirect to login

### 3.2 Server Middleware

All three server middleware are in `server/middleware/` and applied via Nitro's middleware system.

| Middleware | File | Applied To | Logic |
|-----------|------|-----------|-------|
| **auth** | `server/middleware/auth.ts` | All `/api/**` routes except `/api/auth/login` | Extracts Bearer token, verifies via `jose`, loads user from DB into `event.context.user` |
| **active-user** | `server/middleware/active-user.ts` | All authenticated routes except `/api/auth/**` | Returns 403 if `event.context.user.isDisable === true` |
| **admin** | `server/middleware/admin.ts` | All `/api/admin/**` routes | Returns 401 if `event.context.user.role !== UserRoleEnum.AdminUser` |

**Middleware Application Pattern:**

```typescript
// server/middleware/auth.ts
export default defineEventHandler(async (event) => {
  // Skip auth routes
  if (getRequestURL(event).pathname.startsWith('/api/auth/login')) return
  if (!getRequestURL(event).pathname.startsWith('/api/')) return

  const token = getHeader(event, 'authorization')?.replace('Bearer ', '')
  if (!token) throw createError({ statusCode: 401, message: 'Unauthorized' })

  const payload = await verifyToken(token)
  const user = await db.query.users.findFirst({ where: eq(users.id, payload.sub) })
  if (!user) throw createError({ statusCode: 401, message: 'Unauthorized' })

  event.context.user = user
  event.context.token = token
})
```

### 3.3 Permission Checks

- **Board access:** Route handlers check `boardUsers` junction table: `db.query.boardUsers.findFirst({ where: and(eq(boardUsers.userId, user.id), eq(boardUsers.boardId, boardId)) })` or `user.role === UserRoleEnum.AdminUser`
- **Admin override:** Admins bypass board membership checks
- **Centrifugo channel auth:** `POST /api/auth/centrifugo/channel` validates user is board member or admin before authorizing WebSocket channel subscription

---

## 4. Admin Features

All admin routes are under `server/api/admin/` and require both `admin` and `active-user` middleware (applied globally via the middleware chain).

### 4.1 Board Management

| Endpoint | Method | Nitro Route File | Description |
|----------|--------|------------------|-------------|
| `/api/admin/board/all` | GET | `server/api/admin/board/all.get.ts` | List all boards with user count |
| `/api/admin/board/table` | POST | `server/api/admin/board/table.post.ts` | Paginated board table (SearchTable) |
| `/api/admin/board/create` | POST | `server/api/admin/board/create.post.ts` | Create board (name, description, status, icon) |
| `/api/admin/board/[id]` | PUT | `server/api/admin/board/[id].put.ts` | Update board fields |
| `/api/admin/board/[id]` | DELETE | `server/api/admin/board/[id].delete.ts` | Delete board — triggers `DeleteBoardNotification` to all members |
| `/api/admin/board/overview/[id]` | GET | `server/api/admin/board/overview/[id].get.ts` | Full board with lists, cards, and users |
| `/api/admin/board/logs-table/[id]` | POST | `server/api/admin/board/logs-table/[id].post.ts` | Board activity log table |
| `/api/admin/board/members/[id]` | POST | `server/api/admin/board/members/[id].post.ts` | Sync board members (array of user_ids) |
| `/api/admin/board/icons` | GET | `server/api/admin/board/icons.get.ts` | List available board icons |

### 4.2 User Management

| Endpoint | Method | Nitro Route File | Description |
|----------|--------|------------------|-------------|
| `/api/admin/user/all` | GET | `server/api/admin/user/all.get.ts` | List all users |
| `/api/admin/user/table` | POST | `server/api/admin/user/table.post.ts` | Paginated user table (SearchTable) |
| `/api/admin/user/create` | POST | `server/api/admin/user/create.post.ts` | Create user (name, username, password, role, is_disable) |
| `/api/admin/user/[id]` | PUT | `server/api/admin/user/[id].put.ts` | Update user — auto-creates private board if needed |
| `/api/admin/user/update-password/[id]` | POST | `server/api/admin/user/update-password/[id].post.ts` | Change user password |
| `/api/admin/user/[id]` | DELETE | `server/api/admin/user/[id].delete.ts` | Delete user |
| `/api/admin/user/[id]` | GET | `server/api/admin/user/[id].get.ts` | Get user details |

### 4.3 Tag Management

| Endpoint | Method | Nitro Route File | Description |
|----------|--------|------------------|-------------|
| `/api/admin/tag/all` | GET | `server/api/admin/tag/all.get.ts` | List all tags |
| `/api/admin/tag/table` | POST | `server/api/admin/tag/table.post.ts` | Paginated tag table (SearchTable) |
| `/api/admin/tag/create` | POST | `server/api/admin/tag/create.post.ts` | Create tag (name, color, description) |
| `/api/admin/tag/[id]` | PUT | `server/api/admin/tag/[id].put.ts` | Update tag |
| `/api/admin/tag/[id]` | DELETE | `server/api/admin/tag/[id].delete.ts` | Delete tag |
| `/api/admin/tag/[id]` | GET | `server/api/admin/tag/[id].get.ts` | Get tag details |

### 4.4 Notification Logs

| Endpoint | Method | Nitro Route File | Description |
|----------|--------|------------------|-------------|
| `/api/admin/notification/all` | GET | `server/api/admin/notification/all.get.ts` | List all notifications |
| `/api/admin/notification/table` | POST | `server/api/admin/notification/table.post.ts` | Paginated notification table (ordered by created_at desc) |

### 4.5 SearchTable Service

A reusable data table engine (`server/services/searchTable.ts`) used by all admin `table` endpoints. Port of the Laravel SearchTable service to TypeScript.

```typescript
// server/services/searchTable.ts
interface SearchTableOptions {
  query: DrizzleSelectQuery
  body: {
    search?: Array<{ key: string; type: FilterType; value: any }>
    sort?: { key: string; direction: 'asc' | 'desc' }
    per_page?: number
    page?: number
    all?: boolean
  }
}

type FilterType = 'like' | 'equal' | 'date' | 'datetime-bool' | 'in' | 'relation-equal' | 'relation-like' | 'bool'

async function handleSearchTable(options: SearchTableOptions): Promise<PaginatedResponse | FullResponse>
```

Supports:
- **Filter types:** `like`, `equal`, `date`, `datetime-bool`, `in`, `relation-equal`, `relation-like`, `bool`
- **Float conversion** for amount/price fields
- **Sort** by any column with direction
- **Pagination** with configurable page size
- Returns either paginated results or full collection

---

## 5. Client / Kanban Features

All client routes are under `server/api/client/` and require `active-user` middleware.

### 5.1 Board Access

| Endpoint | Method | Nitro Route File | Description |
|----------|--------|------------------|-------------|
| `/api/client/board/all` | GET | `server/api/client/board/all.get.ts` | List boards the user is a member of (id, name, status, icon) |
| `/api/client/board/show/[id]` | GET | `server/api/client/board/show/[id].get.ts` | Full board data (checks membership or admin) |
| `/api/client/board/overview/[id]` | GET | `server/api/client/board/overview/[id].get.ts` | Complete board state: users, lists, cards, card.users |
| `/api/client/board/logs-table/[id]` | POST | `server/api/client/board/logs-table/[id].post.ts` | Board activity logs |

### 5.2 BoardList Management

| Endpoint | Method | Nitro Route File | Description |
|----------|--------|------------------|-------------|
| `/api/client/board-list/[boardId]` | GET | `server/api/client/board-list/[boardId].get.ts` | Get all lists in a board (checks membership) |
| `/api/client/board-list/show/[id]` | GET | `server/api/client/board-list/show/[id].get.ts` | Single list with cards and card users |
| `/api/client/board-list/create/[boardId]` | POST | `server/api/client/board-list/create/[boardId].post.ts` | Create list (name, position) |
| `/api/client/board-list/[id]` | PUT | `server/api/client/board-list/[id].put.ts` | Update list (name, position) |
| `/api/client/board-list/[id]` | DELETE | `server/api/client/board-list/[id].delete.ts` | Delete list |
| `/api/client/board-list/update-position/[id]` | POST | `server/api/client/board-list/update-position/[id].post.ts` | Batch reorder lists (array of {id, position}) |

### 5.3 Card Management

| Endpoint | Method | Nitro Route File | Description |
|----------|--------|------------------|-------------|
| `/api/client/card/create/[boardListId]` | POST | `server/api/client/card/create/[boardListId].post.ts` | Create card with optional users, tags, file attachment |
| `/api/client/card/quick-create/[boardListId]` | POST | `server/api/client/card/quick-create/[boardListId].post.ts` | Quick create — name only, auto-assigns current user |
| `/api/client/card/duplicate/[id]` | POST | `server/api/client/card/duplicate/[id].post.ts` | Clone card to same list |
| `/api/client/card/show/[id]` | GET | `server/api/client/card/show/[id].get.ts` | Full card with users, comments, attachments |
| `/api/client/card/[id]` | PUT | `server/api/client/card/[id].put.ts` | Update card fields (name, description, board_list_id, position) |
| `/api/client/card/[id]` | DELETE | `server/api/client/card/[id].delete.ts` | Delete card — triggers `DeleteCardNotification` |
| `/api/client/card/update-position/[id]` | POST | `server/api/client/card/update-position/[id].post.ts` | Batch reorder cards (array of {id, position} with target board_list_id) |
| `/api/client/card/members/[id]` | POST | `server/api/client/card/members/[id].post.ts` | Sync card members (user_ids array) |
| `/api/client/card/[cardId]/delete-member/[userId]` | POST | `server/api/client/card/[cardId]/delete-member/[userId].post.ts` | Remove single member from card |
| `/api/client/card/tags/[id]` | POST | `server/api/client/card/tags/[id].post.ts` | Sync card tags (tag_ids array) |
| `/api/client/card/update-status/[id]` | POST | `server/api/client/card/update-status/[id].post.ts` | Update a user's status on a card (user_id, status enum) |

### 5.4 Comments

| Endpoint | Method | Nitro Route File | Description |
|----------|--------|------------------|-------------|
| `/api/client/comment/create/[cardId]` | POST | `server/api/client/comment/create/[cardId].post.ts` | Create comment — body required; optional attachment_id or file upload |
| `/api/client/comment/show/[id]` | GET | `server/api/client/comment/show/[id].get.ts` | Single comment with user and attachment |
| `/api/client/comment/[id]` | PUT | `server/api/client/comment/[id].put.ts` | Update comment body or pin status |
| `/api/client/comment/[id]` | DELETE | `server/api/client/comment/[id].delete.ts` | Delete comment |

### 5.5 Attachments

| Endpoint | Method | Nitro Route File | Description |
|----------|--------|------------------|-------------|
| `/api/client/attachment/file-types` | GET | `server/api/client/attachment/file-types.get.ts` | List file type enum values |
| `/api/client/attachment/create/[cardId]` | POST | `server/api/client/attachment/create/[cardId].post.ts` | Upload file attachment (name, file, type) |
| `/api/client/attachment/[id]` | PUT | `server/api/client/attachment/[id].put.ts` | Update attachment |
| `/api/client/attachment/[id]` | DELETE | `server/api/client/attachment/[id].delete.ts` | Delete attachment (removes from storage + DB) |

### 5.6 Tags (Read-only)

| Endpoint | Method | Nitro Route File | Description |
|----------|--------|------------------|-------------|
| `/api/client/tag/all` | GET | `server/api/client/tag/all.get.ts` | List all available tags |

### 5.7 User Profile

| Endpoint | Method | Nitro Route File | Description |
|----------|--------|------------------|-------------|
| `/api/client/user/all` | GET | `server/api/client/user/all.get.ts` | List all users in system |
| `/api/client/user/show/[id]` | GET | `server/api/client/user/show/[id].get.ts` | Get user details |
| `/api/client/user/[id]` | PUT | `server/api/client/user/[id].put.ts` | Update profile (name, username) |
| `/api/client/user/update-avatar/[id]` | POST | `server/api/client/user/update-avatar/[id].post.ts` | Upload avatar image |
| `/api/client/user/delete-avatar/[id]` | POST | `server/api/client/user/delete-avatar/[id].post.ts` | Remove avatar |
| `/api/client/user/update-password/[id]` | POST | `server/api/client/user/update-password/[id].post.ts` | Change password |

---

## 6. Validation Schemas (Zod)

All validation schemas are in `shared/schemas/` and shared between server and client. Enum string-to-integer conversion is handled within the Zod schemas using `.transform()`.

### 6.1 Auth Schemas

```typescript
// shared/schemas/auth.schema.ts
import { z } from 'zod'

export const loginSchema = z.object({
  username: z.string().min(1),
  password: z.string().min(8),
})

export const centrifugoChannelSchema = z.object({
  channel: z.string().min(1),
})
```

### 6.2 Board Schemas

```typescript
// shared/schemas/board.schema.ts
import { z } from 'zod'

export const createBoardSchema = z.object({
  name:        z.string().min(1).max(50),
  description: z.string().min(1),
  status:      z.coerce.number().int().min(0).max(1).optional(),
  icon:        z.coerce.number().int().min(0).max(14).optional(),
})

export const updateBoardSchema = createBoardSchema.partial()

export const syncBoardMembersSchema = z.object({
  user_ids: z.array(z.number().int()),
})
```

### 6.3 BoardList Schemas

```typescript
// shared/schemas/boardList.schema.ts
import { z } from 'zod'

export const createBoardListSchema = z.object({
  name:     z.string().min(1).max(50),
  position: z.number().int(),
})

export const updateBoardListSchema = createBoardListSchema.partial()

export const updatePositionSchema = z.object({
  items: z.array(z.object({
    id:       z.number().int(),
    position: z.number().int(),
  })),
})
```

### 6.4 Card Schemas

```typescript
// shared/schemas/card.schema.ts
import { z } from 'zod'

export const createCardSchema = z.object({
  name:          z.string().min(1).max(255),
  description:   z.string().optional(),
  position:      z.number().int(),
  user_ids:      z.array(z.number().int()).optional(),
  tag_ids:       z.array(z.number().int()).optional(),
})

export const quickCreateCardSchema = z.object({
  name: z.string().min(1).max(255),
})

export const updateCardSchema = z.object({
  name:          z.string().min(1).max(255).optional(),
  description:   z.string().optional(),
  board_list_id: z.number().int().optional(),
  position:      z.number().int().optional(),
})

export const syncCardMembersSchema = z.object({
  user_ids: z.array(z.number().int()),
})

export const syncCardTagsSchema = z.object({
  tag_ids: z.array(z.number().int()),
})

export const updateCardStatusSchema = z.object({
  user_id: z.number().int(),
  status:  z.coerce.number().int().min(0).max(4),
})

export const updateCardPositionSchema = z.object({
  board_list_id: z.number().int(),
  items: z.array(z.object({
    id:       z.number().int(),
    position: z.number().int(),
  })),
})
```

### 6.5 Comment Schemas

```typescript
// shared/schemas/comment.schema.ts
import { z } from 'zod'

export const createCommentSchema = z.object({
  body:          z.string().min(1),
  attachment_id: z.number().int().optional(),
})

export const updateCommentSchema = z.object({
  body: z.string().min(1).optional(),
  pin:  z.boolean().optional(),
})
```

### 6.6 Attachment Schemas

```typescript
// shared/schemas/attachment.schema.ts
import { z } from 'zod'

export const createAttachmentSchema = z.object({
  name: z.string().min(1).max(50),
  type: z.coerce.number().int().min(0).max(5),
})

export const updateAttachmentSchema = createAttachmentSchema.partial()
```

### 6.7 Tag Schemas

```typescript
// shared/schemas/tag.schema.ts
import { z } from 'zod'

export const createTagSchema = z.object({
  name:        z.string().min(1).max(50),
  color:       z.coerce.number().int().min(0).max(8),
  description: z.string().max(255).optional(),
})

export const updateTagSchema = createTagSchema.partial()
```

### 6.8 User Schemas

```typescript
// shared/schemas/user.schema.ts
import { z } from 'zod'

export const createUserSchema = z.object({
  name:       z.string().min(1).max(50),
  username:   z.string().min(1).max(50),
  password:   z.string().min(8),
  role:       z.coerce.number().int().min(0).max(1).optional(),
  is_disable: z.coerce.boolean().optional(),
})

export const updateUserSchema = z.object({
  name:       z.string().min(1).max(50).optional(),
  username:   z.string().min(1).max(50).optional(),
  role:       z.coerce.number().int().min(0).max(1).optional(),
  is_disable: z.coerce.boolean().optional(),
})

export const updatePasswordSchema = z.object({
  current_password: z.string().min(8),
  new_password:     z.string().min(8),
  confirm_password: z.string().min(8),
}).refine((data) => data.new_password === data.confirm_password, {
  message: 'Passwords do not match',
  path: ['confirm_password'],
})

export const updateProfileSchema = z.object({
  name:     z.string().min(1).max(50).optional(),
  username: z.string().min(1).max(50).optional(),
})
```

### 6.9 SearchTable Schema

```typescript
// shared/schemas/searchTable.schema.ts
import { z } from 'zod'

export const searchTableSchema = z.object({
  search: z.array(z.object({
    key:   z.string(),
    type:  z.enum(['like', 'equal', 'date', 'datetime-bool', 'in', 'relation-equal', 'relation-like', 'bool']),
    value: z.any(),
  })).optional(),
  sort: z.object({
    key:       z.string(),
    direction: z.enum(['asc', 'desc']),
  }).optional(),
  per_page: z.number().int().positive().optional(),
  page:     z.number().int().positive().optional(),
  all:      z.boolean().optional(),
})
```

### 6.10 Validation Usage in Route Handlers

```typescript
// Example: server/api/admin/board/create.post.ts
import { createBoardSchema } from '~~/shared/schemas/board.schema'

export default defineEventHandler(async (event) => {
  const body = await readBody(event)
  const validated = createBoardSchema.parse(body) // Throws ZodError on failure

  // ... create board logic
})
```

**Error Handling:**
- Zod validation errors are caught by a global error handler and returned as 422 responses with field-level error messages, matching the original Laravel validation response format:

```json
{
  "status": 422,
  "errors": {
    "name": ["Name is required"],
    "description": ["Description is required"]
  }
}
```

---

## 7. Real-Time System (Centrifugo WebSocket)

### 7.1 Architecture

```
API Route Handler (create/update/delete)
        │
        ├── Database write via Drizzle
        ├── Create CardLog entry (audit trail)
        └── Call broadcastToCentrifugo()
                │
                ▼
        Centrifugo Server (HTTP API)
        ├── Receives publish request
        └── Pushes to subscribed WebSocket clients
                │
                ▼
        Frontend socketEventHandler.ts
        ├── Dispatches to handleBoardEvent() or handleCardEvent()
        └── Updates Pinia stores reactively
```

> **Key Difference from Laravel:** Instead of Eloquent Observers firing events implicitly, Nitro route handlers explicitly call `broadcastToCentrifugo()` after successful database writes. This makes the broadcast flow explicit and easier to trace.

### 7.2 Centrifugo Service

```typescript
// server/services/centrifugo.ts

interface BroadcastPayload {
  channel: string      // e.g., "board#1", "card#5"
  event: string        // e.g., "card_create", "comment_delete"
  data: Record<string, any>
}

async function publishToCentrifugo(payload: BroadcastPayload): Promise<void> {
  // POST to Centrifugo HTTP API at CENTRIFUGO_API_URL
  // Authorization: apikey CENTRIFUGO_API_KEY
  await $fetch(`${config.centrifugoApiUrl}/api/publish`, {
    method: 'POST',
    headers: { Authorization: `apikey ${config.centrifugoApiKey}` },
    body: { channel: payload.channel, data: { event: payload.event, ...payload.data } },
  })
}
```

### 7.3 Broadcast Utility

```typescript
// server/utils/broadcast.ts

async function broadcastToCentrifugo(
  channelType: 'board' | 'card',
  channelId: number,
  event: string,
  data: Record<string, any>
): Promise<void> {
  const channel = `${channelType}#${channelId}`
  await publishToCentrifugo({ channel, event, data })
}
```

### 7.4 Channel Format

- Board channel: `board#{board_id}` — receives board, list, and card events
- Card channel: `card#{card_id}` — receives card detail, comment, attachment, log events

### 7.5 Event Types

**Board Channel Events:**

| Event | Trigger | Data |
|-------|---------|------|
| `board_create` | Board created | Board attributes |
| `board_update` | Board updated | Changed fields |
| `board_delete` | Board deleted | Board attributes |
| `board_add_users` | Users added to board | Board + user data |
| `list_create` | BoardList created | List attributes |
| `list_update` | BoardList updated | Changed fields |
| `list_delete` | BoardList deleted | List attributes |
| `list_position` | Lists reordered | Position data |
| `card_create` | Card created | Card data |
| `card_update` | Card updated | Changed fields |
| `card_delete` | Card deleted | Card data |
| `card_position` | Cards reordered | Position data |
| `card_add_users` | Users assigned to card | Card + user data |
| `card_remove_users` | Users removed from card | Card + user data |

**Card Channel Events:**

| Event | Trigger | Data |
|-------|---------|------|
| `card_update` | Card details changed | Changed fields |
| `card_add_users` | Users assigned | User data |
| `card_remove_users` | Users removed | User data |
| `comment_create` | Comment added | Refreshes card |
| `comment_update` | Comment edited | Refreshes card |
| `comment_delete` | Comment removed | Refreshes card |
| `attachment_create` | File uploaded | Refreshes card |
| `attachment_update` | File updated | Refreshes card |
| `attachment_delete` | File removed | Refreshes card |
| `log_create` | Activity logged | Refreshes card |

### 7.6 Channel Authorization

Endpoint: `POST /api/auth/centrifugo/channel` (`server/api/auth/centrifugo/channel.post.ts`)

Validates that the requesting user is either:
- A member of the board referenced in the channel name (`board#{id}` format)
- An admin user

Returns 403 if unauthorized.

### 7.7 Frontend WebSocket Store (Unchanged)

The `socketStore` (Pinia) manages the Centrifugo connection:

- `initCentrifuge()` — creates Centrifuge client instance with JWT token
- `socketConnect()` — establishes WebSocket connection
- `socketDisconnect()` — closes connection and clears all subscriptions
- `updateToken(newToken)` — refreshes JWT for Centrifugo
- `subscribeToChannels(channels, eventType, callback)` — subscribes to one or more channels with event type filtering (publication/error/join/leave)
- `unsubscribeFromChannels(channels)` — unsubscribes from channels
- `publishEvent(channel, data)` — publishes data to a channel

WebSocket URL: configurable via `wsBaseUrl` runtime config (defaults to `ws://localhost:8000/connection/websocket`; production uses `wss://btask.behkame.com/connection/websocket`)

---

## 8. Notification System

### 8.1 Architecture

The notification system replaces Laravel's built-in notification framework with a custom TypeScript service that maintains the same dual-channel delivery model.

```
Route Handler (e.g., card create)
        │
        ▼
NotificationService.send()
├── Check user preference (notification.{key})
├── If disabled → silent drop
├── Store in PostgreSQL notifications table
└── Enqueue BullMQ job (if Telegram connected)
        │
        ▼
BullMQ Worker processes notification.job.ts
├── Send Telegram message via Bot API HTTP
├── On success → update notification.data.status = 'sent'
└── On failure → update notification.data.status = 'failed'
```

### 8.2 NotificationService

```typescript
// server/services/notification.ts

interface NotificationPayload {
  type: string                        // e.g., 'CreateNewBoardNotification'
  userId: number                      // Recipient user ID
  data: Record<string, any>           // Notification data payload
  preferenceKey?: string              // Optional — user.notification.{key} to check
  telegramMessage?: string            // Formatted message for Telegram
}

class NotificationService {
  /**
   * Send a notification to a user via database + optional Telegram.
   * 1. Checks user preference (if preferenceKey provided)
   * 2. Inserts into notifications table
   * 3. Enqueues Telegram delivery job (if user has chat_id)
   */
  async send(payload: NotificationPayload): Promise<void>

  /**
   * Send notification to multiple users (e.g., all board members)
   */
  async sendToMany(users: User[], payload: Omit<NotificationPayload, 'userId'>): Promise<void>
}
```

### 8.3 Board Notifications (6)

| Notification | Preference Key | Description |
|-------------|---------------|-------------|
| `CreateNewBoardNotification` | — (always sent) | New board created |
| `BoardStatusChangeNotification` | `boardStatusChange` | Board activated/deactivated |
| `BoardInfoChangeNotification` | `boardTitleChange` | Board name/description changed |
| `JoinToBoardNotification` | `joinToBoard` | User added to board |
| `RemoveFromBoardNotification` | `removeFromBoard` | User removed from board |
| `DeleteBoardNotification` | — (always sent) | Board deleted |

### 8.4 Card Notifications (12)

| Notification | Preference Key | Description |
|-------------|---------------|-------------|
| `CreateNewCardNotification` | — (always sent) | New card created |
| `AssignToCardNotification` | `assignToCard` | User assigned to card |
| `RemoveFromCardNotification` | `removeFromCard` | User removed from card |
| `DeleteCardNotification` | — (always sent) | Card deleted |
| `CardUserStatusChangeNotification` | — (always sent) | A user's status changed on card |
| `CardStatusChangeNotification` | `cardStatusChange` | Card status changed |
| `CardNewTagNotification` | `cardNewTag` | Tags added/removed |
| `CardNewCommentNotification` | `cardNewComment` | New comment added (includes code blocks) |
| `CardNewAttachmentNotification` | `cardNewAttachment` | File attached |
| `CardInfoChangeNotification` | `cardTitleChange` | Card name changed |
| `CardDescriptionChangeNotification` | `cardDescriptionChange` | Card description changed |

### 8.5 Notification Status Tracking

BullMQ job event callbacks replace Laravel's `NotificationSent` / `NotificationFailed` listeners:

| Event | Action |
|-------|--------|
| Job `completed` | Sets `notification.data.status = 'sent'` in PostgreSQL |
| Job `failed` | Sets `notification.data.status = 'failed'` in PostgreSQL |

### 8.6 User Notification Preferences

Stored as JSONB in `users.notification` column. All 12 preference keys default to `true` on user creation (via `initializeUserDefaults()` utility):

```
boardStatusChange, boardTitleChange, joinToBoard, removeFromBoard,
assignToCard, removeFromCard, cardStatusChange, cardNewTag,
cardNewComment, cardNewAttachment, cardTitleChange, cardDescriptionChange
```

### 8.7 Telegram Message Formatting

```typescript
// server/services/telegram.ts

// Helpers for Telegram MarkdownV2 formatting (replaces Laravel BaseNotification helpers)
function escape(text: string): string          // Escapes special Markdown chars
function oneLine(text: string): string         // Collapses to single line
function code(text: string): string            // Wraps in backticks `` ` ``
function bold(text: string): string            // Wraps in *bold*
```

---

## 9. Queue System (BullMQ)

### 9.1 Architecture

BullMQ replaces Laravel Queue for async job processing. Uses Redis as the message broker.

```typescript
// server/jobs/queue.ts
import { Queue, Worker } from 'bullmq'

// Named queues
export const notificationQueue = new Queue('notifications', { connection: redisConnection })

// Worker processes jobs
const notificationWorker = new Worker('notifications', async (job) => {
  switch (job.name) {
    case 'send-telegram':
      await sendTelegramMessage(job.data)
      break
    case 'telegram-register':
      await processTelegramRegistration(job.data)
      break
  }
}, { connection: redisConnection })
```

### 9.2 Jobs

| Job Name | Queue | Description |
|----------|-------|-------------|
| `send-telegram` | `notifications` | Sends a Telegram message via Bot API |
| `telegram-register` | `notifications` | Polls Telegram Bot API for `/start {token}` registration messages |

### 9.3 Worker Initialization

```typescript
// server/plugins/bullmq.ts
export default defineNitroPlugin((nitroApp) => {
  // Initialize BullMQ workers on server start
  // Workers automatically process queued jobs
  initializeNotificationWorker()
})
```

---

## 10. Telegram Integration

### 10.1 Connection Flow

1. **User requests connection:** `GET /api/client/telegram/connect`
   - Generates UUID token via `crypto.randomUUID()`
   - Stores token in user's `telegram.id` field
   - Returns Telegram bot deep-link URL: `https://t.me/{botUsername}?start={token}`
2. **User opens bot link:** Sends `/start {token}` to the bot in Telegram
3. **Backend processes:** `telegram-register` BullMQ job polls Telegram Bot API for updates
   - Calls `https://api.telegram.org/bot{token}/getUpdates` with cached offset
   - Filters for `/start {token}` messages matching a user's token
   - Extracts `chat_id` and Telegram `username`
   - Updates DB: `user.telegram = { id: token, chat_id, username }`
4. **Caching:** Job stores Telegram API offset in Redis for incremental polling

### 10.2 Disconnection

`POST /api/client/telegram/disconnect` — clears `telegram.chat_id` and `telegram.username` from user record, preserving the token for reconnection.

### 10.3 Telegram Bot API Client

```typescript
// server/services/telegram.ts

const TELEGRAM_API = 'https://api.telegram.org/bot'

async function sendMessage(chatId: string, text: string, parseMode = 'MarkdownV2'): Promise<void> {
  await $fetch(`${TELEGRAM_API}${config.telegramBotToken}/sendMessage`, {
    method: 'POST',
    body: { chat_id: chatId, text, parse_mode: parseMode },
  })
}

async function getUpdates(offset?: number): Promise<TelegramUpdate[]> {
  const result = await $fetch(`${TELEGRAM_API}${config.telegramBotToken}/getUpdates`, {
    method: 'POST',
    body: { offset },
  })
  return result.result
}
```

### 10.4 Frontend Integration (Unchanged)

The `ConnectTelegram.vue` component provides the connection UI. A `Disconnect` modal confirms disconnection.

---

## 11. Frontend Architecture (Unchanged)

> The frontend SPA is **unchanged** from the original architecture. The only infrastructure change is that the API proxy configuration in `nuxt.config.ts` is no longer needed — Nitro serves both the API routes and the SPA from the same origin.

### 11.1 Pages (8 Routes)

| Route | Page File | Layout | Description |
|-------|----------|--------|-------------|
| `/auth/log-in` | `auth/log-in.vue` | `login` | Login form with GSAP animations, username + password fields |
| `/` | `index.vue` | — | Home — redirects to the user's first (private) board |
| `/:id` | `[id].vue` | `board` | **Main kanban board** — drag-drop cards, filter "my tasks", quick-add, real-time updates |
| `/board` | `board/index.vue` | `default` | Admin: Board CRUD table |
| `/user` | `user/index.vue` | `default` | Admin: User CRUD table |
| `/tag` | `tag/index.vue` | `default` | Admin: Tag CRUD table with color picker |
| `/notification` | `notification/index.vue` | `default` | Admin: Notification log viewer |
| `/profile` | `profile/index.vue` | `default` | User profile display |

### 11.2 Layouts (3)

| Layout | Structure | Used By |
|--------|-----------|---------|
| `board` | Sidebar \| PlanetOrbit + Content \| BoardMembers (right panel) | Kanban board page |
| `default` | Sidebar \| Scrollable Content | Admin pages, profile |
| `login` | Empty (slot only) | Auth page |

All layouts feature a collapsible sidebar with board navigation and GSAP animations.

### 11.3 Components

#### Layout Components
- **Sidebar** — User profile + avatar, board list with icons, admin navigation (users, boards, notifications, tags), logout button
- **BoardMembers** — Right sidebar panel showing board members
- **PlanetOrbit** — Animated orbital visual effect
- **ManagementBackdrop** — Styling/effects wrapper

#### Card Components
- **list.vue** — Kanban column/list container
- **Draggable.vue** — Cards container with vuedraggable for drag-drop reordering
- **Item.vue** — Individual task card display
- **empty.vue** — Empty state placeholder
- **logIn.vue** — Login form card
- **profile.vue** — User profile card

#### Card-Board Components (Task Detail)
- **CommentAndActivity.vue** — Combined comments + activity log feed
- **CommentInput.vue** — Comment form with optional file attachment
- **CommentItem.vue** — Single comment display

#### Modal Components (13+)

| Category | Modals |
|----------|--------|
| **Board/Task** | Add (create/edit task), Delete (confirmation) |
| **Board Management** | Add (create board), Details (view/edit), Edit |
| **User** | Add, Detail (view/edit), Edit, Password change, Delete confirmation |
| **Task** | Task detail modal, Activity Log modal |
| **Tag** | Add, Edit, Delete |
| **Comment** | Delete confirmation |
| **Telegram** | Disconnect confirmation |
| **Log Out** | Logout confirmation |

All modals use a base `modal/index.vue` wrapper (teleported to body, centered overlay, scale animation).

#### UI Kit (Reusable Components)
- **Icon** — Icon wrapper
- **Spinner** — Loading indicator
- **Checkbox** — Styled checkbox
- **Collapse** — Accordion/collapsible panel
- **Option** — Select option
- **Input** — Text input with prefix/suffix icons and size variants
- **DatePickerPersian** — Persian (Jalali) date picker
- **DropDown** — Select dropdown
- **MultiDropDown** — Multi-select dropdown
- **File** — File upload input
- **Table/Search** — Searchable table with filter, sort, and pagination
- **Tag** — Tag display chip with color
- **Avatar** — User avatar with fallback

### 11.4 Pinia Stores (5)

| Store | State | Key Actions |
|-------|-------|-------------|
| **authStore** | `user`, `token`, `is_authenticated` | `login()`, `logout()`, `refreshToken()`, `fetchUser()` |
| **boardStore** | `boards[]`, `privateBoardID`, `isLoading` | `fetchBoards()` (admin gets all, client gets own), `updateBoardInList()`, `removeBoardFromList()` |
| **boardListStore** | `boardLists[]`, `boardMembers[]`, `boardData`, `isLoading` | `getBoardLists(boardId)` (fetches overview), `updateMember()` |
| **sidebarStore** | `isOpen` | `toggle()`, `open()`, `close()` |
| **socketStore** | `state`, `centrifuge`, `subscriptions` | `initCentrifuge()`, `socketConnect()`, `socketDisconnect()`, `subscribeToChannels()`, `unsubscribeFromChannels()`, `publishEvent()` |

### 11.5 Composables (3)

| Composable | Purpose |
|-----------|---------|
| **useAuth** | localStorage wrapper — `login(token)`, `logout()`, `getAuthToken()`, `isAuthenticated()` |
| **useApi** | Complete typed HTTP client with all backend endpoints. Methods: `get<T>()`, `post<T>()`, `put<T>()`, `del<T>()`, `form<T>()`. Auto-attaches Bearer token. Handles 401 (auto-logout), 422 (validation errors), 469 (error field). Organizes endpoints by resource: `auth`, `users`, `boards`, `board_lists`, `tags`, `cards`, `comments`, `attachments`, `notifications`, `centrifugo`, `telegram` |
| **useCardModal** | Global card modal state — `openCardModal` ref (cardId, refreshCardData, cardData), `isCreatingCommentWithAttachment` flag |

### 11.6 TypeScript Types (14)

| Type | Key Fields |
|------|-----------|
| `UserModel` | id, name, username, password, avatar, telegram, notification, role, is_disable |
| `BoardModel` | id, user_id, name, description, status, icon, icon_name?, members[], board_lists[] |
| `BoardListModel` | id, board_id, name, position, cards[] |
| `CardModel` | id, board_list_id?, name, description, position, tags[], users[], attachments[], comments[] |
| `CardUserModel` | id, user_id, card_id, status |
| `CardLogModel` | id, user_id, card_id, board_id, type, data |
| `CommentModel` | id, user_id, user_name, user?, card_id, user_avatar, attachment_id?, attachment?, body, pin |
| `AttachmentModel` | id, user_id, card_id, name, type, path |
| `TagModel` | id, name, color: {name, value}, description |
| `NotificationModel` | id, type, notifiable, data, read_at |
| `BoardUserModel` | id, user_id, board_id |
| `Status / StatusId` | Enum-like with colors: None (#B6B1BE), Pending (#EAB035), Doing (#A777E1), On hold (#E55151), Done (#48CC4C) |
| `FailedJobModel` | id, uuid, connection, queue, payload, exception, failed_at |
| `RequestOptions / Response<T> / PaginatedResponse<T>` | HTTP request/response patterns, `EnsureResponse<T>` utility |

### 11.7 Plugins (5)

| Plugin | Purpose |
|--------|---------|
| `0.auth.client.ts` | Restores auth state from localStorage on app startup |
| `draggable.client.ts` | Registers global `<Draggable>` component (vuedraggable v4) |
| `iconsax.client.ts` | Registers global `<VsxIcon>` component (vue-iconsax v2) |
| `sonner.client.ts` | Registers global `<Toaster>` component + `$toast` helper (vue-sonner) |
| `gsap.client.js` | Registers ScrollTrigger + ScrollToPlugin, provides `$gsap` (GSAP v3.13) |

### 11.8 Route Middleware

**Auth middleware** (`auth.ts`): Applied to all routes except login. Checks `authStore.isLoggedIn` — redirects to `/auth/log-in` if unauthenticated.

---

## 12. Audit Trail & Logging

### 12.1 CardLog System

Every mutation to cards, comments, and attachments is logged via the `createCardLog()` utility function (replaces Laravel Observers):

| Type | Value | Called By |
|------|-------|-----------|
| Card operations | 0 | Card route handlers — create, update, delete |
| Comment operations | 1 | Comment route handlers — create, update, delete |
| Attachment operations | 2 | Attachment route handlers — create, update, delete |

```typescript
// server/utils/cardLog.ts

async function createCardLog(params: {
  userId: number
  cardId: number
  boardId: number
  type: 0 | 1 | 2          // card, comment, attachment
  data: {
    message: string
    changes?: Record<string, any>
    old_data?: Record<string, any>
    [key: string]: any       // card/comment/attachment data
  }
}): Promise<void> {
  const [log] = await db.insert(cardLogs).values(params).returning()

  // Broadcast log_create to card channel
  await broadcastToCentrifugo('card', params.cardId, 'log_create', log)
}
```

Each log entry stores:
- **userId** — who performed the action (from `event.context.user`)
- **cardId** — which card was affected
- **boardId** — which board the card belongs to
- **data** — JSONB containing `{message, changes?, old_data?, card/comment/attachment}`

### 12.2 Activity Timeline

```typescript
// server/utils/cardLog.ts

async function getCombinedActivity(cardId: number): Promise<ActivityItem[]> {
  const [commentsResult, logsResult] = await Promise.all([
    db.query.comments.findMany({ where: eq(comments.cardId, cardId), with: { user: true, attachment: true } }),
    db.query.cardLogs.findMany({ where: eq(cardLogs.cardId, cardId), with: { user: true } }),
  ])

  // Merge and sort chronologically
  return [...commentsResult, ...logsResult].sort((a, b) =>
    new Date(a.createdAt).getTime() - new Date(b.createdAt).getTime()
  )
}
```

### 12.3 Board-Level Logs

Admin and client route handlers expose `logs-table` endpoints per board, enabling a full audit trail of all activity within a board via the SearchTable service.

---

## 13. API Response Transformers

### 13.1 Response Format

All API responses are wrapped in a standard format:

```json
{
  "status": 200,
  "data": { ... }
}
```

```typescript
// server/utils/response.ts

function apiResponse<T>(data: T, status = 200) {
  return { status, data }
}

function apiError(message: string, status = 400, errors?: Record<string, string[]>) {
  return { status, message, errors }
}
```

### 13.2 Transformer Definitions

Transformers replace Laravel API Resources. Each is a pure function that takes a raw DB row (with optional relations) and returns a clean API response shape.

| Transformer | Input → Output |
|------------|----------------|
| **transformUser** | `users` row → `{ id, name, username, avatar, role, telegram, is_disable, status? }` — includes pivot status from `card_user` if present |
| **transformBoard** | `boards` row + relations → `{ id, name, description, icon, icon_name, status, users_count, members[], board_lists[] }` |
| **transformBoardList** | `boardLists` row + relations → `{ id, board_id, name, position, cards[] }` |
| **transformCard** | `cards` row + relations → `{ id, boardlist_id, name, description, position, tags[], users[], comments[], card_attachments[], comment_attachments[] }` — tags are expanded with color enum name + value; attachments split by comment linkage |
| **transformComment** | `comments` row + relations → `{ id, user_id, card_id, body, user_name, user_avatar, attachment }` |
| **transformAttachment** | `attachments` row → `{ id, card_id, name, path, type }` |
| **transformTag** | `tags` row → `{ id, name, description, color: { name, value } }` — expands color enum |
| **transformCardLog** | `cardLogs` row + relations → `{ id, user_id, card_id, board_id, type, data, user, card: { id, name } }` |
| **transformNotification** | `notifications` row → `{ id, type, data, read_at, created_at }` — `created_at` formatted as `Y-m-d H:i:s` |

### 13.3 Enum Conversion in Validation

Zod schemas with `.coerce.number()` handle string-to-integer enum conversion (replaces Laravel's `BaseFormRequest` enum conversion):
- `status` → `CardUserStatusEnum`
- `role` → `UserRoleEnum`
- `color` → `TagColorEnum`
- `icon` → `BoardIconEnum`

---

## 14. Server Utilities

### 14.1 Password Hashing

```typescript
// server/utils/password.ts
import { hash, compare } from 'bcrypt'

const SALT_ROUNDS = 10

async function hashPassword(password: string): Promise<string>
async function verifyPassword(password: string, hashedPassword: string): Promise<boolean>
```

### 14.2 User Defaults

```typescript
// server/utils/userDefaults.ts

function initializeUserDefaults(): { telegram: object; notification: object } {
  return {
    telegram: {
      id: generateRandomToken(12),  // 12-char random string
    },
    notification: {
      boardStatusChange: true,
      boardTitleChange: true,
      joinToBoard: true,
      removeFromBoard: true,
      assignToCard: true,
      removeFromCard: true,
      cardStatusChange: true,
      cardNewTag: true,
      cardNewComment: true,
      cardNewAttachment: true,
      cardTitleChange: true,
      cardDescriptionChange: true,
    },
  }
}
```

### 14.3 File Storage

```typescript
// server/services/fileStorage.ts

const MEDIA_DIR = 'storage/media'
const AVATAR_DIR = 'storage/avatar'

async function uploadFile(file: MultiPartData, directory: string): Promise<string>  // Returns stored path
async function deleteFile(path: string): Promise<void>                              // Removes from disk
function getPublicUrl(path: string): string                                          // Returns public URL
```

- Attachments stored in `storage/media/`
- Avatars stored in `storage/avatar/`
- Files served via Nitro's static asset handling or a `/storage/**` route

---

## 15. Database Management

### 15.1 Drizzle Configuration

```typescript
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit'

export default defineConfig({
  schema: './server/database/schema/index.ts',
  out: './server/database/migrations',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
})
```

### 15.2 Database Client

```typescript
// server/database/index.ts
import { drizzle } from 'drizzle-orm/node-postgres'
import * as schema from './schema'

export const db = drizzle(process.env.DATABASE_URL!, { schema })
```

### 15.3 Migration Commands

```bash
# Generate migration from schema changes
npx drizzle-kit generate

# Apply pending migrations
npx drizzle-kit migrate

# Open Drizzle Studio (database browser)
npx drizzle-kit studio

# Push schema directly (development only)
npx drizzle-kit push
```

### 15.4 Seeder

```typescript
// server/database/seed.ts
// Run via: npx tsx server/database/seed.ts

async function seed() {
  // Create admin user
  // Create default tags
  // Create sample boards, lists, cards
}
```

---

## 16. Infrastructure & Deployment

### 16.1 Docker Services

```yaml
# compose.dev.yaml
services:
  app:
    build: .
    ports: ["3000:3000"]
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/taskmanagement
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=${JWT_SECRET}
      - CENTRIFUGO_API_URL=http://centrifugo:8000/api
      - CENTRIFUGO_API_KEY=${CENTRIFUGO_API_KEY}
      - CENTRIFUGO_SECRET=${CENTRIFUGO_SECRET}
      - TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}
    depends_on: [db, redis, centrifugo]

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: taskmanagement
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  centrifugo:
    image: centrifugo/centrifugo:v5
    ports: ["8000:8000"]
    volumes: ["./centrifugo.json:/centrifugo/config.json"]
    command: centrifugo --config=/centrifugo/config.json

volumes:
  pgdata:
```

### 16.2 Environment Variables

```env
# Database
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/taskmanagement

# Redis (BullMQ + Cache + Token Blacklist)
REDIS_URL=redis://localhost:6379

# JWT
JWT_SECRET=your-secret-key-min-32-chars
JWT_TTL=110000000000          # Minutes — effectively never expires
JWT_REFRESH_TTL=20160         # Minutes — 2 weeks

# Centrifugo
CENTRIFUGO_API_URL=http://localhost:8000/api
CENTRIFUGO_API_KEY=your-centrifugo-api-key
CENTRIFUGO_SECRET=your-centrifugo-secret
WS_BASE_URL=ws://localhost:8000/connection/websocket

# Telegram
TELEGRAM_BOT_TOKEN=your-telegram-bot-token

# File Storage
STORAGE_PATH=./storage
MAX_FILE_SIZE=10485760        # 10MB
```

### 16.3 CORS

Handled via Nitro's built-in `routeRules` in `nuxt.config.ts`:

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    '/api/**': {
      cors: true,
      headers: {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': '*',
        'Access-Control-Allow-Headers': '*',
      },
    },
  },
})
```

### 16.4 Runtime Configuration

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  ssr: false,  // SPA mode

  runtimeConfig: {
    // Server-only (private)
    databaseUrl: process.env.DATABASE_URL,
    redisUrl: process.env.REDIS_URL,
    jwtSecret: process.env.JWT_SECRET,
    jwtTtl: process.env.JWT_TTL,
    jwtRefreshTtl: process.env.JWT_REFRESH_TTL,
    centrifugoApiUrl: process.env.CENTRIFUGO_API_URL,
    centrifugoApiKey: process.env.CENTRIFUGO_API_KEY,
    centrifugoSecret: process.env.CENTRIFUGO_SECRET,
    telegramBotToken: process.env.TELEGRAM_BOT_TOKEN,
    storagePath: process.env.STORAGE_PATH || './storage',
    maxFileSize: parseInt(process.env.MAX_FILE_SIZE || '10485760'),

    // Client-accessible (public)
    public: {
      wsBaseUrl: process.env.WS_BASE_URL || 'ws://localhost:8000/connection/websocket',
      socketDebug: process.env.SOCKET_DEBUG === 'true',
    },
  },
})
```

### 16.5 File Storage

- Attachments and avatars stored on disk via `server/services/fileStorage.ts`
- Files served via a Nitro catch-all route at `/storage/**` or static asset configuration
- File operations: upload, update (replace), delete (removes from disk + DB)

### 16.6 Nuxt Configuration

```typescript
// nuxt.config.ts (complete)
export default defineNuxtConfig({
  compatibilityDate: '2024-11-01',
  future: { compatibilityVersion: 4 },
  ssr: false,

  modules: [
    '@pinia/nuxt',
    '@nuxt/image',
    '@nuxt/icon',
    'nuxt-easy-lightbox',
    '@tailwindcss/vite',
  ],

  // Runtime config as above...

  // Route rules as above...

  nitro: {
    // BullMQ + Drizzle require Node.js runtime
    preset: 'node-server',
  },

  vite: {
    plugins: [tailwindcss()],
  },
})
```

---

## 17. Feature Summary Matrix

| Feature | Server (Nitro) | Client (SPA) |
|---------|---------------|--------------|
| JWT Authentication | ✅ jose — Login/Logout/Refresh, Redis blacklist | ✅ Token persistence, auto-refresh, auth middleware |
| Role-Based Access | ✅ Admin + Normal roles, Nitro middleware | ✅ Conditional admin navigation |
| Board CRUD | ✅ Full CRUD + member sync via Drizzle | ✅ Admin table + modals |
| Board Kanban View | ✅ Overview endpoint with nested Drizzle queries | ✅ Drag-drop columns and cards |
| BoardList Management | ✅ CRUD + batch position update | ✅ Drag-drop list reordering |
| Card Management | ✅ CRUD + quick-create + duplicate + soft delete | ✅ Card modals, inline quick-add |
| Card Member Assignment | ✅ Sync + individual remove via junction table | ✅ Member management in card modal |
| Card Status Tracking | ✅ Per-user status (None/Pending/Doing/OnHold/Done) | ✅ Status badges with colors |
| Tag System | ✅ CRUD + card tag sync (JSONB) | ✅ Tag chips with 9 colors |
| Comments | ✅ CRUD + pin + file attachment | ✅ Comment thread in card modal |
| File Attachments | ✅ Upload/update/delete, 6 file types, disk storage | ✅ File upload in cards and comments |
| Real-Time Updates | ✅ broadcastToCentrifugo() utility, 15+ event types | ✅ socketStore + socketEventHandler |
| Notifications | ✅ 18 types via NotificationService, database + Telegram | ✅ Admin notification log viewer |
| Notification Preferences | ✅ 12 toggleable preferences per user (JSONB) | ✅ User preference management |
| Telegram Bot | ✅ Connect/disconnect, BullMQ job for registration | ✅ ConnectTelegram component + disconnect modal |
| Audit Trail | ✅ CardLog utility for all card/comment/attachment changes | ✅ Activity timeline in card modal |
| User Profile | ✅ Avatar upload/delete, password change (bcrypt) | ✅ Profile page + edit modals |
| Admin Dashboard | ✅ SearchTable service for all entities | ✅ CRUD tables with search/filter/sort/pagination |
| Validation | ✅ Zod schemas (shared between server + client) | ✅ Same Zod schemas for client-side validation |
| Persian Date Picker | — | ✅ vue3-persian-datetime-picker |
| Animations | — | ✅ GSAP (login, sidebar) |
| Dark Theme | — | ✅ Purple/violet palette, custom Tailwind config |
| Image Lightbox | — | ✅ nuxt-easy-lightbox for attachment previews |
| Toast Notifications | — | ✅ vue-sonner |

---

## 18. Dependencies

### 18.1 Server Dependencies

```json
{
  "dependencies": {
    "drizzle-orm": "latest",
    "pg": "latest",
    "jose": "latest",
    "bcrypt": "latest",
    "bullmq": "latest",
    "ioredis": "latest",
    "zod": "latest",
    "formidable": "latest"
  },
  "devDependencies": {
    "drizzle-kit": "latest",
    "@types/pg": "latest",
    "@types/bcrypt": "latest"
  }
}
```

### 18.2 Client Dependencies (Unchanged)

```json
{
  "dependencies": {
    "pinia": "^3.0.4",
    "centrifuge": "^5.5.2",
    "gsap": "^3.13.0",
    "vue-sonner": "latest",
    "vue-draggable-plus": "latest",
    "vue-iconsax": "^2.0.0",
    "vue3-persian-datetime-picker": "^1.2.2",
    "hammerjs": "latest"
  },
  "devDependencies": {
    "@pinia/nuxt": "latest",
    "@nuxt/image": "latest",
    "@nuxt/icon": "latest",
    "nuxt-easy-lightbox": "latest",
    "@tailwindcss/vite": "latest",
    "tailwindcss": "^4.0.0"
  }
}
```
