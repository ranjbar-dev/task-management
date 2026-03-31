# Spec: Client User Profile

## Status: Approved

## Purpose

Provide authenticated users with a self-service profile management page where they can view and update their personal information (name, username, avatar), change their password, manage their Telegram notification channel connection, and configure their 12 notification preferences — all against the `/api/client/user` and `/api/client/telegram` route families.

---

## User Stories

- As a logged-in user, I want to see my profile info (name, username, avatar, role) so I know what the system has on record for me.
- As a logged-in user, I want to update my name and username so I can keep my identity current.
- As a logged-in user, I want to upload or replace my avatar so my profile picture stays current.
- As a logged-in user, I want to delete my avatar and fall back to a default so I can remove a photo I no longer want.
- As a logged-in user, I want to change my password by providing my current password first so only I can authorise the change.
- As a logged-in user, I want to see whether my Telegram account is connected so I know my notification channel status.
- As a logged-in user, I want to connect Telegram by following a deep-link so I can receive push notifications.
- As a logged-in user, I want to disconnect Telegram after confirming so I can stop receiving notifications there.
- As a logged-in user, I want to toggle each of the 12 notification preferences on or off so I control which events create notifications.

---

## Acceptance Criteria

### AC-1 — List Users (`GET /api/client/user/all`)
- Returns an array of all users transformed via `transformUser`.
- Requires `auth` + `active-user` middleware; unauthenticated requests receive 401.
- Response shape: `{ data: UserModel[] }`.

### AC-2 — Get User (`GET /api/client/user/show/[id]`)
- Returns a single user transformed via `transformUser`.
- Requires `auth` + `active-user` middleware.
- Returns 404 when the requested `id` does not exist.
- Response shape: `{ data: UserModel }`.

### AC-3 — Update Profile (`PUT /api/client/user/[id]`)
- Accepts `name`, `username`, and/or `notification` fields.
- Request body validated against `updateProfileSchema` (Zod); returns 422 on validation failure.
- Only the authenticated user may update their own record; returns 403 otherwise.
- `notification` field is a flat object of 12 boolean keys; merged into the existing JSONB column — keys not sent are preserved unchanged.
- On success returns the updated `UserModel` via `transformUser`.
- `authStore.user` is refreshed client-side after a successful update.

### AC-4 — Upload Avatar (`POST /api/client/user/update-avatar/[id]`)
- Request is `multipart/form-data` with a single `file` field.
- If the user already has an avatar, the old file is deleted from `storage/avatar/` before the new one is written.
- On success, `users.avatar` is updated in the DB and the updated `UserModel` is returned.
- Only the authenticated user may upload their own avatar; returns 403 otherwise.
- Returns 422 if no file is provided.

### AC-5 — Delete Avatar (`POST /api/client/user/delete-avatar/[id]`)
- Deletes the file from `storage/avatar/` via `deleteFile()`.
- Sets `users.avatar = null` in the DB.
- Returns the updated `UserModel` with `avatar: null`.
- Only the authenticated user may delete their own avatar; returns 403 otherwise.
- No-ops gracefully when `avatar` is already `null` (returns 200 with unchanged model).

### AC-6 — Change Password (`POST /api/client/user/update-password/[id]`)
- Request body validated against `updatePasswordSchema`; returns 422 on validation failure.
- `current_password` is verified against the stored hash via `verifyPassword()`; returns 401 if incorrect.
- `new_password` and `confirm_password` must match (enforced by Zod `.refine()`); returns 422 if they differ.
- New password is hashed via `hashPassword()` before being saved to the DB.
- Only the authenticated user may change their own password; returns 403 otherwise.
- Returns 200 with a success message on completion; does not return the password hash.

### AC-7 — Telegram Connect (`GET /api/client/telegram/connect`)
- Generates a Telegram deep-link URL in the format `https://t.me/{botName}?start={token}`.
- Stores the token against the user in the DB (used by the bot-side registration flow).
- Response shape: `{ data: { url: string } }`.
- Requires `auth` + `active-user` middleware.

### AC-8 — Telegram Disconnect (`POST /api/client/telegram/disconnect`)
- Clears `users.telegram.chat_id` in the DB; preserves the token so reconnect is possible.
- Requires `auth` + `active-user` middleware.
- Returns 200 with the updated `UserModel`.
- Confirmation modal is shown client-side before the API call is made.

### AC-9 — Profile Page UI (`/profile`)
- Displays current user's avatar (or fallback), name, username, and role label.
- Shows Telegram connection status badge (connected / not connected).
- All three edit sections (profile info, password, notification toggles) are visible on the same page, either as cards or accordion panels.
- Page uses the `default` layout and is protected by the `auth` route middleware (redirects to `/login` if unauthenticated).
- Page data is initialised from `authStore.user` — no additional fetch required on mount if the store is already populated.

### AC-10 — Notification Preferences UI
- Renders 12 toggle switches, one per preference key.
- Toggle state reflects the current values from `authStore.user.notification`.
- Each toggle calls `PUT /api/client/user/[id]` with the updated `notification` object on change.
- Debounce or batched save (≥ 300 ms) prevents excessive API calls from rapid toggling.
- A success toast confirms each save; an error toast surfaces API failures.

### AC-11 — Avatar Upload/Delete UI
- "Upload avatar" triggers a hidden `<input type="file" accept="image/*">`.
- Selected file is submitted via `$fetch` with `multipart/form-data`.
- Preview updates immediately on success.
- "Delete avatar" shows a confirmation dialog then calls `delete-avatar`.
- Both actions refresh `authStore.user` on completion.

---

## Component Tree / File Structure

```
app/
├── pages/
│   └── profile/
│       └── index.vue                     # /profile route — profile page
├── components/
│   └── card/
│       └── profile.vue                   # Profile display card (avatar, name, username, role)
├── components/                           # Shared UI components used on profile page
│   ├── ConnectTelegram.vue               # Deep-link button + connection status badge
│   └── DisconnectTelegramModal.vue       # Confirmation modal before disconnect
└── composables/
    └── api/
        └── use-user-api.ts               # All /api/client/user/* calls
        └── use-telegram-api.ts           # All /api/client/telegram/* calls

server/
└── api/
    └── client/
        ├── user/
        │   ├── all.get.ts
        │   ├── show/
        │   │   └── [id].get.ts
        │   ├── [id].put.ts
        │   ├── update-avatar/
        │   │   └── [id].post.ts
        │   ├── delete-avatar/
        │   │   └── [id].post.ts
        │   └── update-password/
        │       └── [id].post.ts
        └── telegram/
            ├── connect.get.ts
            └── disconnect.post.ts

shared/
└── schemas/
    └── user.schema.ts                    # updateProfileSchema, updatePasswordSchema
```

---

## Data Model

### API Endpoints Consumed

| Method | Endpoint | Request | Response |
|--------|----------|---------|----------|
| `GET` | `/api/client/user/all` | — | `{ data: UserModel[] }` |
| `GET` | `/api/client/user/show/[id]` | — | `{ data: UserModel }` |
| `PUT` | `/api/client/user/[id]` | `UpdateProfileRequest` | `{ data: UserModel }` |
| `POST` | `/api/client/user/update-avatar/[id]` | `multipart/form-data` (`file`) | `{ data: UserModel }` |
| `POST` | `/api/client/user/delete-avatar/[id]` | — | `{ data: UserModel }` |
| `POST` | `/api/client/user/update-password/[id]` | `UpdatePasswordRequest` | `{ message: string }` |
| `GET` | `/api/client/telegram/connect` | — | `{ data: ConnectTelegramResponse }` |
| `POST` | `/api/client/telegram/disconnect` | — | `{ data: UserModel }` |

### TypeScript Interfaces

```typescript
// shared/types/user.types.ts

interface UserModel {
  id: number
  name: string
  username: string
  avatar: string | null
  role: number                            // UserRoleEnum
  telegram: {
    id: string
    chat_id?: string
    username?: string
  } | null
  is_disable: boolean
}

interface UpdateProfileRequest {
  name?: string                           // min 1, max 50 (validated by updateProfileSchema)
  username?: string                       // min 1, max 50 (validated by updateProfileSchema)
  notification?: Record<string, boolean>  // 12 preference keys (merged into JSONB column)
}

interface UpdatePasswordRequest {
  current_password: string               // min 8
  new_password: string                   // min 8
  confirm_password: string               // min 8, must equal new_password
}

interface ConnectTelegramResponse {
  url: string                            // "https://t.me/{botName}?start={token}"
}
```

### Notification Preference Keys

```typescript
// 12 boolean keys stored in users.notification JSONB
type NotificationPreferences = {
  boardStatusChange: boolean
  boardTitleChange: boolean
  joinToBoard: boolean
  removeFromBoard: boolean
  assignToCard: boolean
  removeFromCard: boolean
  cardStatusChange: boolean
  cardNewTag: boolean
  cardNewComment: boolean
  cardNewAttachment: boolean
  cardTitleChange: boolean
  cardDescriptionChange: boolean
}
```

---

## State Management

| Store | Relevant State | Actions / Updates |
|-------|---------------|-------------------|
| `authStore` (`app/stores/auth.ts`) | `user: UserModel`, `token`, `is_authenticated` | Refreshed after profile update, avatar change, Telegram connect/disconnect |

- The profile page reads `authStore.user` directly — no separate local reactive copy unless an edit form requires it.
- After any successful mutation, `authStore.user` is patched with the returned `UserModel` so the sidebar avatar/name reflects changes immediately.
- Notification preference toggles read from and write back to `authStore.user.notification`.

---

## Edge Cases

| Scenario | Expected Behaviour |
|----------|-------------------|
| User submits update-profile with no changed fields | Server returns 200 with unchanged `UserModel`; no DB write needed |
| Avatar upload file exceeds server size limit | Server returns 413; client shows error toast "File too large" |
| Avatar upload with non-image MIME type | Server returns 422; client shows error toast "Invalid file type" |
| `delete-avatar` when `avatar` is already `null` | Server returns 200 with unchanged model; no file-system operation attempted |
| `update-password` with wrong `current_password` | Server returns 401; client shows error toast "Current password is incorrect" |
| `update-password` where `new_password ≠ confirm_password` | Zod returns 422 with `confirm_password` path error; client highlights the field |
| Telegram connect when already connected (`chat_id` present) | Server generates a new token and returns a fresh deep-link URL; existing connection is overwritten on bot-side registration |
| Telegram disconnect when not connected | Server returns 200; no-op on DB |
| Rapid notification preference toggling | Client debounces 300 ms before sending `PUT`; only the final state is sent |
| Network failure during any mutation | Error toast surfaced; local UI state rolled back to pre-mutation value |
| Unauthenticated request to any endpoint | Middleware returns 401; client redirected to `/login` |
| User attempts to update another user's profile | Server returns 403 |

---

## Non-Goals

- Admin-level user management (covered by `admin-user-management.md`).
- Telegram bot command handling and incoming-message polling (covered by `telegram-integration.md`).
- Notification log viewer (covered by `admin-notification-logs.md`).
- Email-based password reset / forgot-password flow — not in scope for this spec.
- OAuth / social login integration.
- Two-factor authentication (2FA).
- Public user profile pages visible to unauthenticated visitors.

---

## Dependencies

| Dependency | Used For |
|------------|----------|
| `shared/schemas/user.schema.ts` | `updateProfileSchema`, `updatePasswordSchema` Zod validation |
| `server/utils/password.ts` | `verifyPassword()`, `hashPassword()` |
| `server/utils/jwt.ts` | Auth middleware token extraction |
| `server/services/fileStorage.ts` | `uploadFile()`, `deleteFile()` for avatar management |
| `server/transformers/user.transformer.ts` | `transformUser()` — shapes all `UserModel` responses |
| `server/middleware/auth.ts` | JWT verification on all endpoints |
| `server/middleware/active-user.ts` | `is_disable` check on all endpoints |
| `server/database/schema/users.ts` | `users` Drizzle table (name, username, avatar, password, notification JSONB, telegram JSONB) |
| `app/stores/auth.ts` | Pinia authStore — source of truth for current user on the client |
| `telegram-integration.md` spec | Defines the token/registration flow that `connect.get.ts` initiates |
| `notification-system.md` spec | Defines the 12 preference keys consumed by the notification toggle UI |
