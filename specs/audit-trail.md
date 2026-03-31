# Spec: Audit Trail & CardLog System

## Status: Approved

## Purpose

Every mutation to cards, comments, and attachments must be logged for audit and activity feed purposes. This spec defines the `createCardLog()` utility, the `cardLogs` database schema, board-level log API endpoints, real-time log broadcasting via Centrifugo, and the combined activity feed displayed in the card detail modal.

This system replaces Laravel Observers with explicit `createCardLog()` calls in each route handler that performs a mutation.

---

## User Stories

- As a **board member**, I want to see a chronological activity feed in the card detail modal so I can understand the history of changes to a card.
- As a **board member**, I want card activity to update in real time without a page refresh so I always see the latest changes.
- As an **admin**, I want to query paginated, filterable audit logs for any board so I can investigate past activity.
- As a **client user**, I want to view activity logs for boards I am a member of so I can audit changes within my own workspace.
- As a **backend developer**, I want a single `createCardLog()` utility so that log creation and broadcasting are always consistent and never duplicated across route handlers.

---

## Acceptance Criteria

### AC-1 — Log Creation on Card CRUD
- [ ] `createCardLog()` is called in the card `create` route handler with `type: 0`
- [ ] `createCardLog()` is called in the card `update` route handler with `type: 0`
- [ ] `createCardLog()` is called in the card `delete` route handler with `type: 0`
- [ ] The `data.changes` field is populated on update with a before/after diff
- [ ] The `data.old_data` field is populated on delete with the card snapshot prior to deletion

### AC-2 — Log Creation on Comment CRUD
- [ ] `createCardLog()` is called in the comment `create` route handler with `type: 1`
- [ ] `createCardLog()` is called in the comment `update` route handler with `type: 1`
- [ ] `createCardLog()` is called in the comment `delete` route handler with `type: 1`

### AC-3 — Log Creation on Attachment CRUD
- [ ] `createCardLog()` is called in the attachment `create` route handler with `type: 2`
- [ ] `createCardLog()` is called in the attachment `update` route handler with `type: 2`
- [ ] `createCardLog()` is called in the attachment `delete` route handler with `type: 2`

### AC-4 — Data Payload Fields
- [ ] Every log entry stores `userId` from `event.context.user`
- [ ] Every log entry stores `cardId` of the affected card
- [ ] Every log entry stores `boardId` resolved via `cards → boardLists → boards`
- [ ] Every log entry stores a `data.message` human-readable string describing the action
- [ ] Update operations include `data.changes` (before/after diff as `Record<string, any>`)
- [ ] Delete operations include `data.old_data` (full snapshot prior to deletion)

### AC-5 — Real-Time Broadcast
- [ ] After every successful `createCardLog()` insert, the log is broadcast to Centrifugo
- [ ] The broadcast targets the `card#{cardId}` channel
- [ ] The broadcast event name is `log_create`
- [ ] The full inserted log row (returned from `.returning()`) is the broadcast payload
- [ ] The frontend `socketEventHandler` refreshes the card activity timeline on receipt of `log_create`

### AC-6 — Combined Activity Feed
- [ ] `getCombinedActivity(cardId)` fetches both comments and cardLogs for the given card in parallel
- [ ] Comments are fetched with their associated `user` and `attachment` relations
- [ ] CardLogs are fetched with their associated `user` relation
- [ ] The merged result is sorted chronologically by `createdAt` ascending
- [ ] The combined result is returned as `ActivityItem[]`

### AC-7 — Board-Level Log Endpoints
- [ ] `POST /api/admin/board/logs-table/[id]` returns paginated, filterable logs for a board (admin access only)
- [ ] `POST /api/client/board/logs-table/[id]` returns paginated, filterable logs for a board (member access only)
- [ ] Both endpoints use the `SearchTable` service for pagination and filtering
- [ ] Both endpoints enforce their respective access middleware (`admin.ts` / `auth.ts` + board membership)

### AC-8 — Database Schema
- [ ] The `card_logs` table exists with columns: `id`, `user_id`, `card_id`, `board_id`, `type`, `data`, `created_at`, `updated_at`
- [ ] `user_id` has a foreign key reference to `users.id`
- [ ] `card_id` has a foreign key reference to `cards.id`
- [ ] `board_id` has a foreign key reference to `boards.id`
- [ ] `type` is a `smallint` (0 = card, 1 = comment, 2 = attachment)
- [ ] `data` is a `jsonb` column defaulting to `{}`
- [ ] `created_at` and `updated_at` use `timestamp with time zone` and default to `now()`

---

## Component Tree / File Structure

```
server/
├── utils/
│   └── cardLog.ts                          # createCardLog() + getCombinedActivity()
├── database/
│   └── schema/
│       └── cardLogs.ts                     # Drizzle table definition
└── api/
    ├── admin/
    │   └── board/
    │       └── logs-table/
    │           └── [id].post.ts            # Admin board activity log table
    └── client/
        └── board/
            └── logs-table/
                └── [id].post.ts            # Client board activity log table

app/
└── components/
    └── card-board/
        └── CommentAndActivity.vue          # Combined comments + activity feed (card detail modal)
```

---

## Data Model

### Database Schema

```typescript
// server/database/schema/cardLogs.ts
export const cardLogs = pgTable('card_logs', {
  id:        serial('id').primaryKey(),
  userId:    integer('user_id').notNull().references(() => users.id),
  cardId:    integer('card_id').notNull().references(() => cards.id),
  boardId:   integer('board_id').notNull().references(() => boards.id),
  type:      smallint('type').notNull(),  // 0=card, 1=comment, 2=attachment
  data:      jsonb('data')
               .$type<{
                 message?: string
                 changes?: Record<string, any>
                 old_data?: Record<string, any>
               }>()
               .notNull()
               .default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

### Log Type Enum

| Type Value | Meaning | Called By |
|---|---|---|
| `0` | Card operation | Card route handlers (create, update, delete) |
| `1` | Comment operation | Comment route handlers (create, update, delete) |
| `2` | Attachment operation | Attachment route handlers (create, update, delete) |

### API Endpoints Consumed

| Method | Path | Access | Description |
|---|---|---|---|
| `GET` | `/api/client/card/show/[id]` | Member | Returns full card with combined activity (`getCombinedActivity()`) |
| `POST` | `/api/admin/board/logs-table/[id]` | Admin | Paginated, filterable board activity logs |
| `POST` | `/api/client/board/logs-table/[id]` | Member | Paginated, filterable board activity logs |

### TypeScript Interfaces

```typescript
// shared/types/cardLog.types.ts

interface CardLogModel {
  id: number
  user_id: number
  card_id: number
  board_id: number
  type: 0 | 1 | 2
  data: {
    message?: string
    changes?: Record<string, any>
    old_data?: Record<string, any>
    [key: string]: any
  }
  createdAt: string
  updatedAt: string
}

interface ActivityItem {
  id: number | string
  type: 'comment' | 'log'
  createdAt: string
  user: UserModel
  // comment-specific fields
  body?: string
  pin?: boolean
  attachment?: AttachmentModel
  // log-specific fields
  logType?: 0 | 1 | 2
  data?: CardLogModel['data']
}
```

---

## State Management

- No dedicated Pinia store is required for card logs.
- The card detail modal fetches combined activity as part of the `GET /api/client/card/show/[id]` response — this is managed by the existing card store or composable.
- Real-time updates are handled by the `socketEventHandler` composable: on receipt of a `log_create` event on the `card#{cardId}` channel, the activity list in the card detail modal is refreshed (either by re-fetching or appending the new item directly from the payload).

---

## Edge Cases

- **boardId resolution**: `boardId` must be resolved server-side via the `cards → boardLists → boards` join before calling `createCardLog()`. It must never be accepted as a client-supplied value.
- **Broadcast failure**: If `broadcastToCentrifugo()` throws, the error should be logged but must not cause the parent route handler to return an error response. The log row has already been persisted.
- **Deleted users**: Logs reference `userId` via a foreign key. If a user is deleted, the FK constraint must be handled (restrict or nullify depending on data retention policy). This is out of scope for this spec but should be noted during schema migration.
- **High-frequency updates**: Rapid successive updates to the same card will produce one log per mutation. No deduplication or debouncing is applied server-side.
- **Missing `data.message`**: All callers must supply a human-readable `message` string. This is a convention, not a DB constraint. Future work may enforce this via Zod.
- **Combined activity sort stability**: When a comment and a log share the exact same `createdAt` timestamp (edge case), sort order between them is undefined. This is acceptable.

---

## Non-Goals

- **Log editing or deletion**: Card logs are immutable. No update or delete endpoints are provided.
- **Log retention policy / archiving**: Out of scope for this spec.
- **Client-side log filtering in card modal**: The combined activity feed in `CommentAndActivity.vue` displays all items without client-side filtering.
- **Diff computation library**: `data.changes` is a plain object constructed manually by the route handler. No structured diff library is required.
- **Per-field change granularity beyond what is described**: Only the fields explicitly diffed by each route handler will appear in `data.changes`.

---

## Dependencies

| Dependency | Purpose |
|---|---|
| `server/database/schema/cards.ts` | FK reference; `boardId` resolved via card → boardList → board |
| `server/database/schema/boards.ts` | FK reference for `board_id` |
| `server/database/schema/users.ts` | FK reference for `user_id` |
| `server/database/schema/comments.ts` | Fetched in `getCombinedActivity()` |
| `server/utils/broadcast.ts` | `broadcastToCentrifugo()` called after every log insert |
| `server/services/searchTable.ts` | Used by both `logs-table` board endpoints for pagination/filtering |
| Centrifugo | Real-time `log_create` broadcast target |
| Drizzle ORM | All DB operations (`db.insert`, `db.query`) |
