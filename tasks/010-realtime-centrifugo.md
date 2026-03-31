# Tasks: Real-Time System — Centrifugo WebSocket Integration
**Spec:** `specs/realtime-centrifugo.md`
**Status:** pending

---

## Overview

Implements the full real-time collaboration layer using Centrifugo as a WebSocket message broker. Work is split across three waves:

1. **Wave 1 (API Agent):** Server-side infrastructure — shared types/schemas, `publishToCentrifugo` service, `broadcastToCentrifugo` utility, channel authorization endpoint, CRUD broadcast wiring, and Nuxt config.
2. **Wave 2 (Frontend Agent):** Client-side — `socketStore` Pinia setup store (connection lifecycle, subscriptions, token refresh), `use-socket-events` composable (board + card event dispatchers), and board page / card modal integration points.
3. **Wave 3 (Testing Agent):** Unit tests for the service, utility, channel auth endpoint, `socketStore`, and event composable.

**Channel types:** `board#{id}` (14 event types) · `card#{id}` (10 event types)

---

## Wave 1 — Data Layer (API Agent)

### Task 010-A: Shared Types & Centrifugo Zod Schema
- **Agent:** API
- **Spec:** `specs/realtime-centrifugo.md`
- **Section:** TypeScript Interfaces · Dependencies
- **Acceptance Criteria:**
  - [ ] AC-10a: `shared/types/centrifugo.types.ts` exports `BroadcastPayload`, `SocketState`, `CentrifugoChannelAuthRequest`, `BoardChannelEvent` union type (14 members: `board_create | board_update | board_delete | board_add_users | list_create | list_update | list_delete | list_position | card_create | card_update | card_delete | card_position | card_add_users | card_remove_users`), `CardChannelEvent` union type (10 members: `card_update | card_add_users | card_remove_users | comment_create | comment_update | comment_delete | attachment_create | attachment_update | attachment_delete | log_create`), and `ChannelType` union type (`'board' | 'card'`).
  - [ ] AC-10b: `shared/schemas/centrifugo.schema.ts` exports `CentrifugoChannelAuthSchema` as `z.object({ channel: z.string().min(1) })`.
- **Dependencies:** `tasks/001-infrastructure-setup.md`, `tasks/003-validation-schemas.md`
- **Output Files:**
  - `shared/types/centrifugo.types.ts`
  - `shared/schemas/centrifugo.schema.ts`

---

### Task 010-B: CentrifugoService — `publishToCentrifugo`
- **Agent:** API
- **Spec:** `specs/realtime-centrifugo.md`
- **Section:** AC-1 — Server: Centrifugo Service
- **Acceptance Criteria:**
  - [ ] AC-1a: `server/services/centrifugo.ts` exports an async `publishToCentrifugo(payload: BroadcastPayload)` function typed with `BroadcastPayload` from `shared/types/centrifugo.types.ts`.
  - [ ] AC-1b: The function issues a `POST` request to `${CENTRIFUGO_API_URL}/api/publish` using `$fetch`.
  - [ ] AC-1c: The `Authorization` header is set to `apikey ${process.env.CENTRIFUGO_API_KEY}`.
  - [ ] AC-1d: The request body is `{ channel: payload.channel, data: { event: payload.event, ...payload.data } }`.
  - [ ] AC-1e: HTTP errors from Centrifugo are caught, logged server-side (e.g., `console.error` or logger), and **do not** rethrow — broadcast failures are non-fatal to the calling route handler.
- **Dependencies:** Task 010-A, `tasks/001-infrastructure-setup.md`
- **Output Files:**
  - `server/services/centrifugo.ts`

---

### Task 010-C: Broadcast Utility — `broadcastToCentrifugo`
- **Agent:** API
- **Spec:** `specs/realtime-centrifugo.md`
- **Section:** AC-2 — Server: Broadcast Utility
- **Acceptance Criteria:**
  - [ ] AC-2a: `server/utils/broadcast.ts` exports an async `broadcastToCentrifugo(channelType: ChannelType, channelId: number | string, event: string, data: Record<string, unknown>)` function.
  - [ ] AC-2b: Channel string is constructed as `` `${channelType}#${channelId}` `` (e.g., `board#1`, `card#5`).
  - [ ] AC-2c: `channelType` parameter is typed as `ChannelType` (`'board' | 'card'`) — TypeScript enforces only these two values.
  - [ ] AC-2d: Delegates to `publishToCentrifugo()` with the constructed `channel`, provided `event`, and `data`.
- **Dependencies:** Task 010-A, Task 010-B
- **Output Files:**
  - `server/utils/broadcast.ts`

---

### Task 010-D: Channel Authorization Endpoint
- **Agent:** API
- **Spec:** `specs/realtime-centrifugo.md`
- **Section:** AC-3 — Server: Channel Authorization Endpoint
- **Acceptance Criteria:**
  - [ ] AC-3a: File `server/api/auth/centrifugo/channel.post.ts` handles `POST /api/auth/centrifugo/channel`.
  - [ ] AC-3b: Request body is validated with `CentrifugoChannelAuthSchema` (`z.object({ channel: z.string().min(1) })`); malformed body returns `400`.
  - [ ] AC-3c: Channel format is validated against the pattern `board#{id}` or `card#{cardId}` (regex); any other format returns `400 Bad Request`.
  - [ ] AC-3d: For `board#{id}` channels: queries the DB to confirm the authenticated user is an active board member **or** has admin role; returns `403` if neither.
  - [ ] AC-3e: For `card#{cardId}` channels: resolves the card's parent board and checks the same membership rule (active board member **or** admin); returns `403` if neither.
  - [ ] AC-3f: On authorization success, returns `200` with `{ result: { channel } }` (Centrifugo-compatible response shape).
  - [ ] AC-3g: The route is protected by the `auth` server middleware; unauthenticated requests receive `401`.
- **Dependencies:** Task 010-A, `tasks/001-infrastructure-setup.md`, `tasks/004-authentication.md`, `tasks/002-database-schema.md`
- **Output Files:**
  - `server/api/auth/centrifugo/channel.post.ts`

---

### Task 010-E: Broadcast Integration — Wire Broadcasts into All CRUD Routes
- **Agent:** API
- **Spec:** `specs/realtime-centrifugo.md`
- **Section:** AC-4 — Server: Broadcast After Every CRUD Operation
- **Acceptance Criteria:**
  - [ ] AC-4a: Board route handlers call `broadcastToCentrifugo('board', boardId, event, data)` **after** DB write for: `board_create`, `board_update`, `board_delete`.
  - [ ] AC-4b: BoardList route handlers call `broadcastToCentrifugo('board', boardId, event, data)` after DB write for: `list_create`, `list_update`, `list_delete`, `list_position`.
  - [ ] AC-4c: Card route handlers call `broadcastToCentrifugo('board', boardId, event, data)` after DB write for: `card_create`, `card_update`, `card_delete`, `card_position`.
  - [ ] AC-4d: Board user-assignment route calls `broadcastToCentrifugo('board', boardId, 'board_add_users', data)` after DB write.
  - [ ] AC-4e: Card user-assignment route calls **both**:
    - `broadcastToCentrifugo('board', boardId, 'card_add_users' | 'card_remove_users', data)`
    - `broadcastToCentrifugo('card', cardId, 'card_add_users' | 'card_remove_users', data)`
  - [ ] AC-4f: Comment route handlers call `broadcastToCentrifugo('card', cardId, event, data)` after DB write for: `comment_create`, `comment_update`, `comment_delete`.
  - [ ] AC-4g: Attachment route handlers call `broadcastToCentrifugo('card', cardId, event, data)` after DB write for: `attachment_create`, `attachment_update`, `attachment_delete`.
  - [ ] AC-4h: CardLog creation calls `broadcastToCentrifugo('card', cardId, 'log_create', data)` after DB write.
  - [ ] AC-4i: All broadcast calls are placed **after** the DB write and after any audit trail entry — never before.
- **Dependencies:** Task 010-C, `tasks/002-database-schema.md`
- **Output Files:**
  - `server/api/client/board/*.ts` *(modify existing board CRUD handlers)*
  - `server/api/client/board/[id]/lists/*.ts` *(modify existing list CRUD handlers)*
  - `server/api/client/board/[id]/lists/[listId]/cards/*.ts` *(modify existing card CRUD handlers)*
  - `server/api/client/card/[id]/comments/*.ts` *(modify existing comment CRUD handlers)*
  - `server/api/client/card/[id]/attachments/*.ts` *(modify existing attachment CRUD handlers)*

---

### Task 010-F: Environment & Nuxt Config
- **Agent:** API
- **Spec:** `specs/realtime-centrifugo.md`
- **Section:** AC-10 — Environment & Configuration
- **Acceptance Criteria:**
  - [ ] AC-10c: `CENTRIFUGO_API_URL`, `CENTRIFUGO_API_KEY`, and `CENTRIFUGO_SECRET` are consumed as server-only `runtimeConfig` keys in `nuxt.config.ts` (not inside `public`).
  - [ ] AC-10d: `WS_BASE_URL` is mapped to `runtimeConfig.public.wsBaseUrl` in `nuxt.config.ts`.
  - [ ] AC-10e: Default dev value for `runtimeConfig.public.wsBaseUrl` is `ws://localhost:8000/connection/websocket`.
  - [ ] AC-10f: `centrifuge` npm package at version `5.5.2` is listed in `package.json` under `dependencies`.
  - [ ] AC-10g: Centrifugo Docker Compose service is added to `compose.dev.yaml` with image `centrifugo/centrifugo:v5`, port `8000:8000`, and config volume mount (`./centrifugo.json:/centrifugo/config.json`).
- **Dependencies:** `tasks/001-infrastructure-setup.md`
- **Output Files:**
  - `nuxt.config.ts` *(modify — add runtimeConfig keys)*
  - `compose.dev.yaml` *(modify — add centrifugo service)*
  - `package.json` *(modify — add centrifuge@5.5.2)*

---

## Wave 2 — Frontend (Frontend Agent)

### Task 010-G: `socketStore` — Connection Lifecycle & Subscription Management
- **Agent:** Frontend
- **Spec:** `specs/realtime-centrifugo.md`
- **Section:** AC-5, AC-6, AC-9 — Client: socketStore
- **Acceptance Criteria:**
  - [ ] AC-5a: `app/stores/socket.ts` is a Pinia setup store exported as `socketStore`.
  - [ ] AC-5b: Reactive state: `state: Ref<'disconnected' | 'connecting' | 'connected'>` (default `'disconnected'`), `centrifuge: Ref<Centrifuge | null>` (default `null`), `subscriptions: Ref<Map<string, Subscription>>` (default empty Map).
  - [ ] AC-5c: `initCentrifuge(token: string)` creates a `new Centrifuge(runtimeConfig.public.wsBaseUrl, { token })` instance and assigns it to the `centrifuge` ref. Does **not** connect.
  - [ ] AC-5d: `socketConnect()` calls `centrifuge.connect()`, sets `state` to `'connecting'`, and sets `state` to `'connected'` inside the Centrifuge `connect` event handler.
  - [ ] AC-5e: `socketDisconnect()` calls `centrifuge.disconnect()`, clears all entries from the `subscriptions` Map, and sets `state` to `'disconnected'`.
  - [ ] AC-5f: `state` transitions follow exactly: `'disconnected'` → `'connecting'` → `'connected'` → `'disconnected'`.
  - [ ] AC-5g: Multiple calls to `socketConnect()` when `state` is already `'connected'` or `'connecting'` are idempotent (early return, no duplicate connections).
  - [ ] AC-6a: `subscribeToChannels(channels: string[], eventType: 'publication' | 'error' | 'join' | 'leave', callback: Function)` iterates `channels`; for each channel NOT already in `subscriptions`, creates a `Subscription` using `centrifuge.newSubscription(channel)`, registers the event listener for `eventType`, calls `subscription.subscribe()`, and stores the subscription in the `subscriptions` Map keyed by channel name.
  - [ ] AC-6b: Subscribing to a channel that already exists in `subscriptions` does not create a duplicate — idempotent.
  - [ ] AC-6c: The Centrifugo client is configured to use `POST /api/auth/centrifugo/channel` as the subscription token endpoint for private channels.
  - [ ] AC-6d: `unsubscribeFromChannels(channels: string[])` calls `subscription.unsubscribe()` and deletes each entry from the `subscriptions` Map. If a channel is not in the Map, it is a no-op.
  - [ ] AC-9a: `updateToken(newToken: string)` calls `centrifuge.setToken(newToken)` to refresh the session without disconnecting.
  - [ ] AC-9b: `publishEvent(channel: string, data: Record<string, unknown>)` calls `centrifuge.publish(channel, data)` (for presence/peer signaling).
- **Dependencies:** Task 010-A, Task 010-F
- **Output Files:**
  - `app/stores/socket.ts`

---

### Task 010-H: `use-socket-events` Composable — Board & Card Event Dispatchers
- **Agent:** Frontend
- **Spec:** `specs/realtime-centrifugo.md`
- **Section:** AC-7 (handleBoardEvent) · AC-8 (handleCardEvent)
- **Acceptance Criteria:**
  - [ ] AC-7a: `app/composables/api/use-socket-events.ts` exports a `useSocketEvents()` composable that returns both `handleBoardEvent` and `handleCardEvent` functions.
  - [ ] AC-7b: `handleBoardEvent(event)` dispatches all 14 board-channel event types to the correct Pinia store actions:
    - `board_update` → update board metadata in `boardStore`.
    - `board_delete` → remove the board from `boardStore` and call `navigateTo('/')` (or equivalent away-navigation).
    - `board_add_users` → update the board member list in `boardStore`.
    - `list_create` → append the new list to `boardStore`.
    - `list_update` → update list attributes in `boardStore`.
    - `list_delete` → remove the list from `boardStore`.
    - `list_position` → reorder the lists array in `boardStore`.
    - `card_create` → append the card to the correct list in `boardStore`.
    - `card_update` → update card attributes in `boardStore`.
    - `card_delete` → remove the card from its list in `boardStore`.
    - `card_position` → reorder cards within or across lists in `boardStore`.
    - `card_add_users` → update the card's assignee list in `boardStore`.
    - `card_remove_users` → update the card's assignee list in `boardStore`.
    - Unknown event types → no-op (default case, no error thrown).
  - [ ] AC-8a: `handleCardEvent(event)` dispatches all 10 card-channel event types:
    - `card_update`, `card_add_users`, `card_remove_users` → patch card fields in `cardStore`.
    - `comment_create`, `comment_update`, `comment_delete` → trigger comments refresh in `cardStore`.
    - `attachment_create`, `attachment_update`, `attachment_delete` → trigger attachments refresh in `cardStore`.
    - `log_create` → trigger activity log refresh in `cardStore`.
    - Unknown event types → no-op (default case, no error thrown).
- **Dependencies:** Task 010-G
- **Output Files:**
  - `app/composables/api/use-socket-events.ts`

---

### Task 010-I: Board Page — WebSocket Lifecycle Integration
- **Agent:** Frontend
- **Spec:** `specs/realtime-centrifugo.md`
- **Section:** AC-7 — Client: Board Page Event Handling (mount/unmount lifecycle)
- **Acceptance Criteria:**
  - [ ] AC-7c: On board page `onMounted`:
    1. `socketStore.initCentrifuge(authStore.token)` is called with the current user's JWT.
    2. `socketStore.socketConnect()` is called.
    3. `socketStore.subscribeToChannels(['board#<id>'], 'publication', handleBoardEvent)` is called with the board ID from the route.
  - [ ] AC-7d: On board page `onUnmounted`:
    1. `socketStore.unsubscribeFromChannels(['board#<id>'])` is called.
    2. `socketStore.socketDisconnect()` is called.
- **Dependencies:** Task 010-G, Task 010-H
- **Output Files:**
  - `app/pages/board/[id].vue` *(modify — add onMounted/onUnmounted WebSocket lifecycle)*

---

### Task 010-J: Card Modal — WebSocket Subscription Lifecycle
- **Agent:** Frontend
- **Spec:** `specs/realtime-centrifugo.md`
- **Section:** AC-8 — Client: Card Modal Event Handling (open/close lifecycle)
- **Acceptance Criteria:**
  - [ ] AC-8b: When the card modal opens, `socketStore.subscribeToChannels(['card#<id>'], 'publication', handleCardEvent)` is called with the card's ID.
  - [ ] AC-8c: When the card modal closes, `socketStore.unsubscribeFromChannels(['card#<id>'])` is called.
  - [ ] AC-8d: If the modal is closed before the subscription is confirmed, `unsubscribeFromChannels` is a no-op (channel not in Map).
- **Dependencies:** Task 010-G, Task 010-H
- **Output Files:**
  - `app/components/card-modal.vue` *(modify — add subscription on open, unsubscribe on close)*

---

## Wave 3 — Tests (Testing Agent)

### Task 010-K: Server-Side Unit Tests
- **Agent:** Testing
- **Spec:** `specs/realtime-centrifugo.md`
- **Section:** AC-1 · AC-2 · AC-3
- **Acceptance Criteria:**
  - [ ] `publishToCentrifugo` — happy path: correct `POST` URL, `Authorization: apikey <key>` header, and body shape `{ channel, data: { event, ...data } }` are passed to `$fetch`.
  - [ ] `publishToCentrifugo` — error case: `$fetch` throws an HTTP error → error is logged but function resolves without rethrowing.
  - [ ] `broadcastToCentrifugo` — constructs channel string as `${channelType}#${channelId}` and delegates to `publishToCentrifugo` with correct arguments.
  - [ ] `broadcastToCentrifugo` — TypeScript compile error if `channelType` is not `'board'` or `'card'` (type-level test via `@ts-expect-error`).
  - [ ] Channel auth endpoint — happy path (board membership): returns `200` with `{ result: { channel } }`.
  - [ ] Channel auth endpoint — happy path (admin): returns `200` regardless of board membership.
  - [ ] Channel auth endpoint — invalid format: returns `400` for a channel string not matching `board#N` or `card#N`.
  - [ ] Channel auth endpoint — non-member non-admin: returns `403`.
  - [ ] Channel auth endpoint — unauthenticated: returns `401` (auth middleware).
  - [ ] Channel auth endpoint — card channel resolves parent board for membership check.
- **Dependencies:** Task 010-A, Task 010-B, Task 010-C, Task 010-D
- **Output Files:**
  - `tests/unit/server/services/centrifugo.test.ts`
  - `tests/unit/server/utils/broadcast.test.ts`
  - `tests/unit/server/api/auth/centrifugo/channel.test.ts`

---

### Task 010-L: Client-Side Unit Tests
- **Agent:** Testing
- **Spec:** `specs/realtime-centrifugo.md`
- **Section:** AC-5 · AC-6 · AC-7 · AC-8 · AC-9
- **Acceptance Criteria:**
  - [ ] `socketStore.initCentrifuge(token)` — creates a `Centrifuge` instance with correct `wsBaseUrl` and token; does not call `connect()`.
  - [ ] `socketStore.socketConnect()` — calls `centrifuge.connect()` and sets `state` to `'connecting'`; state transitions to `'connected'` after the mock `connect` event fires.
  - [ ] `socketStore.socketConnect()` — idempotent: second call when already `'connected'` does not call `connect()` again.
  - [ ] `socketStore.socketDisconnect()` — calls `centrifuge.disconnect()`, clears `subscriptions` Map, sets `state` to `'disconnected'`.
  - [ ] `socketStore.subscribeToChannels` — creates subscription for a new channel and stores it in the Map.
  - [ ] `socketStore.subscribeToChannels` — does not create a duplicate subscription for an already-subscribed channel.
  - [ ] `socketStore.unsubscribeFromChannels` — calls `subscription.unsubscribe()` and removes from Map.
  - [ ] `socketStore.unsubscribeFromChannels` — is a no-op when the channel is not in the Map.
  - [ ] `socketStore.updateToken(newToken)` — calls `centrifuge.setToken(newToken)`.
  - [ ] `handleBoardEvent` — dispatches all 14 `BoardChannelEvent` types to the correct store action (one test per event type, mocked store).
  - [ ] `handleBoardEvent` — unknown event type is a no-op (no error thrown).
  - [ ] `handleCardEvent` — dispatches all 10 `CardChannelEvent` types to the correct store action (one test per event type, mocked store).
  - [ ] `handleCardEvent` — unknown event type is a no-op (no error thrown).
- **Dependencies:** Task 010-G, Task 010-H, Task 010-I, Task 010-J
- **Output Files:**
  - `tests/unit/app/stores/socket.test.ts`
  - `tests/unit/app/composables/api/use-socket-events.test.ts`
