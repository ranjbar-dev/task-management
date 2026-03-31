# Spec: Client Card Management

## Status: Approved

---

## Purpose

Provide full card lifecycle management for authenticated, active users within their accessible boards. This covers creating, reading, updating, deleting, duplicating, and reordering cards; syncing members and tags; updating per-user card status; and ensuring all mutations broadcast real-time events via Centrifugo and trigger the appropriate notifications.

---

## User Stories

1. **As a board member**, I can create a card in a board list with a name, optional description, optional assignees, and optional tags so that work items are tracked on the kanban board.
2. **As a board member**, I can quick-create a card by entering only a name so that I can add items rapidly without leaving the kanban flow.
3. **As a board member**, I can view a card's full detail — members, tags, comments, attachments, and activity — so that I have all context in one place.
4. **As a board member**, I can edit a card's name, description, list assignment, or position so that I can keep cards organised and up to date.
5. **As a board member**, I can duplicate a card so that I can reuse a card structure without re-entering fields.
6. **As a board member**, I can delete a card (soft delete) so that it is removed from the board without permanently losing data.
7. **As a board member**, I can drag cards between lists or reorder them within a list so that priorities are reflected visually.
8. **As a board member**, I can sync card members (add/remove multiple in one call) or remove a single member so that assignments stay accurate.
9. **As a board member**, I can sync tags on a card so that cards are categorised consistently.
10. **As a board member**, I can update my own (or another member's) status on a card so that progress is visible to the team.
11. **As a board member**, I receive in-app notifications when I am assigned, removed, or when a card I am on changes, so that I stay informed.

---

## Acceptance Criteria

### AC-1 — Create Card (`POST /api/client/card/create/[boardListId]`)

- [ ] Requires `auth` + `active-user` middleware.
- [ ] Validates body against `createCardSchema`: `name` (1–255 chars, required), `description` (optional string), `position` (required int), `user_ids` (optional int array), `tag_ids` (optional int array).
- [ ] Returns `400` with Zod validation errors on invalid body.
- [ ] Inserts card into `cards` table with correct `board_list_id` and `position`.
- [ ] If `user_ids` provided, inserts rows into `card_user` pivot; triggers `CreateNewCardNotification` to each assigned user.
- [ ] If `tag_ids` provided, inserts rows into `card_tag` pivot.
- [ ] Broadcasts `card_create` to `board#{boardId}`.
- [ ] Returns `201` with card transformed by `transformCard`.

### AC-2 — Quick Create Card (`POST /api/client/card/quick-create/[boardListId]`)

- [ ] Requires `auth` + `active-user` middleware.
- [ ] Validates body against `quickCreateCardSchema`: `name` (1–255 chars, required).
- [ ] Inserts card with `name` and auto-assigned `position` (appended to end of list).
- [ ] Automatically assigns the current authenticated user to the card (`card_user` row).
- [ ] Triggers `CreateNewCardNotification` to the creating user.
- [ ] Broadcasts `card_create` to `board#{boardId}`.
- [ ] Returns `201` with card transformed by `transformCard`.

### AC-3 — Duplicate Card (`POST /api/client/card/duplicate/[id]`)

- [ ] Requires `auth` + `active-user` middleware.
- [ ] Returns `404` if card does not exist or is soft-deleted (`deleted_at IS NOT NULL`).
- [ ] Clones card record to the **same** `board_list_id` with `name` appended with ` (copy)`.
- [ ] Clones position as last in the list (appended after the original card or at end).
- [ ] Does **NOT** clone members, comments, attachments, or tags.
- [ ] Broadcasts `card_create` to `board#{boardId}`.
- [ ] Returns `201` with duplicated card transformed by `transformCard`.

### AC-4 — Show Card (`GET /api/client/card/show/[id]`)

- [ ] Requires `auth` + `active-user` middleware.
- [ ] Returns `404` if card does not exist or is soft-deleted.
- [ ] Returns `200` with card transformed by `transformCard` including: `id`, `boardlist_id`, `name`, `description`, `position`, `tags[]`, `users[]` (with pivot `status` from `card_user`), `comments[]`, `card_attachments[]` (not linked to a comment), `comment_attachments[]` (linked to a comment).
- [ ] Tags include expanded `color` (enum name + value).

### AC-5 — Update Card (`PUT /api/client/card/[id]`)

- [ ] Requires `auth` + `active-user` middleware.
- [ ] Validates body against `updateCardSchema`: `name` (1–255, optional), `description` (optional), `board_list_id` (optional int), `position` (optional int).
- [ ] Returns `404` if card not found or soft-deleted.
- [ ] Updates only supplied fields.
- [ ] If `name` changed → triggers `CardInfoChangeNotification` to all card members; broadcasts `card_update` to `board#{boardId}` and `card#{cardId}`.
- [ ] If `description` changed → triggers `CardDescriptionChangeNotification` to all card members; broadcasts `card_update` to `board#{boardId}` and `card#{cardId}`.
- [ ] If `board_list_id` or `position` changed → broadcasts `card_update` to `board#{boardId}` and `card#{cardId}`.
- [ ] Returns `200` with updated card via `transformCard`.

### AC-6 — Delete Card (`DELETE /api/client/card/[id]`)

- [ ] Requires `auth` + `active-user` middleware.
- [ ] Returns `404` if card not found or already soft-deleted.
- [ ] Sets `deleted_at = now()` via `softDeleteCard()` utility — does **not** remove the DB row.
- [ ] Triggers `DeleteCardNotification` to all card members.
- [ ] Broadcasts `card_delete` to `board#{boardId}`.
- [ ] Returns `200` with success message.

### AC-7 — Update Card Position (`POST /api/client/card/update-position/[id]`)

- [ ] Requires `auth` + `active-user` middleware.
- [ ] Validates body against `updateCardPositionSchema`: `board_list_id` (int, required), `items` (array of `{ id: int, position: int }`, required).
- [ ] Batch-updates `position` for all provided card IDs in `items`; sets `board_list_id` on the moved card.
- [ ] Broadcasts `card_position` to `board#{boardId}`.
- [ ] Returns `200` with success message.

### AC-8 — Sync Card Members (`POST /api/client/card/members/[id]`)

- [ ] Requires `auth` + `active-user` middleware.
- [ ] Validates body against `syncCardMembersSchema`: `user_ids` (int array, required).
- [ ] Returns `404` if card not found or soft-deleted.
- [ ] Computes diff: newly added user IDs vs removed user IDs against existing `card_user` rows.
- [ ] Inserts `card_user` rows for new users → triggers `AssignToCardNotification` to each newly added user.
- [ ] Deletes `card_user` rows for removed users → triggers `RemoveFromCardNotification` to each removed user.
- [ ] Broadcasts `card_add_users` (for additions) and `card_remove_users` (for removals) to `board#{boardId}` and `card#{cardId}`.
- [ ] Returns `200` with updated card via `transformCard`.

### AC-9 — Remove Single Card Member (`POST /api/client/card/[cardId]/delete-member/[userId]`)

- [ ] Requires `auth` + `active-user` middleware.
- [ ] Returns `404` if card not found or member not on card.
- [ ] Deletes the `card_user` row for the specified `userId`.
- [ ] Triggers `RemoveFromCardNotification` to the removed user.
- [ ] Broadcasts `card_remove_users` to `board#{boardId}` and `card#{cardId}`.
- [ ] Returns `200` with success message.

### AC-10 — Sync Card Tags (`POST /api/client/card/tags/[id]`)

- [ ] Requires `auth` + `active-user` middleware.
- [ ] Validates body against `syncCardTagsSchema`: `tag_ids` (int array, required).
- [ ] Returns `404` if card not found or soft-deleted.
- [ ] Replaces existing `card_tag` rows with the provided `tag_ids` (full sync — delete all, re-insert).
- [ ] Triggers `CardNewTagNotification` to all card members.
- [ ] Returns `200` with updated card via `transformCard`.

### AC-11 — Update Card Status (`POST /api/client/card/update-status/[id]`)

- [ ] Requires `auth` + `active-user` middleware.
- [ ] Validates body against `updateCardStatusSchema`: `user_id` (int, required), `status` (coerced int 0–4, required).
- [ ] Returns `404` if card not found or user is not a member of the card.
- [ ] Updates `card_user.status` for the given `user_id` on the card.
- [ ] Triggers `CardStatusChangeNotification` to all card members.
- [ ] Returns `200` with updated card via `transformCard`.

### AC-12 — Soft Delete Behaviour

- [ ] `softDeleteCard(cardId)` sets `cards.deleted_at = now()`.
- [ ] `restoreCard(cardId)` sets `cards.deleted_at = null`.
- [ ] All card queries (show, list within board) filter `WHERE deleted_at IS NULL` using Drizzle `isNull(cards.deletedAt)`.
- [ ] Soft-deleted cards never appear in kanban board listing or card show.

### AC-13 — Real-Time Broadcasts (Centrifugo)

| Event | Channel(s) | Trigger |
|-------|-----------|---------|
| `card_create`        | `board#{boardId}` | create, quick-create, duplicate |
| `card_update`        | `board#{boardId}`, `card#{cardId}` | update (name, description, list, position) |
| `card_delete`        | `board#{boardId}` | soft delete |
| `card_position`      | `board#{boardId}` | update-position |
| `card_add_users`     | `board#{boardId}`, `card#{cardId}` | member sync (additions) |
| `card_remove_users`  | `board#{boardId}`, `card#{cardId}` | member sync (removals), delete-member |

- [ ] All broadcasts are fire-and-forget via `broadcastToCentrifugo` utility and do not block the API response.

### AC-14 — Notification Triggers

| Notification | Trigger |
|-------------|---------|
| `CreateNewCardNotification` | Card created (full create or quick-create) — sent to all assigned users |
| `DeleteCardNotification` | Card soft-deleted — sent to all card members |
| `AssignToCardNotification` | User added via member sync — sent to each newly added user |
| `RemoveFromCardNotification` | User removed via member sync or delete-member — sent to removed user |
| `CardNewTagNotification` | Tags synced — sent to all card members |
| `CardStatusChangeNotification` | Status updated — sent to all card members |
| `CardInfoChangeNotification` | Card name changed — sent to all card members |
| `CardDescriptionChangeNotification` | Card description changed — sent to all card members |

- [ ] All notifications are dispatched via `NotificationService` (BullMQ queue) — not inline.

---

## Component Tree / File Structure

```
server/api/client/card/
├── create/
│   └── [boardListId].post.ts       # AC-1
├── quick-create/
│   └── [boardListId].post.ts       # AC-2
├── duplicate/
│   └── [id].post.ts                # AC-3
├── show/
│   └── [id].get.ts                 # AC-4
├── [id].put.ts                     # AC-5
├── [id].delete.ts                  # AC-6
├── update-position/
│   └── [id].post.ts                # AC-7
├── members/
│   └── [id].post.ts                # AC-8
├── [cardId]/
│   └── delete-member/
│       └── [userId].post.ts        # AC-9
├── tags/
│   └── [id].post.ts                # AC-10
└── update-status/
    └── [id].post.ts                # AC-11

server/transformers/
└── card.transformer.ts             # transformCard()

server/utils/
├── soft-delete.ts                  # softDeleteCard(), restoreCard()
└── broadcast.ts                    # broadcastToCentrifugo()

shared/schemas/
└── card.schema.ts                  # All card Zod schemas

shared/enums/
└── CardUserStatusEnum.ts           # None=0, Pending=1, Doing=2, OnHold=3, Done=4

app/components/card/
└── Item.vue                        # Kanban card chip: name, tag chips, assignee avatars, status badge

app/composables/
└── use-card-modal.ts               # useCardModal: openCardModal ref, isCreatingCommentWithAttachment

app/composables/api/
└── use-card-api.ts                 # Wraps all $fetch / useFetch calls for card endpoints

app/modals/
└── task-detail.vue                 # Card detail modal: full card view with members, tags, comments, attachments
```

---

## Data Model

### API Endpoints Consumed

| Method | Endpoint | Request Body | Response |
|--------|----------|-------------|---------|
| `POST` | `/api/client/card/create/[boardListId]` | `createCardSchema` | `201` `transformCard` |
| `POST` | `/api/client/card/quick-create/[boardListId]` | `quickCreateCardSchema` | `201` `transformCard` |
| `POST` | `/api/client/card/duplicate/[id]` | _(none)_ | `201` `transformCard` |
| `GET`  | `/api/client/card/show/[id]` | _(none)_ | `200` `transformCard` |
| `PUT`  | `/api/client/card/[id]` | `updateCardSchema` | `200` `transformCard` |
| `DELETE` | `/api/client/card/[id]` | _(none)_ | `200` success message |
| `POST` | `/api/client/card/update-position/[id]` | `updateCardPositionSchema` | `200` success message |
| `POST` | `/api/client/card/members/[id]` | `syncCardMembersSchema` | `200` `transformCard` |
| `POST` | `/api/client/card/[cardId]/delete-member/[userId]` | _(none)_ | `200` success message |
| `POST` | `/api/client/card/tags/[id]` | `syncCardTagsSchema` | `200` `transformCard` |
| `POST` | `/api/client/card/update-status/[id]` | `updateCardStatusSchema` | `200` `transformCard` |

### TypeScript Interfaces

```typescript
// shared/types/card.types.ts

export interface CardAttachment {
  id: number
  name: string
  path: string
  type: string
  created_at: string
}

export interface CardComment {
  id: number
  user_id: number
  card_id: number
  content: string
  is_pinned: boolean
  created_at: string
  updated_at: string
  user: CardCommentUser
  attachments: CardAttachment[]
}

export interface CardCommentUser {
  id: number
  name: string
  avatar: string | null
}

export interface CardTag {
  id: number
  name: string
  color: {
    name: string  // e.g., "Violet"
    value: string // e.g., "#7C3AED"
  }
}

export interface CardUser {
  id: number
  name: string
  avatar: string | null
  status: CardUserStatus  // pivot from card_user
}

export type CardUserStatus = 0 | 1 | 2 | 3 | 4

export interface Card {
  id: number
  boardlist_id: number
  name: string
  description: string | null
  position: number
  tags: CardTag[]
  users: CardUser[]
  comments: CardComment[]
  card_attachments: CardAttachment[]       // attachments NOT linked to a comment
  comment_attachments: CardAttachment[]    // attachments linked to a comment
}

// Request shapes (mirror Zod schemas)
export interface CreateCardRequest {
  name: string
  description?: string
  position: number
  user_ids?: number[]
  tag_ids?: number[]
}

export interface QuickCreateCardRequest {
  name: string
}

export interface UpdateCardRequest {
  name?: string
  description?: string
  board_list_id?: number
  position?: number
}

export interface SyncCardMembersRequest {
  user_ids: number[]
}

export interface SyncCardTagsRequest {
  tag_ids: number[]
}

export interface UpdateCardStatusRequest {
  user_id: number
  status: CardUserStatus
}

export interface UpdateCardPositionRequest {
  board_list_id: number
  items: Array<{ id: number; position: number }>
}
```

```typescript
// shared/enums/CardUserStatusEnum.ts

export const CardUserStatusEnum = {
  None:    0,
  Pending: 1,
  Doing:   2,
  OnHold:  3,
  Done:    4,
} as const

export type CardUserStatusEnumValue = typeof CardUserStatusEnum[keyof typeof CardUserStatusEnum]

// UI display colors
export const CardUserStatusColorMap: Record<CardUserStatusEnumValue, string> = {
  0: '#B6B1BE', // None
  1: '#EAB035', // Pending
  2: '#A777E1', // Doing
  3: '#E55151', // On hold
  4: '#48CC4C', // Done
}
```

### Transformer

```typescript
// server/transformers/card.transformer.ts (illustrative)

// Input: card DB row + { users, comments, attachments }
// Output: Card interface (see above)
// - users: includes pivot status from card_user
// - tags: each tag has expanded color { name, value }
// - attachments split:
//     card_attachments     → where comment_id IS NULL
//     comment_attachments  → where comment_id IS NOT NULL
transformCard(cardRow, { users, comments, attachments }): Card
```

---

## State Management

### `useCardModal` composable (`app/composables/use-card-modal.ts`)

```typescript
interface OpenCardModalOptions {
  cardId: number
  cardData?: Card
  refreshCardData?: () => Promise<void>
}

// Composable returns:
const openCardModal: Ref<OpenCardModalOptions | null>
const isCreatingCommentWithAttachment: Ref<boolean>

function openModal(options: OpenCardModalOptions): void
function closeModal(): void
```

- `openCardModal` is `null` when no modal is open; setting it to an options object opens the card detail modal.
- `isCreatingCommentWithAttachment` flag is used inside the modal to indicate an in-progress comment-with-attachment flow, preventing accidental close.

### Board Store (`app/stores/board.ts`) — Card slice

- Board store holds `lists: BoardList[]`, each list containing `cards: Card[]`.
- On `card_create` Centrifugo event → push new card to the correct list in store.
- On `card_update` event → find card in store by ID and merge updated fields.
- On `card_delete` event → remove card from store by ID.
- On `card_position` event → re-sort cards in affected list(s).
- On `card_add_users` / `card_remove_users` events → update `users[]` on the relevant card.

### `use-card-api` composable (`app/composables/api/use-card-api.ts`)

All card API calls are centralised here. Components call composable functions; they never call `$fetch` directly.

```typescript
function createCard(boardListId: number, body: CreateCardRequest): Promise<Card>
function quickCreateCard(boardListId: number, body: QuickCreateCardRequest): Promise<Card>
function duplicateCard(id: number): Promise<Card>
function showCard(id: number): Promise<Card>
function updateCard(id: number, body: UpdateCardRequest): Promise<Card>
function deleteCard(id: number): Promise<void>
function updateCardPosition(id: number, body: UpdateCardPositionRequest): Promise<void>
function syncCardMembers(id: number, body: SyncCardMembersRequest): Promise<Card>
function deleteCardMember(cardId: number, userId: number): Promise<void>
function syncCardTags(id: number, body: SyncCardTagsRequest): Promise<Card>
function updateCardStatus(id: number, body: UpdateCardStatusRequest): Promise<Card>
```

---

## Edge Cases

1. **Concurrent position updates**: Two users drag the same card simultaneously. The last `update-position` call wins. No optimistic locking — acceptable for this scope.
2. **Duplicate of a soft-deleted card**: Returns `404`. Cannot duplicate a deleted card.
3. **Sync members with empty array**: Passing `user_ids: []` removes all members from the card.
4. **Sync tags with empty array**: Passing `tag_ids: []` clears all tags on the card.
5. **Quick-create — position collision**: If two cards are created simultaneously at the same position, Drizzle batch insert can result in duplicate positions. Position is treated as a display hint; the frontend re-orders by returned position.
6. **Status update for non-member**: Returns `404` — user must be in `card_user` to have a status.
7. **Delete-member for user not on card**: Returns `404` — no row to delete.
8. **Board access check**: All card endpoints must verify the requesting user has access to the card's parent board (via board membership), not just that the card exists.
9. **Large `items` array in update-position**: Batch update must use a Drizzle transaction to avoid partial updates if one fails.
10. **`isCreatingCommentWithAttachment` guard**: The card detail modal must not close while this flag is `true` to prevent losing an in-progress file upload.

---

## Non-Goals

- Card **restore** (un-delete) is an admin-only operation — not exposed via client endpoints in this spec.
- File attachment upload/delete is handled by the `client-attachments` spec — not duplicated here.
- Comment create/edit/delete/pin is handled by the `client-comments` spec.
- Admin-level card management (view soft-deleted cards, bulk restore) is out of scope.
- Card **copy to a different board list** is not supported — duplicate always copies to the same list.
- Real-time conflict resolution or operational transform for simultaneous edits is out of scope.
- Pagination of comments or attachments on the Show Card endpoint — all are returned in full.

---

## Dependencies

| Dependency | Used for |
|-----------|---------|
| `server/middleware/auth.ts` | JWT validation on all card endpoints |
| `server/middleware/active-user.ts` | Ensures user account is not suspended |
| `server/transformers/card.transformer.ts` | Standardises all card API responses |
| `server/utils/broadcast.ts` → `broadcastToCentrifugo()` | Centrifugo real-time events |
| `server/utils/soft-delete.ts` → `softDeleteCard()`, `restoreCard()` | Soft delete logic |
| `server/services/notification.ts` → `NotificationService` | BullMQ-backed notification dispatch |
| `server/database/schema/cards.ts` | `cards` table schema with `deletedAt` column |
| `server/database/schema/card-user.ts` | `card_user` pivot (userId, cardId, status) |
| `server/database/schema/card-tag.ts` | `card_tag` pivot (cardId, tagId) |
| `shared/schemas/card.schema.ts` | Zod validation for all card request bodies |
| `shared/enums/CardUserStatusEnum.ts` | Status values 0–4 + display colors |
| `specs/client-comments.md` | Comment endpoints and attachment linking |
| `specs/client-attachments.md` | File upload/delete for card attachments |
| `specs/notification-system.md` | Notification types and BullMQ job |
| `specs/realtime-centrifugo.md` | Centrifugo broadcast utility and channel conventions |
| `specs/api-transformers.md` | `transformCard` contract and `apiResponse`/`apiError` helpers |
| `specs/audit-trail.md` | `CardLog` integration for card activity feed |
