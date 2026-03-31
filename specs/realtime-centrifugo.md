# Spec: Real-Time System — Centrifugo WebSocket Integration

## Status: Approved

---

## Purpose

Provide a real-time collaboration layer for the task management application using Centrifugo as a WebSocket message broker. All CRUD operations (boards, lists, cards, comments, attachments, logs) must broadcast events to subscribed clients immediately after a successful database write. The frontend maintains a persistent WebSocket connection and reactively updates Pinia stores without requiring manual page refreshes.

---

## User Stories

- **As a board member**, I want to see cards added, moved, or deleted by other users in real time without refreshing the page.
- **As a board member**, I want to see list changes (create, rename, reorder, delete) reflected immediately on the kanban board.
- **As a card assignee**, I want comments, attachments, and activity logs to appear in the card modal in real time while it is open.
- **As a board member**, I want to see user assignments to cards appear instantly without a page reload.
- **As an admin**, I want to be authorized to subscribe to any board or card channel regardless of direct board membership.
- **As a developer**, I want a single `broadcastToCentrifugo()` utility that all API route handlers can call after a database write without duplicating connection logic.
- **As a developer**, I want the WebSocket connection and all subscriptions to be cleanly torn down when navigating away from a board or closing a card modal, preventing memory leaks.

---

## Acceptance Criteria

### AC-1: Server — Centrifugo Service (`publishToCentrifugo`)
- [ ] `server/services/centrifugo.ts` exports an async `publishToCentrifugo(payload: BroadcastPayload)` function.
- [ ] The function issues a `POST` request to `${CENTRIFUGO_API_URL}/api/publish` using `$fetch`.
- [ ] The `Authorization` header uses the `apikey` scheme with the `CENTRIFUGO_API_KEY` env var.
- [ ] The request body is `{ channel: payload.channel, data: { event: payload.event, ...payload.data } }`.
- [ ] On HTTP error from Centrifugo, the error is logged server-side but **does not** throw to the calling route handler (broadcast failures are non-fatal).

### AC-2: Server — Broadcast Utility (`broadcastToCentrifugo`)
- [ ] `server/utils/broadcast.ts` exports an async `broadcastToCentrifugo(channelType, channelId, event, data)` function.
- [ ] Channel string is constructed as `${channelType}#${channelId}` (e.g., `board#1`, `card#5`).
- [ ] Delegates to `publishToCentrifugo()` with the constructed channel and provided event/data.
- [ ] Accepted `channelType` values are `'board'` and `'card'` only (TypeScript union type enforced).

### AC-3: Server — Channel Authorization Endpoint
- [ ] File: `server/api/auth/centrifugo/channel.post.ts` → `POST /api/auth/centrifugo/channel`.
- [ ] Request body validated with Zod: `z.object({ channel: z.string().min(1) })`.
- [ ] Accepts channels matching `board#{id}` or `card#{cardId}` format; rejects any other format with `400`.
- [ ] For `board#{id}` channels: user must be an active board member **or** an admin.
- [ ] For `card#{cardId}` channels: the parent board membership is checked (same rule — member or admin).
- [ ] Returns `200` with `{ result: { channel } }` on authorization success (Centrifugo-compatible response shape).
- [ ] Returns `403` if the user is not a board member and not an admin.
- [ ] Protected by the `auth` server middleware (unauthenticated requests receive `401`).

### AC-4: Server — Broadcast After Every CRUD Operation
- [ ] Every API route that creates, updates, or deletes a **Board** calls `broadcastToCentrifugo('board', boardId, 'board_create' | 'board_update' | 'board_delete', data)` after the DB write.
- [ ] Every API route that creates, updates, deletes, or reorders a **BoardList** calls `broadcastToCentrifugo('board', boardId, 'list_create' | 'list_update' | 'list_delete' | 'list_position', data)`.
- [ ] Every API route that creates, updates, deletes, or reorders a **Card** calls `broadcastToCentrifugo('board', boardId, 'card_create' | 'card_update' | 'card_delete' | 'card_position', data)`.
- [ ] User assignment to a board calls `broadcastToCentrifugo('board', boardId, 'board_add_users', data)`.
- [ ] User assignment/removal on a card calls both:
  - `broadcastToCentrifugo('board', boardId, 'card_add_users' | 'card_remove_users', data)`
  - `broadcastToCentrifugo('card', cardId, 'card_add_users' | 'card_remove_users', data)`
- [ ] **Comment** CRUD calls `broadcastToCentrifugo('card', cardId, 'comment_create' | 'comment_update' | 'comment_delete', data)`.
- [ ] **Attachment** CRUD calls `broadcastToCentrifugo('card', cardId, 'attachment_create' | 'attachment_update' | 'attachment_delete', data)`.
- [ ] **CardLog** creation calls `broadcastToCentrifugo('card', cardId, 'log_create', data)`.
- [ ] Broadcast calls are made **after** the DB write and audit trail entry succeed.

### AC-5: Client — `socketStore` Initialization & Connection Lifecycle
- [ ] `app/stores/socket.ts` is a Pinia setup store exporting `socketStore`.
- [ ] `initCentrifuge(token: string)` creates a `Centrifuge` instance pointed at `runtimeConfig.public.wsBaseUrl`, passing the JWT Bearer token for authentication.
- [ ] `socketConnect()` calls `centrifuge.connect()` and sets `state` to `'connecting'`; updates to `'connected'` on the `connect` event.
- [ ] `socketDisconnect()` calls `centrifuge.disconnect()`, clears all entries in `subscriptions`, and sets `state` to `'disconnected'`.
- [ ] `state` transitions: `'disconnected'` → `'connecting'` → `'connected'` → `'disconnected'`.
- [ ] Multiple calls to `socketConnect()` when already connected are idempotent (no duplicate connections).

### AC-6: Client — `subscribeToChannels` / `unsubscribeFromChannels`
- [ ] `subscribeToChannels(channels, eventType, callback)` creates a Centrifugo `Subscription` for each channel not already in `subscriptions`, registers the listener for the given `eventType`, and stores subscriptions in the `subscriptions` Map.
- [ ] `eventType` must be one of `'publication' | 'error' | 'join' | 'leave'`.
- [ ] Subscribing to a channel that already exists in `subscriptions` does not create a duplicate subscription.
- [ ] The channel authorization endpoint `POST /api/auth/centrifugo/channel` is used by the Centrifugo client when subscribing to private channels.
- [ ] `unsubscribeFromChannels(channels)` calls `subscription.unsubscribe()` and removes each entry from `subscriptions`.

### AC-7: Client — Board Page Event Handling
- [ ] On board page mount:
  1. `socketStore.initCentrifuge()` is called with the current user's JWT token.
  2. `socketStore.socketConnect()` is called.
  3. `socketStore.subscribeToChannels(['board#<id>'], 'publication', handleBoardEvent)` is called.
- [ ] `handleBoardEvent(event)` dispatches each of the 14 board-channel event types to the correct Pinia store action:
  - `board_update` → update board in store.
  - `board_delete` → remove board and navigate away.
  - `board_add_users` → update board member list.
  - `list_create` → append list to board store.
  - `list_update` → update list attributes in store.
  - `list_delete` → remove list from store.
  - `list_position` → reorder lists in store.
  - `card_create` → append card to the correct list in store.
  - `card_update` → update card attributes in store.
  - `card_delete` → remove card from store.
  - `card_position` → reorder cards within or across lists.
  - `card_add_users` → update card assignee list in store.
  - `card_remove_users` → update card assignee list in store.
- [ ] On board page unmount:
  1. `socketStore.unsubscribeFromChannels(['board#<id>'])` is called.
  2. `socketStore.socketDisconnect()` is called.

### AC-8: Client — Card Modal Event Handling
- [ ] On card modal open: `socketStore.subscribeToChannels(['card#<id>'], 'publication', handleCardEvent)`.
- [ ] `handleCardEvent(event)` triggers a card data refresh for all 10 card-channel event types:
  - `card_update`, `card_add_users`, `card_remove_users` → patch card fields in store.
  - `comment_create`, `comment_update`, `comment_delete` → trigger comments refresh.
  - `attachment_create`, `attachment_update`, `attachment_delete` → trigger attachments refresh.
  - `log_create` → trigger activity log refresh.
- [ ] On card modal close: `socketStore.unsubscribeFromChannels(['card#<id>'])`.

### AC-9: Client — Token Refresh
- [ ] `updateToken(newToken: string)` calls the Centrifugo client's token refresh method so the WebSocket session remains valid after a JWT rotation.
- [ ] `updateToken` is called whenever the auth store refreshes the user's JWT.

### AC-10: Environment & Configuration
- [ ] `CENTRIFUGO_API_URL`, `CENTRIFUGO_API_KEY`, and `CENTRIFUGO_SECRET` are server-only env vars (not exposed to the client).
- [ ] `WS_BASE_URL` is exposed as `runtimeConfig.public.wsBaseUrl` in `nuxt.config.ts`.
- [ ] Default dev value for `WS_BASE_URL`: `ws://localhost:8000/connection/websocket`.
- [ ] `centrifuge` npm package v5.5.2 is installed as a client dependency.

---

## Component Tree / File Structure

```
server/
├── services/
│   └── centrifugo.ts              # publishToCentrifugo() — HTTP client to Centrifugo API
├── utils/
│   └── broadcast.ts               # broadcastToCentrifugo() — channel string builder + delegate
└── api/
    └── auth/
        └── centrifugo/
            └── channel.post.ts    # POST /api/auth/centrifugo/channel — subscription auth

app/
├── stores/
│   └── socket.ts                  # socketStore — Pinia setup store (Centrifuge instance + subs)
└── composables/
    └── api/
        └── use-socket-events.ts   # handleBoardEvent() + handleCardEvent() dispatchers

shared/
└── schemas/
    └── centrifugo.schema.ts       # Zod: CentrifugoChannelAuthSchema
```

---

## Data Model

### Channel Format

| Channel Pattern | Scope | Subscriber |
|---|---|---|
| `board#{board_id}` | Board, list, and card-level events | Any board member or admin |
| `card#{card_id}` | Card detail, comments, attachments, logs | Any board member who can see the card, or admin |

### Board Channel Events (14 event types)

| Event | Trigger | Payload Data |
|---|---|---|
| `board_update` | Board metadata updated | `{ boardId, changes: Partial<Board> }` |
| `board_delete` | Board deleted | `{ boardId }` |
| `board_add_users` | User(s) added to board | `{ boardId, users: BoardMember[] }` |
| `list_create` | BoardList created | `{ list: BoardList }` |
| `list_update` | BoardList updated | `{ listId, changes: Partial<BoardList> }` |
| `list_delete` | BoardList deleted | `{ listId }` |
| `list_position` | Lists reordered | `{ lists: { id: number; position: number }[] }` |
| `card_create` | Card created | `{ card: Card }` |
| `card_update` | Card updated | `{ cardId, changes: Partial<Card> }` |
| `card_delete` | Card deleted | `{ cardId, listId }` |
| `card_position` | Cards reordered (within/across lists) | `{ cards: { id: number; listId: number; position: number }[] }` |
| `card_add_users` | Users assigned to card | `{ cardId, users: CardAssignee[] }` |
| `card_remove_users` | Users removed from card | `{ cardId, userIds: number[] }` |

> Note: `board_create` is included in the architecture but is not useful for subscribed board clients (they would not yet be subscribed). It is broadcast for admin dashboard use.

### Card Channel Events (10 event types)

| Event | Trigger | Client Action |
|---|---|---|
| `card_update` | Card fields changed | Patch card in store |
| `card_add_users` | Users assigned | Patch assignees in store |
| `card_remove_users` | Users removed | Patch assignees in store |
| `comment_create` | Comment added | Refresh comments list |
| `comment_update` | Comment edited | Refresh comments list |
| `comment_delete` | Comment removed | Refresh comments list |
| `attachment_create` | File uploaded | Refresh attachments list |
| `attachment_update` | File metadata updated | Refresh attachments list |
| `attachment_delete` | File removed | Refresh attachments list |
| `log_create` | Activity log entry created | Refresh activity log |

---

## API Endpoints Consumed

| Method | Endpoint | Request | Response |
|---|---|---|---|
| `POST` | `/api/auth/centrifugo/channel` | `{ channel: string }` | `{ result: { channel: string } }` \| `403` \| `400` |

> All other broadcasts are server-to-Centrifugo only (not consumed by the frontend directly).

### TypeScript Interfaces

```typescript
// shared/types/centrifugo.types.ts

interface BroadcastPayload {
  channel: string              // e.g., "board#1", "card#5"
  event: string                // e.g., "card_create", "comment_delete"
  data: Record<string, unknown>
}

interface SocketState {
  state: 'disconnected' | 'connecting' | 'connected'
  centrifuge: Centrifuge | null
  subscriptions: Map<string, Subscription>
}

interface CentrifugoChannelAuthRequest {
  channel: string              // e.g., "board#1"
}

type BoardChannelEvent =
  | 'board_create'
  | 'board_update'
  | 'board_delete'
  | 'board_add_users'
  | 'list_create'
  | 'list_update'
  | 'list_delete'
  | 'list_position'
  | 'card_create'
  | 'card_update'
  | 'card_delete'
  | 'card_position'
  | 'card_add_users'
  | 'card_remove_users'

type CardChannelEvent =
  | 'card_update'
  | 'card_add_users'
  | 'card_remove_users'
  | 'comment_create'
  | 'comment_update'
  | 'comment_delete'
  | 'attachment_create'
  | 'attachment_update'
  | 'attachment_delete'
  | 'log_create'

type ChannelType = 'board' | 'card'
```

---

## State Management

### `socketStore` (`app/stores/socket.ts`)

**State**
```typescript
const state = ref<'disconnected' | 'connecting' | 'connected'>('disconnected')
const centrifuge = ref<Centrifuge | null>(null)
const subscriptions = ref<Map<string, Subscription>>(new Map())
```

**Actions**

| Action | Description |
|---|---|
| `initCentrifuge(token)` | Creates `new Centrifuge(wsBaseUrl, { token })` instance. Does not connect yet. |
| `socketConnect()` | Calls `centrifuge.connect()`, transitions `state` to `'connecting'`, then `'connected'` on success. |
| `socketDisconnect()` | Calls `centrifuge.disconnect()`, clears `subscriptions` Map, sets `state` to `'disconnected'`. |
| `updateToken(newToken)` | Calls `centrifuge.setToken(newToken)` to refresh the session token. |
| `subscribeToChannels(channels, eventType, callback)` | For each channel, creates/reuses a `Subscription`, registers the event handler, stores in `subscriptions`. |
| `unsubscribeFromChannels(channels)` | For each channel, calls `subscription.unsubscribe()` and removes from `subscriptions` Map. |
| `publishEvent(channel, data)` | Publishes a message on the given channel (used for presence/peer signaling if needed). |

**Connection Lifecycle**

```
Board page mount
  └─ initCentrifuge(token)
  └─ socketConnect()
  └─ subscribeToChannels(['board#<id>'], 'publication', handleBoardEvent)

Card modal open
  └─ subscribeToChannels(['card#<id>'], 'publication', handleCardEvent)

Card modal close
  └─ unsubscribeFromChannels(['card#<id>'])

Board page unmount
  └─ unsubscribeFromChannels(['board#<id>'])
  └─ socketDisconnect()
```

**Interactions with other stores**
- `socketStore` → `boardStore`: board/list/card mutations via `handleBoardEvent`
- `socketStore` → `cardStore`: card detail/comment/attachment/log refresh via `handleCardEvent`
- `authStore` → `socketStore`: calls `updateToken()` after JWT rotation

---

## Edge Cases

| Scenario | Expected Behaviour |
|---|---|
| Centrifugo server is unreachable during broadcast | Error is caught and logged server-side; the originating API handler returns a success response to the client anyway. Real-time delivery is best-effort. |
| User loses network connection mid-session | `centrifuge` library handles reconnection with exponential backoff automatically. `state` reflects `'connecting'` during reconnect. |
| User opens the same board in two tabs | Each tab maintains its own `socketStore` instance and independent subscriptions. Both receive broadcasts. |
| Channel auth endpoint called with invalid channel format | Returns `400 Bad Request`. |
| Non-member attempts to subscribe to a private board channel | Channel authorization returns `403`; Centrifugo denies the subscription. |
| Card modal is closed before the subscription is confirmed | `unsubscribeFromChannels` is idempotent — if the subscription is not in the Map, it is a no-op. |
| JWT expires while WebSocket is connected | `updateToken()` must be called by the auth store upon token refresh to keep the Centrifugo session valid. If not refreshed, Centrifugo will disconnect the client on token expiry. |
| Rapid sequence of events for the same card | Each event is processed in order of arrival. The last write wins for field patches; refresh-based events (comments/attachments) re-fetch the full list. |
| `broadcastToCentrifugo` called with unknown event string | No runtime error; the event is published as-is. Frontend `handleBoardEvent`/`handleCardEvent` should have a default no-op case for unknown events. |
| Board deleted while user is on the board page | `board_delete` event triggers navigation away from the board page and removal of the board from the store. |

---

## Non-Goals

- **Presence indicators** (showing who is currently viewing a board) — not in scope for this spec.
- **Typing indicators** in comments — not in scope.
- **Optimistic UI updates** — all UI updates are driven by server-confirmed WebSocket events, not local speculation.
- **Message history / replay** — Centrifugo channel history is not configured; clients that were offline will not receive missed events and must reload page data on reconnect.
- **Client-to-client direct messaging** via Centrifugo — all messages flow server → Centrifugo → clients only.
- **Read receipts** or acknowledgment of received events.
- **Rate limiting of broadcast calls** — handled at the Centrifugo server configuration level, not in application code.

---

## Dependencies

| Dependency | Type | Version | Purpose |
|---|---|---|---|
| `centrifuge` | npm (client) | 5.5.2 | WebSocket client library for the browser |
| `centrifugo/centrifugo` | Docker image | v5 | WebSocket message broker server |
| Drizzle ORM | internal | — | DB writes that precede each broadcast |
| `jose` / `server/utils/jwt.ts` | internal | — | JWT issued to user, passed to Centrifuge client at init |
| `server/middleware/auth.ts` | internal | — | Protects the channel authorization endpoint |
| `server/database/schema/boards.ts` | internal | — | Board membership lookup in channel auth |
| Zod | shared | — | `CentrifugoChannelAuthSchema` request validation |
| Redis | infrastructure | — | Not directly used by Centrifugo integration (used by BullMQ separately) |

### Environment Variables

| Variable | Scope | Description |
|---|---|---|
| `CENTRIFUGO_API_URL` | Server only | Base URL of Centrifugo HTTP API (e.g., `http://localhost:8000`) |
| `CENTRIFUGO_API_KEY` | Server only | API key for Centrifugo publish endpoint |
| `CENTRIFUGO_SECRET` | Server only | Token signing secret (shared with Centrifugo config) |
| `WS_BASE_URL` | Public (client) | WebSocket connection URL exposed as `runtimeConfig.public.wsBaseUrl` |

### Docker Compose Entry (dev)
```yaml
centrifugo:
  image: centrifugo/centrifugo:v5
  ports:
    - "8000:8000"
  volumes:
    - ./centrifugo.json:/centrifugo/config.json
  command: centrifugo --config=/centrifugo/config.json
```
