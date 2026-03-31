# Tasks: Client User Profile
**Spec:** `specs/client-user-profile.md`
**Status:** pending

---

## Overview

Implements the self-service profile management page at `/profile`. Covers 6 user API endpoints (`all`, `show/[id]`, `[id]` update, `update-avatar`, `delete-avatar`, `update-password`) and 2 Telegram endpoints (`connect`, `disconnect`), along with the corresponding client composables, profile page, notification preference toggles (12 keys with debounce), avatar upload/delete UI, and Telegram connect/disconnect components.

Execution order: Wave 1 (API routes + composables) → Wave 2 (Frontend page + components) → Wave 3 (Tests).

---

## Wave 1 — Data Layer (API Agent)

### Task 020-A: User Route Handlers & Client Composable

- **Agent:** API
- **Spec:** `specs/client-user-profile.md`
- **Section:** Acceptance Criteria AC-1 through AC-6; Data Model → API Endpoints; Data Model → TypeScript Interfaces; Dependencies
- **Acceptance Criteria:**
  - [ ] AC-1: `GET /api/client/user/all` returns `{ data: UserModel[] }` with every user transformed via `transformUser`; requires `auth` + `active-user` middleware; unauthenticated requests receive 401.
  - [ ] AC-2: `GET /api/client/user/show/[id]` returns `{ data: UserModel }` for the requested id; requires `auth` + `active-user`; returns 404 when the id does not exist in the DB.
  - [ ] AC-3: `PUT /api/client/user/[id]` accepts `name`, `username`, and/or `notification` fields; validates request body against `updateProfileSchema` (422 on failure); returns 403 if the authenticated user is not the record owner; merges the `notification` flat object into the existing JSONB column — keys not sent are preserved unchanged; returns the updated `UserModel` via `transformUser`.
  - [ ] AC-4: `POST /api/client/user/update-avatar/[id]` accepts `multipart/form-data` with a single `file` field; deletes the existing avatar from `storage/avatar/` via `deleteFile()` before writing the new one; updates `users.avatar` in the DB; returns the updated `UserModel`; returns 403 if not own record; returns 422 if no file is provided.
  - [ ] AC-5: `POST /api/client/user/delete-avatar/[id]` deletes the avatar file from `storage/avatar/` via `deleteFile()`; sets `users.avatar = null` in the DB; no-ops gracefully when `avatar` is already `null` (returns 200 with unchanged model); returns the updated `UserModel`; returns 403 if not own record.
  - [ ] AC-6: `POST /api/client/user/update-password/[id]` validates request body against `updatePasswordSchema` (422 on failure); verifies `current_password` against the stored hash via `verifyPassword()` (401 if incorrect); `new_password` and `confirm_password` must match as enforced by Zod `.refine()` (422 if they differ); hashes the new password via `hashPassword()` before saving; returns 403 if not own record; returns 200 with a success message — does not return the password hash.
- **Dependencies:** `tasks/004-authentication.md`, `tasks/005-api-transformers.md`, `tasks/003-validation-schemas.md`, `tasks/019-client-attachments.md`
- **Output Files:**
  - `server/api/client/user/all.get.ts`
  - `server/api/client/user/show/[id].get.ts`
  - `server/api/client/user/[id].put.ts`
  - `server/api/client/user/update-avatar/[id].post.ts`
  - `server/api/client/user/delete-avatar/[id].post.ts`
  - `server/api/client/user/update-password/[id].post.ts`
  - `app/composables/api/use-user-api.ts`

---

### Task 020-B: Telegram Route Handlers & Client Composable

- **Agent:** API
- **Spec:** `specs/client-user-profile.md`
- **Section:** Acceptance Criteria AC-7, AC-8; Data Model → API Endpoints; Data Model → TypeScript Interfaces (`ConnectTelegramResponse`); Edge Cases
- **Acceptance Criteria:**
  - [ ] AC-7: `GET /api/client/telegram/connect` generates a Telegram deep-link URL in the format `https://t.me/{botName}?start={token}`; stores the token against the user in the DB; returns `{ data: { url: string } }`; requires `auth` + `active-user` middleware. When user is already connected (`chat_id` present), generates a fresh token and returns a new URL (existing connection overwritten on bot-side registration).
  - [ ] AC-8 (server): `POST /api/client/telegram/disconnect` clears `users.telegram.chat_id` in the DB while preserving the token; requires `auth` + `active-user` middleware; returns 200 with the updated `UserModel`; no-ops when `chat_id` is already absent (returns 200).
- **Dependencies:** `tasks/004-authentication.md`, `tasks/005-api-transformers.md`, `tasks/009-telegram-integration.md`
- **Output Files:**
  - `server/api/client/telegram/connect.get.ts`
  - `server/api/client/telegram/disconnect.post.ts`
  - `app/composables/api/use-telegram-api.ts`

---

## Wave 2 — Frontend (Frontend Agent)

### Task 020-C: Profile Page, Edit Sections & Avatar UI

- **Agent:** Frontend
- **Spec:** `specs/client-user-profile.md`
- **Section:** Acceptance Criteria AC-9, AC-10, AC-11; Component Tree; State Management; Edge Cases
- **Dependencies:** Task 020-A, Task 020-B
- **Acceptance Criteria:**
  - [ ] AC-9: Page at `/profile` renders current user's avatar (or placeholder fallback), name, username, and role label.
  - [ ] AC-9: Telegram connection status badge (connected / not connected) is derived from `authStore.user.telegram.chat_id`.
  - [ ] AC-9: All three edit sections — profile info (name/username), change password, and notification toggles — are visible on the same page (cards or accordion panels).
  - [ ] AC-9: Page uses the `default` layout and is protected by the `auth` route middleware; unauthenticated users are redirected to `/login`.
  - [ ] AC-9: Page data is initialised from `authStore.user` — no additional fetch on mount if the store is already populated.
  - [ ] AC-9: After any successful mutation (profile update, avatar change, password change), `authStore.user` is patched with the returned `UserModel` so the sidebar avatar/name reflects changes immediately.
  - [ ] AC-10: Renders 12 toggle switches — one per preference key: `boardStatusChange`, `boardTitleChange`, `joinToBoard`, `removeFromBoard`, `assignToCard`, `removeFromCard`, `cardStatusChange`, `cardNewTag`, `cardNewComment`, `cardNewAttachment`, `cardTitleChange`, `cardDescriptionChange`.
  - [ ] AC-10: Toggle state is initialised from and bound to `authStore.user.notification`.
  - [ ] AC-10: Each toggle change calls `PUT /api/client/user/[id]` via `use-user-api.ts` with the full updated `notification` object.
  - [ ] AC-10: Saves are debounced ≥ 300 ms to prevent excessive API calls from rapid toggling — only the final state is sent.
  - [ ] AC-10: Success toast confirms each save; error toast surfaces API failures; local toggle state is rolled back to the pre-mutation value on network failure.
  - [ ] AC-11: "Upload avatar" triggers a hidden `<input type="file" accept="image/*">`; the selected file is submitted via `$fetch` as `multipart/form-data`; avatar preview updates immediately on success; `authStore.user` is refreshed from the response.
  - [ ] AC-11: "Delete avatar" shows a confirmation dialog before calling the `delete-avatar` endpoint; avatar reverts to fallback on success; `authStore.user` is refreshed from the response.
  - [ ] AC-11 (edge): Server 413 response shows error toast "File too large"; 422 shows "Invalid file type".
- **Output Files:**
  - `app/pages/profile/index.vue`
  - `app/components/card/profile.vue`

---

### Task 020-D: Telegram Connect/Disconnect UI

- **Agent:** Frontend
- **Spec:** `specs/client-user-profile.md`
- **Section:** Acceptance Criteria AC-7 (client), AC-8 (client); Component Tree; State Management; Edge Cases
- **Dependencies:** Task 020-B, Task 020-C
- **Acceptance Criteria:**
  - [ ] AC-7 (client): `connect-telegram.vue` renders a connection status badge derived from `authStore.user.telegram.chat_id`; a "Connect Telegram" button calls `use-telegram-api.ts` to fetch the deep-link URL and opens it (e.g. `window.open`); button is hidden or replaced with "Connected" state when `chat_id` is present.
  - [ ] AC-8 (client): `modal/telegram/disconnect.vue` is a confirmation modal that must be dismissed or confirmed before the `POST /api/client/telegram/disconnect` call is made; `authStore.user` is patched with the returned `UserModel` on success.
  - [ ] AC-8 (edge): Disconnect when not connected still returns 200; the UI dismisses the modal and does not show an error.
- **Output Files:**
  - `app/components/connect-telegram.vue`
  - `app/components/modal/telegram/disconnect.vue`

---

## Wave 3 — Tests (Testing Agent)

### Task 020-E: Server Endpoint Unit Tests

- **Agent:** Testing
- **Spec:** `specs/client-user-profile.md`
- **Section:** Acceptance Criteria AC-1 through AC-8; Edge Cases
- **Dependencies:** Task 020-A, Task 020-B
- **Acceptance Criteria Tested:**
  - [ ] AC-1: `all.get` — returns `UserModel[]` array for authenticated user; returns 401 when unauthenticated.
  - [ ] AC-2: `show/[id].get` — returns `UserModel` for valid id; returns 404 for unknown id; returns 401 when unauthenticated.
  - [ ] AC-3: `[id].put` — updates `name` and `username`; merges partial `notification` object (absent keys preserved); returns 422 on invalid body; returns 403 when authenticated user ≠ target id.
  - [ ] AC-4: `update-avatar/[id].post` — uploads file, deletes old avatar, updates DB, returns `UserModel`; returns 422 when no file provided; returns 403 on wrong user.
  - [ ] AC-5: `delete-avatar/[id].post` — nulls `users.avatar`, deletes file on disk; returns 200 no-op when avatar is already `null`; returns 403 on wrong user.
  - [ ] AC-6: `update-password/[id].post` — changes password successfully with correct `current_password`; returns 401 on incorrect `current_password`; returns 422 when `new_password ≠ confirm_password`; returns 403 on wrong user; does not include password hash in response.
  - [ ] AC-7: `telegram/connect.get` — returns `{ data: { url: string } }` with correct deep-link format; token is stored in DB; generates fresh URL when already connected.
  - [ ] AC-8: `telegram/disconnect.post` — clears `chat_id` while preserving token; returns updated `UserModel`; returns 200 and no-ops when `chat_id` is already absent.
- **Output Files:**
  - `tests/unit/server/api/client/user/all.get.test.ts`
  - `tests/unit/server/api/client/user/show-id.get.test.ts`
  - `tests/unit/server/api/client/user/id.put.test.ts`
  - `tests/unit/server/api/client/user/update-avatar.post.test.ts`
  - `tests/unit/server/api/client/user/delete-avatar.post.test.ts`
  - `tests/unit/server/api/client/user/update-password.post.test.ts`
  - `tests/unit/server/api/client/telegram/connect.get.test.ts`
  - `tests/unit/server/api/client/telegram/disconnect.post.test.ts`

---

### Task 020-F: Component Tests & E2E Profile Flow

- **Agent:** Testing
- **Spec:** `specs/client-user-profile.md`
- **Section:** Acceptance Criteria AC-9 through AC-11; Edge Cases
- **Dependencies:** Task 020-C, Task 020-D
- **Acceptance Criteria Tested:**
  - [ ] AC-9: Profile page mounts with correct user data from `authStore.user`; `auth` route middleware redirects unauthenticated visitors to `/login`; no extra fetch is triggered when store is already populated.
  - [ ] AC-10: All 12 notification toggle switches render; toggling one fires a debounced `PUT` (≥ 300 ms); success toast shown on response; error toast shown on failure; toggle reverts on network failure.
  - [ ] AC-11:"Upload avatar" — file input interaction triggers multipart `$fetch`; avatar preview updates on success; 413 response shows "File too large" toast; 422 shows "Invalid file type" toast.
  - [ ] AC-11: "Delete avatar" — confirmation dialog blocks immediate action; confirm triggers endpoint call; avatar reverts to fallback.
  - [ ] AC-7/8 (component): Telegram connect button fetches deep-link; disconnect modal renders before API call is made; modal dismiss cancels the action.
  - [ ] E2E: Full profile update flow — navigate to `/profile`, update name/username, verify `authStore.user` is patched and the updated values are reflected in the UI without a page reload.
- **Output Files:**
  - `tests/unit/app/components/card/profile.test.ts`
  - `tests/unit/app/components/connect-telegram.test.ts`
  - `tests/e2e/profile.spec.ts`
