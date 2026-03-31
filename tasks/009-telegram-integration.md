# Tasks: Telegram Integration
**Spec:** `specs/telegram-integration.md`
**Status:** pending

---

## Overview

Implements the full Telegram bot integration: a reusable Bot API client with MarkdownV2 formatting helpers, a `telegramRegister` BullMQ job that polls `getUpdates` with a Redis offset to register users via `/start {token}`, and the `connect` / `disconnect` server endpoints. Frontend UI (`ConnectTelegram.vue`, `DisconnectTelegramModal.vue`) is owned by `tasks/020-client-user-profile.md` and is cross-referenced in Wave 2 below.

**Execution order:**
1. Wave 1A — Shared types (no dependencies)
2. Wave 1B — Telegram service (depends on 1A)
3. Wave 1C — BullMQ job (depends on 1B)
4. Wave 1D — Server endpoints (depends on 1B; parallel with 1C)
5. Wave 2 — Frontend (cross-reference to task 020; depends on Wave 1D)
6. Wave 3 — Tests (depends on Wave 1 completion)

---

## Wave 1 — Data Layer (API Agent)

### Task 009-A: Shared TypeScript Interfaces
- **Agent:** API
- **Spec:** `specs/telegram-integration.md`
- **Section:** TypeScript Interfaces
- **Acceptance Criteria:**
  - [ ] `TelegramUser` interface — `id: string`, optional `chat_id?: string`, optional `username?: string`
  - [ ] `TelegramUpdate` interface — `update_id: number`, optional `message` with `chat.id: number`, `from.username?: string`, `text: string`
  - [ ] `ConnectTelegramResponse` interface — `url: string`
- **Dependencies:** `tasks/001-infrastructure-setup.md`, `tasks/002-database-schema.md`
- **Output Files:**
  - `shared/types/telegram.types.ts`

---

### Task 009-B: Telegram Bot API Client + MarkdownV2 Helpers
- **Agent:** API
- **Spec:** `specs/telegram-integration.md`
- **Section:** Bot API Client, MarkdownV2 Helpers, Environment & Config
- **Acceptance Criteria:**
  - [ ] `TELEGRAM_BOT_TOKEN` is read exclusively via `useRuntimeConfig().telegramBotToken` — never hard-coded anywhere in the service
  - [ ] An empty or missing `TELEGRAM_BOT_TOKEN` causes the service to throw a descriptive error at call time (before making any HTTP request)
  - [ ] `sendMessage(chatId: string, text: string, parseMode?: string)` uses `$fetch` to `POST https://api.telegram.org/bot{token}/sendMessage` with `{ chat_id, text, parse_mode }`; `parseMode` defaults to `'MarkdownV2'`
  - [ ] `getUpdates(offset?: number)` uses `$fetch` to `POST https://api.telegram.org/bot{token}/getUpdates` with `{ offset }` and returns `TelegramUpdate[]` (extracted from `result`)
  - [ ] `escape(text: string)` escapes all 18 MarkdownV2 reserved characters: `_  *  [  ]  (  )  ~  \`  >  #  +  -  =  |  {  }  .  !`
  - [ ] `oneLine(text: string)` collapses newline characters and sequences of multiple spaces into a single space
  - [ ] `code(text: string)` wraps the raw text in backtick fences (no escaping applied inside)
  - [ ] `bold(text: string)` first applies `escape()` to the text then wraps in `*...*`
- **Dependencies:** Task 009-A
- **Output Files:**
  - `server/services/telegram.ts`

---

### Task 009-C: BullMQ Queue Registration + `telegramRegister` Job
- **Agent:** API
- **Spec:** `specs/telegram-integration.md`
- **Section:** BullMQ Jobs — `telegramRegister.job.ts` Logic Flow, Connect Flow ACs, Redis Keys
- **Acceptance Criteria:**
  - [ ] A `telegram-register` queue / worker entry is registered in `server/jobs/queue.ts` (add alongside existing `notifications` queue; do not remove existing entries)
  - [ ] The `telegramRegister` worker is initialised in the Nitro server plugin that bootstraps BullMQ workers
  - [ ] Job step 1: reads `telegram:offset` from Redis; defaults to `0` if the key does not exist
  - [ ] Job step 2: calls `getUpdates(offset)` from `server/services/telegram.ts`
  - [ ] Job step 3: for each update where `message.text` starts with `/start `, extracts `token = text.split(' ')[1]`
  - [ ] Job step 4: queries DB for a user where `telegram->>'id' = token` **and** `telegram->>'chat_id' IS NULL` (prevents overwriting already-connected users)
  - [ ] Job step 5 (match found): updates the user record — sets `telegram.chat_id = String(message.chat.id)` and `telegram.username = message.from.username`
  - [ ] Job step 6: updates Redis `telegram:offset` to `max(update_id) + 1` after processing all updates (or leaves it unchanged when the result array is empty)
  - [ ] `/start` messages with an unknown token are silently ignored; the offset still advances
  - [ ] When `getUpdates` throws (Telegram API unreachable), the job fails and BullMQ retries with its configured back-off; no partial offset write occurs
  - [ ] When `getUpdates` returns an empty array, the job exits cleanly without touching the Redis offset
- **Dependencies:** `tasks/001-infrastructure-setup.md`, `tasks/002-database-schema.md`, `tasks/008-notification-system.md`, Task 009-B
- **Output Files:**
  - `server/jobs/telegramRegister.job.ts`
  - `server/jobs/queue.ts` *(update — add `telegram-register` queue registration)*
  - `server/plugins/` *(update whichever Nitro plugin initialises BullMQ workers to register the telegramRegister worker)*

---

### Task 009-D: Connect Endpoint
- **Agent:** API
- **Spec:** `specs/telegram-integration.md`
- **Section:** Token Generation ACs, API Endpoints Consumed
- **Acceptance Criteria:**
  - [ ] Route is `GET /api/client/telegram/connect`; protected by the existing `auth` server middleware
  - [ ] Generates a fresh UUID token using `crypto.randomUUID()` (no external dependency)
  - [ ] Stores the UUID in the authenticated user's `telegram.id` DB field (overwrites any existing value, preserving or initialising `chat_id` / `username` as-is)
  - [ ] Returns `{ data: { url: string } }` where `url` is `https://t.me/{BOT_USERNAME}?start={token}`; `BOT_USERNAME` sourced from `useRuntimeConfig().telegramBotUsername`
  - [ ] Calling this endpoint multiple times each return a new token; previous deep-links become invalid because the stored `telegram.id` is overwritten
- **Dependencies:** `tasks/001-infrastructure-setup.md`, `tasks/002-database-schema.md`, `tasks/004-authentication.md`, Task 009-B
- **Output Files:**
  - `server/api/client/telegram/connect.get.ts`

---

### Task 009-E: Disconnect Endpoint
- **Agent:** API
- **Spec:** `specs/telegram-integration.md`
- **Section:** Disconnect Flow ACs, API Endpoints Consumed
- **Acceptance Criteria:**
  - [ ] Route is `POST /api/client/telegram/disconnect`; protected by the existing `auth` server middleware
  - [ ] Clears `telegram.chat_id` and `telegram.username` from the authenticated user's record (set to `undefined` / omitted)
  - [ ] Preserves `telegram.id` (the connection token) — it is **not** cleared on disconnect
  - [ ] Returns `{ message: string }` (e.g. `"Telegram disconnected successfully"`) with HTTP 200
  - [ ] When called for a user who is not connected (`chat_id` already absent), returns 200 as a no-op — no error
- **Dependencies:** `tasks/001-infrastructure-setup.md`, `tasks/002-database-schema.md`, `tasks/004-authentication.md`
- **Output Files:**
  - `server/api/client/telegram/disconnect.post.ts`

---

## Wave 2 — Frontend (Frontend Agent)

> **Cross-reference — no new task here.**
>
> `ConnectTelegram.vue` (deep-link button + connection status badge) and `DisconnectTelegramModal.vue` (confirmation modal) are fully specified in `specs/client-user-profile.md` sections AC-7 and AC-8. All frontend implementation for the Telegram connect/disconnect UI is owned by **`tasks/020-client-user-profile.md`**.
>
> The Frontend Agent working on task 020 must:
> - Consume `GET /api/client/telegram/connect` (Task 009-D) and `POST /api/client/telegram/disconnect` (Task 009-E)
> - Derive connection state from `user.telegram.chat_id` on the `authStore`
> - Refresh the auth store user object after a successful disconnect so that `chat_id` is cleared in the UI
>
> **Dependency:** Tasks 009-D and 009-E must be complete before task 020 Telegram UI work begins.

---

## Wave 3 — Tests (Testing Agent)

### Task 009-F: Unit Tests — Telegram Service
- **Agent:** Testing
- **Spec:** `specs/telegram-integration.md`
- **Section:** Bot API Client ACs, MarkdownV2 Helpers ACs, Environment & Config ACs
- **Acceptance Criteria:**
  - [ ] `sendMessage` — mocks `$fetch`; asserts correct URL, body `{ chat_id, text, parse_mode }`, and default `parse_mode = 'MarkdownV2'`
  - [ ] `getUpdates` — mocks `$fetch`; asserts correct URL, body `{ offset }`, returns mapped `TelegramUpdate[]`
  - [ ] `getUpdates` with no offset argument — asserts `$fetch` called without `offset` property (or with `undefined`)
  - [ ] `escape` — covers all 18 reserved MarkdownV2 characters each appearing in isolation and combined
  - [ ] `oneLine` — asserts newlines and multiple spaces collapsed to single space
  - [ ] `code` — asserts backtick wrapping without additional escaping
  - [ ] `bold` — asserts `escape()` applied then wrapped in `*...*`
  - [ ] Missing `TELEGRAM_BOT_TOKEN` (empty string) — asserts a descriptive `Error` is thrown before any `$fetch` call
- **Dependencies:** Task 009-B
- **Output Files:**
  - `tests/unit/server/services/telegram.test.ts`

---

### Task 009-G: Unit Tests — `telegramRegister` Job
- **Agent:** Testing
- **Spec:** `specs/telegram-integration.md`
- **Section:** BullMQ Jobs — Logic Flow, Edge Cases
- **Acceptance Criteria:**
  - [ ] Happy path: mocks `getUpdates` returning one `/start {token}` update for an unconnected user; asserts DB update sets `chat_id` + `username` and Redis `telegram:offset` incremented to `update_id + 1`
  - [ ] Unknown token: mocks a `/start unknownToken` update; asserts no DB write; asserts offset still advances
  - [ ] Already-connected user: mocks update matching a user whose `chat_id` is already set; asserts no DB overwrite; asserts offset advances
  - [ ] Empty result: mocks `getUpdates` returning `[]`; asserts no DB write and Redis offset unchanged
  - [ ] Telegram API unreachable: mocks `getUpdates` throwing; asserts the job throws (so BullMQ records a failure); asserts no Redis offset write
  - [ ] Offset initialisation: mocks Redis returning `null`; asserts `getUpdates` called with offset `0`
- **Dependencies:** Task 009-C
- **Output Files:**
  - `tests/unit/server/jobs/telegramRegister.test.ts`

---

### Task 009-H: Unit Tests — Server Endpoints
- **Agent:** Testing
- **Spec:** `specs/telegram-integration.md`
- **Section:** Token Generation ACs, Disconnect Flow ACs, Edge Cases
- **Acceptance Criteria:**
  - [ ] `connect.get.ts` — authenticated user: asserts `crypto.randomUUID()` called, result stored in `telegram.id`, response contains `url` with correct `t.me` deep-link format
  - [ ] `connect.get.ts` — calling twice returns different tokens (UUID changes on each call)
  - [ ] `connect.get.ts` — unauthenticated request returns 401
  - [ ] `disconnect.post.ts` — authenticated user with active connection: asserts `chat_id` and `username` cleared from DB; `id` preserved; response `{ message: string }` with 200
  - [ ] `disconnect.post.ts` — user not currently connected: asserts 200 no-op (no error thrown)
  - [ ] `disconnect.post.ts` — unauthenticated request returns 401
- **Dependencies:** Tasks 009-D, 009-E
- **Output Files:**
  - `tests/unit/server/api/client/telegram/connect.test.ts`
  - `tests/unit/server/api/client/telegram/disconnect.test.ts`
