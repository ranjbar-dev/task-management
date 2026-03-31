# Spec: Telegram Integration

## Status: Approved

---

## Purpose

Enable users to connect their Telegram account to the application so that in-app notifications are delivered to them via a Telegram bot. The integration covers token-based device linking (connect flow), account unlinking (disconnect flow), a BullMQ polling job that registers `/start` commands from the Telegram Bot API, a reusable Bot API client, and MarkdownV2 message formatting helpers.

---

## User Stories

- **As a user**, I want to connect my Telegram account so that I receive task notifications directly in Telegram.
- **As a user**, I want a one-click deep-link so that I can start the bot without manually entering a token.
- **As a user**, I want to disconnect Telegram whenever I choose without losing the ability to reconnect later.
- **As a developer**, I want a reusable Telegram service so that any server-side code can send formatted Telegram messages in one call.

---

## Acceptance Criteria

### Token Generation
- [ ] Every user record has a `telegram.id` field populated with a 12-character random string upon account creation (seeder responsibility).
- [ ] `GET /api/client/telegram/connect` generates a fresh UUID token via `crypto.randomUUID()`, stores it in the user's `telegram.id` DB field, and returns a deep-link URL.
- [ ] The returned URL follows the format `https://t.me/{BOT_USERNAME}?start={token}` where `BOT_USERNAME` is derived from runtime config.

### Connect Flow
- [ ] The deep-link URL is displayed to the user in `ConnectTelegram.vue` as a clickable link / button.
- [ ] When the user clicks the link and sends `/start {token}` to the bot, the `telegram-register` BullMQ job detects it within the next polling cycle.
- [ ] The job calls `https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/getUpdates` with the cached Redis offset to fetch only new updates.
- [ ] The job filters updates for messages whose text matches `/start {token}` against any stored user token.
- [ ] On match, the job updates the user DB record: `telegram.chat_id = message.chat.id` (as string) and `telegram.username = message.from.username`.
- [ ] The Redis offset key (`telegram:offset`) is updated to `max(update_id) + 1` after each successful poll to prevent reprocessing.

### Disconnect Flow
- [ ] `POST /api/client/telegram/disconnect` clears `telegram.chat_id` and `telegram.username` from the user's record.
- [ ] `telegram.id` (the connection token) is **preserved** after disconnection to allow future reconnection without regenerating a base token.
- [ ] The UI shows a confirmation modal before dispatching the disconnect request.
- [ ] After disconnection, `ConnectTelegram.vue` reverts to the "connect" state.

### Message Delivery
- [ ] When a notification is inserted into the DB and the user has `telegram.chat_id` set, `NotificationService` enqueues a `send-telegram` BullMQ job.
- [ ] The `send-telegram` job calls `sendMessage(chatId, text, 'MarkdownV2')` from `server/services/telegram.ts`.
- [ ] Messages use MarkdownV2 formatting helpers (`bold`, `code`, `escape`, `oneLine`) to prevent Telegram parse errors.

### Bot API Client
- [ ] `sendMessage` accepts `chatId: string`, `text: string`, and an optional `parseMode` (defaults to `'MarkdownV2'`).
- [ ] `getUpdates` accepts an optional numeric `offset` and returns `TelegramUpdate[]`.
- [ ] Both functions use `$fetch` for HTTP requests, targeting the base URL `https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/`.

### MarkdownV2 Helpers
- [ ] `escape(text)` escapes all MarkdownV2 reserved characters (`_`, `*`, `[`, `]`, `(`, `)`, `~`, `` ` ``, `>`, `#`, `+`, `-`, `=`, `|`, `{`, `}`, `.`, `!`).
- [ ] `oneLine(text)` collapses newlines and multiple spaces into a single line.
- [ ] `code(text)` wraps text in backtick fences for inline code.
- [ ] `bold(text)` wraps escaped text in `*...*`.

### Environment & Config
- [ ] `TELEGRAM_BOT_TOKEN` environment variable is read via Nuxt runtime config (`useRuntimeConfig().telegramBotToken`) — never hard-coded.
- [ ] Missing or empty `TELEGRAM_BOT_TOKEN` causes the service to throw a descriptive startup error.

---

## Component Tree / File Structure

```
server/
├── api/
│   └── client/
│       └── telegram/
│           ├── connect.get.ts          # GET  /api/client/telegram/connect
│           └── disconnect.post.ts      # POST /api/client/telegram/disconnect
├── services/
│   └── telegram.ts                     # Bot API client + MarkdownV2 helpers
└── jobs/
    ├── queue.ts                         # BullMQ queue definitions (telegram-register)
    └── telegramRegister.job.ts         # Polls getUpdates, matches tokens, updates DB

app/
└── components/
    └── ConnectTelegram.vue             # Connect UI — deep-link display + disconnect modal
```

---

## Data Model

### DB Field — `users.telegram`

```typescript
// server/database/schema/users.ts
telegram: jsonb('telegram').$type<{
  id: string         // 12-char random token; generated at user creation; preserved on disconnect
  chat_id?: string   // Telegram numeric chat ID (stored as string); set after /start handshake
  username?: string  // Telegram @username; set after /start handshake
}>()
```

### Redis Keys

| Key | Type | TTL | Description |
|-----|------|-----|-------------|
| `telegram:offset` | String | None | Last processed `update_id + 1` for incremental `getUpdates` polling |

---

## API Endpoints Consumed

| Method | Endpoint | Request | Response |
|--------|----------|---------|----------|
| `GET` | `/api/client/telegram/connect` | — | `{ url: string }` (deep-link URL) |
| `POST` | `/api/client/telegram/disconnect` | — | `{ message: string }` |

### External API (Telegram Bot API)

| Method | URL | Body | Response |
|--------|-----|------|----------|
| `POST` | `https://api.telegram.org/bot{token}/getUpdates` | `{ offset?: number }` | `{ ok: boolean, result: TelegramUpdate[] }` |
| `POST` | `https://api.telegram.org/bot{token}/sendMessage` | `{ chat_id, text, parse_mode }` | Telegram message object |

---

## TypeScript Interfaces

```typescript
// shared/types/telegram.types.ts

interface TelegramUser {
  id: string        // internal connection token (preserved across connect/disconnect cycles)
  chat_id?: string  // Telegram chat ID; undefined when not connected
  username?: string // Telegram @username; undefined when not connected
}

interface TelegramUpdate {
  update_id: number
  message?: {
    chat: { id: number }
    from: { username?: string }
    text: string
  }
}

interface ConnectTelegramResponse {
  url: string  // e.g., "https://t.me/MyTaskBot?start=abc123-uuid-token"
}
```

---

## State Management

No dedicated Pinia store is required. The connection state (is Telegram connected?) is derived from `user.telegram.chat_id` in the existing `auth` store or user profile composable.

**ConnectTelegram.vue** manages local component state:

| State | Type | Description |
|-------|------|-------------|
| `deepLinkUrl` | `string \| null` | The generated bot deep-link URL; null until connect is called |
| `isConnecting` | `boolean` | True while `GET /connect` is in-flight |
| `showDisconnectModal` | `boolean` | Controls disconnect confirmation modal visibility |
| `isDisconnecting` | `boolean` | True while `POST /disconnect` is in-flight |

After a successful disconnect, the parent page/profile store should refresh the user object so that `telegram.chat_id` is cleared.

---

## BullMQ Jobs

| Job Name | Queue | Trigger | Description |
|----------|-------|---------|-------------|
| `send-telegram` | `notifications` | NotificationService after DB insert | Sends formatted message to `chat_id` via `sendMessage()` |
| `telegram-register` | `notifications` | Scheduled / recurring | Polls `getUpdates` with Redis offset; matches `/start {token}` to a user; writes `chat_id` + `username` to DB |

### `telegramRegister.job.ts` — Logic Flow

```
1. Read offset from Redis (key: telegram:offset); default 0
2. Call getUpdates(offset)
3. For each update where message.text starts with "/start ":
   a. Extract token = text.split(" ")[1]
   b. Query DB: SELECT user WHERE telegram->>'id' = token AND telegram->>'chat_id' IS NULL
   c. If found: UPDATE user SET telegram = { id: token, chat_id: message.chat.id, username: message.from.username }
4. Update offset = max(update_id) + 1 → store in Redis
```

---

## Edge Cases

| Scenario | Expected Behaviour |
|----------|--------------------|
| User clicks connect multiple times | Each call overwrites `telegram.id` with a new UUID; old deep-links become invalid |
| `/start` message with an unknown token | Silently ignored; no DB write; offset still advances |
| `/start` message for already-connected user | Ignored (query filters `chat_id IS NULL`); prevents duplicate writes |
| Telegram API returns no updates | Job exits cleanly; Redis offset unchanged |
| Telegram API is unreachable | `getUpdates` throws; job fails; BullMQ retries with back-off |
| `sendMessage` fails (invalid `chat_id`) | `send-telegram` job fails; BullMQ retries; no notification re-enqueue from application layer |
| User disconnects mid-polling cycle | `chat_id` is cleared; subsequent `send-telegram` jobs check `chat_id` before calling `sendMessage` |
| MarkdownV2 unescaped special char in notification text | `escape()` must be applied before calling `sendMessage`; unescaped text causes Telegram 400 error |
| Empty `TELEGRAM_BOT_TOKEN` | Service throws at call time with a descriptive error |

---

## Non-Goals

- OAuth or Telegram Login Widget flow (only bot-based token pairing is in scope).
- Sending Telegram messages outside the `notifications` queue (e.g., inline from route handlers).
- Two-way bot commands beyond `/start` registration (e.g., `/stop`, reply handling).
- Telegram group or channel messaging (only private `chat_id` to individual users).
- Webhook-based bot updates (polling via `getUpdates` is the chosen approach).
- UI for customising which notification types are delivered via Telegram (that is part of the `notification-system` spec).

---

## Dependencies

| Dependency | Role |
|------------|------|
| `BullMQ` | Job queues for `send-telegram` and `telegram-register` |
| `Redis` | Stores `telegram:offset` for incremental polling |
| `$fetch` (Nitro built-in) | HTTP client for Telegram Bot API calls |
| `crypto.randomUUID()` | Token generation in the connect endpoint |
| [notification-system spec](./notification-system.md) | Defines when `send-telegram` jobs are enqueued |
| [database-schema spec](./database-schema.md) | Defines `users.telegram` JSONB column |
| [client-user-profile spec](./client-user-profile.md) | Hosts the `ConnectTelegram.vue` component on the profile page |
| `TELEGRAM_BOT_TOKEN` env var | Required at server startup; read via `useRuntimeConfig()` |
