# Tasks: Notification System
**Spec:** `specs/notification-system.md`
**Status:** pending

---

## Overview

Implement the complete server-side notification pipeline: preference-gated DB persistence, BullMQ async Telegram delivery with status tracking, all 18 notification type payloads (6 board + 12 card), user preference initialisation, and two admin read endpoints. There is **no frontend UI in this spec** — notification preference toggles are handled by `tasks/020-client-user-profile.md`.

---

## Wave 1 — Data Layer (API Agent)

### Task 008-A: NotificationService — Core Pipeline & Notification Types

- **Agent:** API
- **Spec:** `specs/notification-system.md`
- **Section:** AC-1 (Preference Checking), AC-2 (Database Insert), AC-3 (BullMQ Enqueue), AC-5 (sendToMany), AC-6 (Board Notifications — 6 Types), AC-7 (Card Notifications — 12 Types); Component Tree (`server/services/notification.ts`); Data Model (`NotificationPayload`, `NotificationModel`, `TelegramDeliveryStatus`); Notification Pipeline Flow; Edge Cases
- **Acceptance Criteria:**
  - [ ] AC-1: `NotificationService.send()` reads `user.notification[preferenceKey]` from the database when a `preferenceKey` is provided.
  - [ ] AC-1: If the resolved preference value is `false`, the notification is silently dropped — no DB insert, no BullMQ job, no error thrown.
  - [ ] AC-1: If `preferenceKey` is `undefined` or `null`, the notification is always processed (no preference check performed).
  - [ ] AC-2: All non-dropped notifications are inserted into the `notifications` table with `data` as a JSON-stringified payload (initial `status: 'pending'`).
  - [ ] AC-2: The `type` column stores the notification class name string (e.g., `'CreateNewBoardNotification'`).
  - [ ] AC-2: The `notifiable_id` column stores the recipient's user ID; `read_at` defaults to `null` on creation.
  - [ ] AC-3: After a successful DB insert, if the recipient user has a `chat_id`, a `send-telegram` job is enqueued to the `notifications` BullMQ queue with `{ telegramMessage, chat_id }` as payload.
  - [ ] AC-3: If the user has no `chat_id` or `telegramMessage` is not provided, no BullMQ job is enqueued.
  - [ ] AC-5: `NotificationService.sendToMany(users, payload)` iterates the user array and calls the full `send()` pipeline per user, populating `userId` from each individual `User` object.
  - [ ] AC-5: `sendToMany` called with an empty array is a no-op — zero iterations, no errors.
  - [ ] AC-5: Each user in `sendToMany` is evaluated independently; users with preference `false` are silently dropped while others proceed.
  - [ ] AC-6: `CreateNewBoardNotification` — no `preferenceKey`; always sent; triggered on board creation.
  - [ ] AC-6: `BoardStatusChangeNotification` — `preferenceKey: 'boardStatusChange'`; triggered when board is activated or deactivated.
  - [ ] AC-6: `BoardInfoChangeNotification` — `preferenceKey: 'boardTitleChange'`; triggered when board name or description changes.
  - [ ] AC-6: `JoinToBoardNotification` — `preferenceKey: 'joinToBoard'`; triggered when a user is added to a board.
  - [ ] AC-6: `RemoveFromBoardNotification` — `preferenceKey: 'removeFromBoard'`; triggered when a user is removed from a board.
  - [ ] AC-6: `DeleteBoardNotification` — no `preferenceKey`; always sent; triggered on board deletion.
  - [ ] AC-7: `CreateNewCardNotification` — no `preferenceKey`; always sent; triggered on card creation.
  - [ ] AC-7: `AssignToCardNotification` — `preferenceKey: 'assignToCard'`; triggered when a user is assigned to a card.
  - [ ] AC-7: `RemoveFromCardNotification` — `preferenceKey: 'removeFromCard'`; triggered when a user is removed from a card.
  - [ ] AC-7: `DeleteCardNotification` — no `preferenceKey`; always sent; triggered on card deletion.
  - [ ] AC-7: `CardUserStatusChangeNotification` — no `preferenceKey`; always sent; triggered when a user's status on a card changes.
  - [ ] AC-7: `CardStatusChangeNotification` — `preferenceKey: 'cardStatusChange'`; triggered when card status changes.
  - [ ] AC-7: `CardNewTagNotification` — `preferenceKey: 'cardNewTag'`; triggered when tags are added or removed from a card.
  - [ ] AC-7: `CardNewCommentNotification` — `preferenceKey: 'cardNewComment'`; triggered on new comment (includes code block rendering in `telegramMessage`).
  - [ ] AC-7: `CardNewAttachmentNotification` — `preferenceKey: 'cardNewAttachment'`; triggered when a file is attached.
  - [ ] AC-7: `CardInfoChangeNotification` — `preferenceKey: 'cardTitleChange'`; triggered when card name changes.
  - [ ] AC-7: `CardDescriptionChangeNotification` — `preferenceKey: 'cardDescriptionChange'`; triggered when card description changes.
  - [ ] `NotificationPayload`, `NotificationModel`, and `TelegramDeliveryStatus` types are defined and exported (in `shared/types/notification.types.ts` or server-local).
  - [ ] `data` column content is always written via `JSON.stringify()` — no raw object inserts.
- **Dependencies:** `tasks/001-infrastructure-setup.md`, `tasks/002-database-schema.md`
- **Output Files:**
  - `server/services/notification.ts`
  - `shared/types/notification.types.ts`

---

### Task 008-B: BullMQ Queue, Worker & Nitro Plugin

- **Agent:** API
- **Spec:** `specs/notification-system.md`
- **Section:** AC-4 (Telegram Delivery), AC-9 (BullMQ Worker Initialization); Component Tree (`server/jobs/queue.ts`, `server/jobs/notification.job.ts`, `server/plugins/bullmq.ts`); Notification Pipeline Flow; Edge Cases (job failure, non-2xx Telegram response)
- **Acceptance Criteria:**
  - [ ] AC-4: The BullMQ worker processes `send-telegram` jobs by calling the Telegram Bot API via `server/services/telegram.ts`.
  - [ ] AC-4: Telegram messages are formatted with MarkdownV2 using helpers: `escape()`, `oneLine()`, `code()`, `bold()` (defined in `server/services/telegram.ts`).
  - [ ] AC-4: On job `completed` event, `notification.data.status` is updated to `'sent'` in PostgreSQL.
  - [ ] AC-4: On job `failed` event (including all retry exhaustion), `notification.data.status` is updated to `'failed'` in PostgreSQL.
  - [ ] AC-9: `server/plugins/bullmq.ts` (Nitro plugin) calls `initializeNotificationWorker()` on server startup.
  - [ ] AC-9: The worker listens on the `notifications` queue and handles at minimum the `send-telegram` and `telegram-register` job names.
  - [ ] AC-9: Both the BullMQ Queue and Worker use the shared `redisConnection` instance defined in `server/jobs/queue.ts`.
  - [ ] `server/jobs/queue.ts` exports the named `notifications` Queue, the Worker constructor/factory, and the `redisConnection` instance.
  - [ ] `server/jobs/notification.job.ts` exports `initializeNotificationWorker()` — the function that wires up all job name handlers and event listeners on the worker.
  - [ ] Telegram API non-2xx responses are treated as job failures; BullMQ retry logic applies per its default configuration.
- **Dependencies:** `tasks/008-A`, `tasks/001-infrastructure-setup.md`
- **Note:** `server/services/telegram.ts` (Telegram Bot API client) is a shared dependency also covered by `tasks/009-telegram-integration.md`. Coordinate with that task to avoid duplicate implementation. If `009` has not yet been executed, stub the Telegram client call in this task and finalise in `009`.
- **Output Files:**
  - `server/jobs/queue.ts`
  - `server/jobs/notification.job.ts`
  - `server/plugins/bullmq.ts`
  - `server/services/telegram.ts` *(shared with tasks/009 — implement MarkdownV2 helpers and sendMessage here if not yet created)*

---

### Task 008-C: User Notification Preference Defaults

- **Agent:** API
- **Spec:** `specs/notification-system.md`
- **Section:** AC-8 (User Notification Preferences — Defaults); Data Model → User Notification Preferences table
- **Acceptance Criteria:**
  - [ ] AC-8: `initializeUserDefaults()` sets all 12 preference keys to `true` in the `users.notification` JSONB column on user creation.
  - [ ] AC-8: The 12 keys initialised are: `boardStatusChange`, `boardTitleChange`, `joinToBoard`, `removeFromBoard`, `assignToCard`, `removeFromCard`, `cardStatusChange`, `cardNewTag`, `cardNewComment`, `cardNewAttachment`, `cardTitleChange`, `cardDescriptionChange`.
  - [ ] AC-8: `initializeUserDefaults()` also sets `users.telegram.id` to a randomly generated 12-character token (per `tasks/002-database-schema.md` AC-4 observer contract).
  - [ ] Preference keys not in the 12-key list have no effect on always-sent notification types — no additional keys are added.
  - [ ] `initializeUserDefaults()` is called from the user-creation route handler (admin or auth) — the function itself must not assume the caller context.
- **Dependencies:** `tasks/002-database-schema.md`, `tasks/001-infrastructure-setup.md`
- **Output Files:**
  - `server/utils/user-defaults.ts` *(or co-located in `server/services/notification.ts` — implementer's discretion; must be importable from user-creation route handlers)*

---

### Task 008-D: Admin Notification Endpoints

- **Agent:** API
- **Spec:** `specs/notification-system.md`
- **Section:** AC-10 (Admin Endpoints); Component Tree (`server/api/admin/notification/`); Data Model → `NotificationModel`; API Endpoints Consumed table
- **Acceptance Criteria:**
  - [ ] AC-10: `GET /api/admin/notification/all` returns all notification records; protected by admin middleware.
  - [ ] AC-10: `POST /api/admin/notification/table` returns a paginated, date-descending list of notifications using `SearchTableService`; protected by admin middleware.
  - [ ] Both endpoints use `apiResponse` / `apiError` helpers from `server/utils/response.ts`.
  - [ ] Both endpoints apply the `admin` server middleware guard (import or rely on route-level middleware resolution).
  - [ ] `table.post.ts` accepts a SearchTable request body validated by the shared `searchTableSchema`.
  - [ ] Notification records in both responses are passed through a transformer before returning (maps `data` text column to a parsed object in the response shape).
- **Dependencies:** `tasks/008-A`, `tasks/006-search-table-service.md`, `tasks/005-api-transformers.md`, `tasks/004-authentication.md`
- **Output Files:**
  - `server/api/admin/notification/all.get.ts`
  - `server/api/admin/notification/table.post.ts`

---

## Wave 2 — Frontend (Frontend Agent)

> **Not in scope for this spec.**
> The spec explicitly states: *"No client-side components are in scope for this spec. Notification reads and in-app display are covered by a separate spec."* and *"There is no Pinia store or client state for the notification pipeline itself."*
>
> Notification preference toggle UI (the 12 preference keys) is handled in **`tasks/020-client-user-profile.md`**.

---

## Wave 3 — Tests (Testing Agent)

### Task 008-E: Notification System Tests

- **Agent:** Testing
- **Spec:** `specs/notification-system.md`
- **Section:** All ACs (AC-1 through AC-10); Edge Cases table; Notification Pipeline Flow
- **Acceptance Criteria:**
  - [ ] Unit test: `NotificationService.send()` with a `preferenceKey` that is `false` → no DB insert, no BullMQ job enqueued.
  - [ ] Unit test: `NotificationService.send()` with no `preferenceKey` → always inserts into DB regardless of user preferences.
  - [ ] Unit test: `NotificationService.send()` with user having a `chat_id` and a `telegramMessage` → BullMQ `add()` is called with correct job name and payload.
  - [ ] Unit test: `NotificationService.send()` with user having no `chat_id` → BullMQ `add()` is NOT called.
  - [ ] Unit test: `NotificationService.send()` with `telegramMessage` not provided → BullMQ `add()` is NOT called even when user has `chat_id`.
  - [ ] Unit test: `NotificationService.sendToMany()` with an empty array → no DB inserts, no BullMQ jobs, no errors thrown.
  - [ ] Unit test: `NotificationService.sendToMany()` with mixed preferences → only users with `true` preference (or no preference key) result in inserts.
  - [ ] Unit test: BullMQ worker `completed` event handler → `notification.data.status` updated to `'sent'` in DB.
  - [ ] Unit test: BullMQ worker `failed` event handler → `notification.data.status` updated to `'failed'` in DB.
  - [ ] Unit test: `initializeUserDefaults()` → all 12 preference keys present and set to `true`; `telegram.id` is a 12-character string.
  - [ ] Integration test: DB insert produces a row with correct `type`, `notifiable_id`, `data` (valid JSON-stringified payload with `status: 'pending'`), and `read_at: null`.
  - [ ] Integration test: `GET /api/admin/notification/all` returns 200 with notification array (admin JWT required); returns 401/403 for non-admin.
  - [ ] Integration test: `POST /api/admin/notification/table` returns paginated results sorted by date descending (admin JWT required).
  - [ ] Edge case test: `notifiable_id` cascade — verify notification rows are deleted when the referenced user is deleted.
- **Dependencies:** `tasks/008-A`, `tasks/008-B`, `tasks/008-C`, `tasks/008-D`
- **Output Files:**
  - `tests/unit/server/services/notification.test.ts`
  - `tests/unit/server/jobs/notification.test.ts`
  - `tests/unit/server/utils/user-defaults.test.ts`
  - `tests/e2e/admin/notification.spec.ts`
