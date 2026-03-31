# Spec: Notification System

## Status: Approved

---

## Purpose

Implement a full-stack notification pipeline that persists notifications to PostgreSQL, respects per-user preferences, and optionally delivers messages to Telegram via BullMQ background jobs. Every significant board and card event generates a notification; 12 of the 18 types are gated by user preferences. Telegram delivery is fire-and-forget with status tracking (`sent` / `failed`).

---

## User Stories

1. **As a user**, I want to receive notifications when I am added to or removed from a board or card, so I know my access has changed.
2. **As a user**, I want to receive notifications when a board or card I belong to is updated, deleted, or has its status changed.
3. **As a user**, I want to control which notification types I receive, so I am not overwhelmed by events I don't care about.
4. **As a user**, I want notifications delivered to my Telegram account (if connected), so I stay informed without checking the app.
5. **As an admin**, I want to view all system notifications in a paginated table, so I can audit notification activity.

---

## Acceptance Criteria

### AC-1: Preference Checking
- [ ] When `NotificationService.send()` is called with a `preferenceKey`, it reads `user.notification[preferenceKey]` from the database.
- [ ] If the preference value is `false`, the notification is silently dropped — no database insert, no BullMQ job.
- [ ] If no `preferenceKey` is provided, the notification is always processed (no preference check).
- [ ] Notifications with preference key `undefined` or `null` are treated as always-sent.

### AC-2: Database Insert
- [ ] All non-dropped notifications are inserted into the `notifications` table.
- [ ] The `data` column stores a JSON-stringified object containing the full notification payload.
- [ ] The `type` column stores the notification class name (e.g., `'CreateNewBoardNotification'`).
- [ ] The `notifiable_id` column stores the recipient's user ID.
- [ ] `read_at` defaults to `null` on creation.
- [ ] `created_at` and `updated_at` are set automatically.

### AC-3: BullMQ Enqueue
- [ ] After a successful DB insert, if the recipient user has a `chat_id` (Telegram connected), a `send-telegram` job is enqueued to the `notifications` queue.
- [ ] The job payload includes the `telegramMessage` string and the user's `chat_id`.
- [ ] If the user has no `chat_id`, no BullMQ job is enqueued.

### AC-4: Telegram Delivery
- [ ] The BullMQ worker processes `send-telegram` jobs by calling the Telegram Bot API.
- [ ] Messages are formatted using MarkdownV2 with helpers: `escape()`, `oneLine()`, `code()`, `bold()`.
- [ ] On job success (`completed` event), `notification.data.status` is updated to `'sent'` in PostgreSQL.
- [ ] On job failure (`failed` event), `notification.data.status` is updated to `'failed'` in PostgreSQL.

### AC-5: Multi-User Delivery (sendToMany)
- [ ] `NotificationService.sendToMany(users, payload)` sends the same notification to each user in the array.
- [ ] Each invocation runs the full pipeline per user (preference check → DB insert → BullMQ enqueue).
- [ ] The `userId` field is populated from each individual `User` object in the array.

### AC-6: Board Notifications — 6 Types
- [ ] `CreateNewBoardNotification` — always sent; triggered on board creation.
- [ ] `BoardStatusChangeNotification` — gated by `boardStatusChange` preference; triggered when board is activated/deactivated.
- [ ] `BoardInfoChangeNotification` — gated by `boardTitleChange` preference; triggered when board name or description changes.
- [ ] `JoinToBoardNotification` — gated by `joinToBoard` preference; triggered when a user is added to a board.
- [ ] `RemoveFromBoardNotification` — gated by `removeFromBoard` preference; triggered when a user is removed from a board.
- [ ] `DeleteBoardNotification` — always sent; triggered on board deletion.

### AC-7: Card Notifications — 12 Types
- [ ] `CreateNewCardNotification` — always sent; triggered on card creation.
- [ ] `AssignToCardNotification` — gated by `assignToCard` preference; triggered when a user is assigned to a card.
- [ ] `RemoveFromCardNotification` — gated by `removeFromCard` preference; triggered when a user is removed from a card.
- [ ] `DeleteCardNotification` — always sent; triggered on card deletion.
- [ ] `CardUserStatusChangeNotification` — always sent; triggered when a user's status on a card changes.
- [ ] `CardStatusChangeNotification` — gated by `cardStatusChange` preference; triggered when card status changes.
- [ ] `CardNewTagNotification` — gated by `cardNewTag` preference; triggered when tags are added or removed.
- [ ] `CardNewCommentNotification` — gated by `cardNewComment` preference; triggered on new comment (includes code block rendering).
- [ ] `CardNewAttachmentNotification` — gated by `cardNewAttachment` preference; triggered when a file is attached.
- [ ] `CardInfoChangeNotification` — gated by `cardTitleChange` preference; triggered when card name changes.
- [ ] `CardDescriptionChangeNotification` — gated by `cardDescriptionChange` preference; triggered when card description changes.

### AC-8: User Notification Preferences — Defaults
- [ ] On user creation, `initializeUserDefaults()` sets all 12 preference keys to `true` in the `users.notification` JSONB column.
- [ ] The 12 keys are: `boardStatusChange`, `boardTitleChange`, `joinToBoard`, `removeFromBoard`, `assignToCard`, `removeFromCard`, `cardStatusChange`, `cardNewTag`, `cardNewComment`, `cardNewAttachment`, `cardTitleChange`, `cardDescriptionChange`.
- [ ] Preference keys not listed here have no effect on always-sent notification types.

### AC-9: BullMQ Worker Initialization
- [ ] The Nitro plugin at `server/plugins/bullmq.ts` calls `initializeNotificationWorker()` on server startup.
- [ ] The worker listens on the `notifications` queue and handles at minimum the `send-telegram` and `telegram-register` job names.
- [ ] Worker and queue both use the shared `redisConnection` instance.

### AC-10: Admin Endpoints
- [ ] `GET /api/admin/notification/all` returns all notification records (requires admin middleware).
- [ ] `POST /api/admin/notification/table` returns a paginated, date-descending list of notifications using the SearchTable service (requires admin middleware).

---

## Component Tree / File Structure

```
server/
├── services/
│   ├── notification.ts          # NotificationService class (send, sendToMany)
│   └── telegram.ts              # Telegram Bot API client + MarkdownV2 helpers
├── jobs/
│   ├── queue.ts                 # BullMQ Queue + Worker definitions + redisConnection
│   └── notification.job.ts      # Async notification dispatch job implementation
├── plugins/
│   └── bullmq.ts                # Nitro plugin — initializes BullMQ worker on startup
└── database/
    └── schema/
        └── notifications.ts     # Drizzle ORM table definition

server/api/admin/notification/
├── all.get.ts                   # GET /api/admin/notification/all
└── table.post.ts                # POST /api/admin/notification/table
```

No client-side components are in scope for this spec. Notification reads and in-app display are covered by a separate spec.

---

## Data Model

### Database Table: `notifications`

```typescript
// server/database/schema/notifications.ts
export const notifications = pgTable('notifications', {
  id:           uuid('id').primaryKey().defaultRandom(),
  type:         varchar('type', { length: 255 }).notNull(),
  notifiableId: integer('notifiable_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  data:         text('data').notNull(),          // JSON.stringify(NotificationPayload)
  readAt:       timestamp('read_at', { withTimezone: true }),
  createdAt:    timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:    timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

#### `data` JSON shape (within the `text` column)

```json
{
  "type": "CardNewCommentNotification",
  "userId": 42,
  "status": "sent" | "failed" | "pending",
  "telegramMessage": "...",
  ...additional payload fields
}
```

### User Notification Preferences (JSONB on `users` table)

Stored in `users.notification` as JSONB. Initialized by `initializeUserDefaults()`:

| Preference Key         | Default | Controls |
|------------------------|---------|----------|
| `boardStatusChange`    | `true`  | `BoardStatusChangeNotification` |
| `boardTitleChange`     | `true`  | `BoardInfoChangeNotification` |
| `joinToBoard`          | `true`  | `JoinToBoardNotification` |
| `removeFromBoard`      | `true`  | `RemoveFromBoardNotification` |
| `assignToCard`         | `true`  | `AssignToCardNotification` |
| `removeFromCard`       | `true`  | `RemoveFromCardNotification` |
| `cardStatusChange`     | `true`  | `CardStatusChangeNotification` |
| `cardNewTag`           | `true`  | `CardNewTagNotification` |
| `cardNewComment`       | `true`  | `CardNewCommentNotification` |
| `cardNewAttachment`    | `true`  | `CardNewAttachmentNotification` |
| `cardTitleChange`      | `true`  | `CardInfoChangeNotification` |
| `cardDescriptionChange`| `true`  | `CardDescriptionChangeNotification` |

Always-sent types (no preference key): `CreateNewBoardNotification`, `DeleteBoardNotification`, `CreateNewCardNotification`, `DeleteCardNotification`, `CardUserStatusChangeNotification`.

---

### API Endpoints Consumed

| Endpoint | Method | Auth | Description |
|----------|--------|------|-------------|
| `/api/admin/notification/all` | GET | Admin | Fetch all notification records |
| `/api/admin/notification/table` | POST | Admin | Paginated, sorted notification table |

All notification insertions are internal server-to-server (no external REST call — `NotificationService.send()` is invoked directly from route handlers).

---

### TypeScript Interfaces

```typescript
// shared/types/notification.types.ts (or server-local)

interface NotificationPayload {
  type: string                    // Notification class name
  userId: number                  // Recipient user ID
  data: Record<string, unknown>   // Arbitrary notification data
  preferenceKey?: string          // Optional — user.notification.{key} to check
  telegramMessage?: string        // Formatted MarkdownV2 string for Telegram
}

interface NotificationModel {
  id: string                      // UUID
  type: string                    // Notification class name
  notifiable: UserModel           // Resolved recipient user
  data: string                    // JSON-stringified payload
  read_at: string | null          // ISO timestamp or null
}

// Status field embedded in data JSON
type TelegramDeliveryStatus = 'pending' | 'sent' | 'failed'
```

---

## State Management

This feature is entirely server-side. There is no Pinia store or client state for the notification pipeline itself.

- **PostgreSQL** — durable storage for all notification records.
- **Redis (BullMQ)** — transient job queue for Telegram delivery; jobs are not durably replayed beyond BullMQ's built-in retry.
- **Status field in `data`** — the only mutable field post-insert; updated by the BullMQ worker event handlers.

---

## Notification Pipeline (Flow)

```
Route Handler (e.g., card create)
        │
        ▼
NotificationService.send(payload)
├── preferenceKey provided?
│   ├── YES → read user.notification[key] from DB
│   │          ├── false → SILENT DROP (return early)
│   │          └── true  → continue
│   └── NO  → continue (always-sent)
├── INSERT into notifications table (data includes status: 'pending')
└── user.chat_id present?
    ├── YES → enqueue 'send-telegram' job to BullMQ notifications queue
    └── NO  → done
            │
            ▼
BullMQ Worker (server/jobs/queue.ts)
├── job.name === 'send-telegram'
│   ├── Call Telegram Bot API with MarkdownV2 message
│   ├── On completed → UPDATE notification.data.status = 'sent'
│   └── On failed    → UPDATE notification.data.status = 'failed'
└── job.name === 'telegram-register'
    └── processTelegramRegistration(job.data)
```

---

## Edge Cases

| Scenario | Expected Behaviour |
|----------|-------------------|
| User has no `chat_id` | No BullMQ job enqueued; notification stored in DB only |
| User preference key is `false` | Silent drop — no DB insert, no job, no error thrown |
| BullMQ job fails all retries | `notification.data.status` set to `'failed'`; no re-notification to user |
| `sendToMany` called with empty array | No-op — loop executes zero iterations |
| `sendToMany` called with mixed preferences | Each user evaluated independently; some may be silently dropped |
| Telegram API returns non-2xx | Job marked failed; worker retry logic applies per BullMQ config |
| `data` column content is not valid JSON | Should never happen — always use `JSON.stringify()` before insert |
| `notifiable_id` references a deleted user | Cascade delete removes the notification row automatically |
| `telegramMessage` not provided | No Telegram job is enqueued regardless of `chat_id` |

---

## Non-Goals

- **In-app notification bell / unread count** — covered by a separate client-facing spec.
- **Mark as read endpoint** — out of scope for this spec.
- **Email delivery** — not planned.
- **Notification templates / i18n** — message strings are constructed at the call site.
- **Per-notification retry UI** — no manual retry mechanism; BullMQ handles automatic retries.
- **Telegram bot conversation handling / incoming messages** — covered by `telegram-integration.md`.
- **Client-side polling or WebSocket push for new notifications** — covered by `realtime-centrifugo.md`.

---

## Dependencies

| Dependency | Role |
|------------|------|
| `bullmq` | Job queue and worker for async Telegram delivery |
| `redis` (shared connection) | BullMQ transport layer |
| `drizzle-orm` + PostgreSQL | Notification persistence |
| `server/services/telegram.ts` | Telegram Bot API client and MarkdownV2 formatting |
| `server/utils/response.ts` | `apiResponse` / `apiError` helpers for admin endpoints |
| `server/middleware/admin.ts` | Auth guard on admin notification endpoints |
| `server/services/search-table.ts` | Pagination / sort / filter for `table.post.ts` |
| `users` table (`users.notification` JSONB, `users.chat_id`) | Preference lookup and Telegram routing |
| `notifications` DB schema | Defined in `server/database/schema/notifications.ts` |

### Internal Callers
`NotificationService.send()` and `sendToMany()` are called directly from route handlers in:
- `server/api/admin/board/` (board CRUD events)
- `server/api/client/board/` (join/remove member events)
- `server/api/client/card/` (card lifecycle events)
- `server/api/client/comment/` (`CardNewCommentNotification`)
- `server/api/client/attachment/` (`CardNewAttachmentNotification`)
