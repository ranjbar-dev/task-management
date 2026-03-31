# Spec: Admin Notification Logs

## Status: Approved

## Purpose

Provide administrators with a read-only log viewer for all system notifications. Admins can browse, filter, sort, and paginate through the full notification history — including delivery status (pending/sent/failed via Telegram), recipient, notification type, and read state. No create, update, or delete operations are available from this view.

---

## User Stories

- **As an admin**, I want to see a paginated table of all system notifications so I can audit what was sent and to whom.
- **As an admin**, I want to filter notifications by type so I can focus on a specific class of events (e.g., board vs. card notifications).
- **As an admin**, I want to filter by recipient user so I can troubleshoot notification delivery for a specific account.
- **As an admin**, I want to filter by read/unread status so I can identify notifications that users have not yet acknowledged.
- **As an admin**, I want to see the Telegram delivery status (pending/sent/failed) for each notification so I can identify delivery failures.
- **As an admin**, I want the table to default to most-recent-first ordering so the latest activity is immediately visible.

---

## Acceptance Criteria

### AC-1: List All Notifications (`GET /api/admin/notification/all`)
- [ ] Endpoint returns all notifications without pagination.
- [ ] Response is an array of transformed notification objects using `transformNotification`.
- [ ] Requires `auth`, `active-user`, and `admin` middleware (role === `UserRoleEnum.AdminUser`).
- [ ] Returns 403 if the authenticated user is not an admin.
- [ ] Returns 401 if no valid JWT is present.

### AC-2: Paginated Notification Table (`POST /api/admin/notification/table`)
- [ ] Endpoint accepts a SearchTable request body and returns a paginated response.
- [ ] Default sort is `created_at` DESC (most recent first).
- [ ] Results are filterable by `type` (exact or partial match).
- [ ] Results are filterable by `notifiable_id` (recipient user ID).
- [ ] Results are filterable by `read_at` (datetime-bool: `null` → unread, non-null → read).
- [ ] Pagination metadata (total, page, per_page, last_page) is included in the response envelope.
- [ ] Requires `auth`, `active-user`, and `admin` middleware.
- [ ] Returns 422 if the request body fails Zod validation.

### AC-3: Notification Log Viewer Page (`app/pages/notification/index.vue`)
- [ ] Page is accessible only to authenticated admin users; non-admins are redirected.
- [ ] Uses the `default` layout.
- [ ] Renders a searchable, paginated table using the `Table/Search` UI component.
- [ ] Table displays columns: **Type**, **Recipient**, **Delivery Status**, **Read At**, **Created At**.
- [ ] Delivery status is parsed from the `data` JSON field and displayed as a badge/chip: `pending` (neutral), `sent` (success), `failed` (destructive).
- [ ] `read_at` is displayed as a formatted datetime string when present, or "Unread" when `null`.
- [ ] `created_at` is displayed using the transformer format `"Y-m-d H:i:s"`.
- [ ] Filter controls allow filtering by notification type (dropdown of 18 types), recipient user, and read/unread status.
- [ ] Table sorts client-side or re-fetches on column header click; `created_at` is the default sort with DESC direction.
- [ ] No create, edit, or delete actions are exposed anywhere on the page.
- [ ] Loading state is shown while the table data is fetching.
- [ ] Empty state is shown when no notifications match the current filters.

### AC-4: Delivery Status Display
- [ ] The `data` field is parsed as JSON on the client; the `status` property is extracted.
- [ ] If `status` is missing or unrecognised, the cell renders "Unknown".
- [ ] Status values map to distinct visual indicators:
  - `pending` → neutral/grey badge
  - `sent` → green/success badge
  - `failed` → red/destructive badge

### AC-5: Notification Type Coverage
- [ ] All 18 notification types are recognised and displayable without error:
  - Board: `CreateNewBoardNotification`, `BoardStatusChangeNotification`, `BoardInfoChangeNotification`, `JoinToBoardNotification`, `RemoveFromBoardNotification`, `DeleteBoardNotification`
  - Card: `CreateNewCardNotification`, `AssignToCardNotification`, `RemoveFromCardNotification`, `DeleteCardNotification`, `CardUserStatusChangeNotification`, `CardStatusChangeNotification`, `CardNewTagNotification`, `CardNewCommentNotification`, `CardNewAttachmentNotification`, `CardInfoChangeNotification`, `CardDescriptionChangeNotification`

---

## Component Tree / File Structure

```
server/api/admin/notification/
├── all.get.ts          # GET  /api/admin/notification/all
└── table.post.ts       # POST /api/admin/notification/table

app/pages/notification/
└── index.vue           # Admin notification log viewer (layout: default)

app/composables/api/
└── use-admin-notification-api.ts   # API composable for notification endpoints

shared/schemas/
└── notification.schema.ts          # Zod schema for SearchTable request body (if not already shared)
```

> **Note:** No new Pinia store is required; table state is managed locally in the composable/page.

---

## Data Model

### Database Schema

```typescript
// server/database/schema/notifications.ts
export const notifications = pgTable('notifications', {
  id:           uuid('id').primaryKey().defaultRandom(),
  type:         varchar('type', { length: 255 }).notNull(),
  notifiableId: integer('notifiable_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  data:         text('data').notNull(),            // JSON-stringified payload; contains `status` field
  readAt:       timestamp('read_at', { withTimezone: true }),
  createdAt:    timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:    timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

### `data` JSON Payload Shape

```typescript
interface NotificationData {
  status?: 'pending' | 'sent' | 'failed'  // Telegram delivery status
  // Additional type-specific fields (not displayed in the admin table)
}
```

Delivery status lifecycle:
- `pending` — default value set when the notification row is created, before the BullMQ Telegram job runs.
- `sent` — updated by the job upon successful Telegram API delivery.
- `failed` — updated by the job when Telegram delivery throws or returns an error.

### Transformer

```typescript
// server/transformers/notification.transformer.ts
transformNotification(row) // Returns: { id, type, data, read_at, created_at }
// created_at formatted as "Y-m-d H:i:s"
// data is returned as raw JSON string — client must JSON.parse() to read status
```

---

### API Endpoints Consumed

| Method | Endpoint | Request | Response |
|--------|----------|---------|----------|
| `GET` | `/api/admin/notification/all` | — | `{ data: NotificationModel[] }` |
| `POST` | `/api/admin/notification/table` | `SearchTableRequest` | `{ data: NotificationModel[], meta: PaginationMeta }` |

#### SearchTable Request Body (POST `/table`)

```typescript
interface SearchTableRequest {
  page: number           // 1-based
  per_page: number       // rows per page
  sort?: {
    column: string       // e.g. "created_at"
    direction: 'asc' | 'desc'
  }
  filters?: {
    type?: string                   // notification type string
    notifiable_id?: number          // recipient user ID
    read_at?: boolean | null        // true = read, false/null = unread
  }
}
```

#### Pagination Meta

```typescript
interface PaginationMeta {
  total: number
  per_page: number
  current_page: number
  last_page: number
}
```

---

### TypeScript Interfaces

```typescript
// shared/types/notification.types.ts

interface NotificationModel {
  id: string               // UUID
  type: string             // One of the 18 notification type strings
  notifiable: UserModel    // Recipient user (joined/expanded from notifiable_id)
  data: string             // Raw JSON string — parse to read NotificationData
  read_at: string | null   // Formatted datetime or null
  created_at: string       // "Y-m-d H:i:s"
}

interface NotificationData {
  status?: 'pending' | 'sent' | 'failed'
  // Additional type-specific fields (board id, card id, etc.)
}
```

> `UserModel` is the existing shared user interface; it must be resolved from the `notifiable_id` foreign key in the transformer or via a join in the query.

---

## State Management

No dedicated Pinia store is needed. The page uses local reactive state via the API composable:

```typescript
// app/composables/api/use-admin-notification-api.ts
export function useAdminNotificationApi() {
  async function fetchAll(): Promise<NotificationModel[]>
  async function fetchTable(params: SearchTableRequest): Promise<{ data: NotificationModel[], meta: PaginationMeta }>
  return { fetchAll, fetchTable }
}
```

Page-level state (`ref`/`reactive`):
- `notifications` — current page rows
- `pagination` — current `PaginationMeta`
- `filters` — active filter values
- `sort` — active sort column + direction
- `loading` — boolean fetch guard

---

## Edge Cases

| Scenario | Expected Behaviour |
|----------|-------------------|
| `data` field is not valid JSON | Display "Unknown" in the Delivery Status cell; do not throw |
| Notification `type` is not in the known 18-type list | Render the raw string as-is; no error |
| `notifiable` user has been deleted | Cascade delete removes the notification row; row should not appear |
| `read_at` is `null` | Display "Unread" in the Read At column |
| `status` key missing from `data` JSON | Display "Unknown" delivery status badge |
| Empty result set after filtering | Show empty-state message (e.g., "No notifications found") |
| Network or server error on table fetch | Show an error toast; preserve the previous table state |
| Non-admin user accesses `/notification` | Redirect to home page via route middleware |

---

## Non-Goals

- **No mutation operations** — admins cannot mark notifications as read, resend, or delete them from this interface.
- **No notification creation** — notifications are generated by the server (NotificationService) only; no admin-driven creation.
- **No real-time updates** — the log viewer is a static paginated view; live push of new notifications is out of scope.
- **No bulk actions** — no select-all, bulk delete, or bulk export.
- **No notification content preview** — the `data` JSON body details (beyond `status`) are not rendered; only the status field is surfaced.
- **No per-row drill-down page** — the table is the complete view; no notification detail page is required.

---

## Dependencies

| Dependency | Role |
|------------|------|
| `search-table-service.md` | SearchTable service used by the `/table` endpoint for filtering, sorting, and pagination |
| `api-transformers.md` | `transformNotification()` transformer shapes every row before returning |
| `authentication.md` | `auth`, `active-user`, and `admin` server middleware gate both endpoints |
| `database-schema.md` | `notifications` table schema and its relation to `users` |
| `notification-system.md` | Defines the 18 notification types and the `status` lifecycle in the `data` field |
| `validation-schemas.md` | SearchTable Zod schema used for request body validation on the `/table` endpoint |
| `UserModel` | Shared interface for the resolved `notifiable` recipient in `NotificationModel` |
