# Spec: Admin User Management

## Status: Approved

---

## Purpose

Provide administrators with a full CRUD interface for managing application users. This covers listing, creating, viewing, updating, and deleting users, as well as changing passwords. All operations are restricted to users with `role === UserRoleEnum.AdminUser` and are accessed through a dedicated admin page with modal-based workflows.

---

## User Stories

- As an admin, I can view a paginated, searchable table of all users so that I can monitor and manage the user base.
- As an admin, I can create a new user with a name, username, password, role, and disabled status so that onboarding is handled centrally.
- As an admin, I can view the full details of any user so that I can audit their current state.
- As an admin, I can update a user's name, username, role, and disabled status so that their account reflects current requirements.
- As an admin, I can change any user's password so that access can be recovered or rotated without user action.
- As an admin, I can delete a user so that inactive or unauthorized accounts are removed from the system.
- As an admin, I am prevented from creating a user with a duplicate username so that login credentials remain unambiguous.

---

## Acceptance Criteria

### AC-1: List All Users
- `GET /api/admin/user/all` returns an array of all users transformed via `transformUser`.
- Response excludes the `password` field for every user.
- Requires `auth`, `active-user`, and `admin` middleware; returns `401` / `403` otherwise.

### AC-2: Paginated User Table
- `POST /api/admin/user/table` accepts SearchTable service parameters (pagination, sort, filters).
- Returns a paginated result set transformed via `transformUser`.
- Requires `auth`, `active-user`, and `admin` middleware.

### AC-3: Get Single User
- `GET /api/admin/user/[id]` returns the full user record for the given ID via `transformUser`.
- Returns `404` when the user does not exist.
- Requires `auth`, `active-user`, and `admin` middleware.

### AC-4: Create User
- `POST /api/admin/user/create` validates the request body against `createUserSchema`.
  - Required: `name` (1–50 chars), `username` (1–50 chars), `password` (min 8 chars).
  - Optional: `role` (0 or 1, defaults to `UserRoleEnum.NormalUser`), `is_disable` (boolean, defaults to `false`).
- Returns `422` with field-level errors on validation failure.
- Returns `409` (or equivalent) when `username` is already taken.
- Password is hashed with `bcrypt` at `SALT_ROUNDS = 10` before persisting.
- After insert, `initializeUserDefaults()` is called:
  - Sets `telegram.id` to a 12-character random token.
  - Sets all 12 notification preferences to `true`.
- Returns the created user via `transformUser` with HTTP `201`.
- Requires `auth`, `active-user`, and `admin` middleware.

### AC-5: Update User
- `PUT /api/admin/user/[id]` validates the request body against `updateUserSchema`.
  - All fields optional: `name`, `username`, `role`, `is_disable`.
- Returns `404` when the user does not exist.
- Returns `422` with field-level errors on validation failure.
- Returns `409` (or equivalent) if the new `username` conflicts with an existing user.
- When `role` is set to `UserRoleEnum.AdminUser` (or when the user otherwise lacks a personal board), auto-creates a private board for the user if one does not already exist.
- Returns the updated user via `transformUser`.
- Requires `auth`, `active-user`, and `admin` middleware.

### AC-6: Change User Password
- `POST /api/admin/user/update-password/[id]` validates the request body against `updatePasswordSchema`.
  - Required: `current_password` (min 8), `new_password` (min 8), `confirm_password` (min 8).
  - `new_password` must equal `confirm_password`; returns `422` otherwise.
- Returns `404` when the user does not exist.
- Verifies `current_password` against the stored hash via `verifyPassword`; returns `400` / `401` on mismatch.
- Hashes `new_password` with `bcrypt` at `SALT_ROUNDS = 10` before persisting.
- Returns HTTP `200` on success.
- Requires `auth`, `active-user`, and `admin` middleware.

### AC-7: Delete User
- `DELETE /api/admin/user/[id]` permanently removes the user record.
- Returns `404` when the user does not exist.
- Returns HTTP `200` (or `204`) on success.
- Requires `auth`, `active-user`, and `admin` middleware.

### AC-8: Frontend Page
- `app/pages/user/index.vue` uses the `default` layout and is only accessible to `AdminUser` role (enforced by route middleware).
- Renders a paginated, searchable user table powered by `/api/admin/user/table`.
- Provides action buttons to open Add, Edit, Detail, Password, and Delete modals per row.

### AC-9: Modals
- **User Add modal** — form with `name`, `username`, `password`, `role`, `is_disable`; calls `POST /api/admin/user/create`; refreshes table on success.
- **User Detail modal** — read-only display of user fields from `GET /api/admin/user/[id]`.
- **User Edit modal** — pre-populated form with `name`, `username`, `role`, `is_disable`; calls `PUT /api/admin/user/[id]`; refreshes table on success.
- **User Password modal** — form with `current_password`, `new_password`, `confirm_password`; calls `POST /api/admin/user/update-password/[id]`.
- **User Delete confirmation modal** — confirm prompt; calls `DELETE /api/admin/user/[id]`; refreshes table on success.
- All modals surface Zod validation errors inline and display API errors via `vue-sonner` toast.

---

## Component Tree / File Structure

```
app/
└── pages/
    └── user/
        └── index.vue                          # Admin user CRUD page (layout: default)

app/composables/api/
└── use-user-admin-api.ts                      # Wraps all 7 endpoints

app/components/user/
├── user-table.vue                             # Paginated table driven by /api/admin/user/table
├── user-add-modal.vue                         # Create user
├── user-detail-modal.vue                      # View user details
├── user-edit-modal.vue                        # Update user
├── user-password-modal.vue                    # Change password
└── user-delete-modal.vue                      # Delete confirmation

server/api/admin/user/
├── all.get.ts                                 # GET  /api/admin/user/all
├── table.post.ts                              # POST /api/admin/user/table
├── create.post.ts                             # POST /api/admin/user/create
├── [id].get.ts                                # GET  /api/admin/user/[id]
├── [id].put.ts                                # PUT  /api/admin/user/[id]
├── [id].delete.ts                             # DELETE /api/admin/user/[id]
└── update-password/
    └── [id].post.ts                           # POST /api/admin/user/update-password/[id]

server/utils/
├── password.ts                                # hashPassword, verifyPassword (bcrypt)
└── userDefaults.ts                            # initializeUserDefaults()

server/transformers/
└── user.transformer.ts                        # transformUser()

shared/schemas/
└── user.schema.ts                             # createUserSchema, updateUserSchema, updatePasswordSchema

shared/enums/
└── UserRoleEnum.ts                            # { NormalUser: 0, AdminUser: 1 }
```

---

## Data Model

### Database Table: `users`

| Column         | Type                        | Notes                                      |
|----------------|-----------------------------|--------------------------------------------|
| `id`           | `serial` (PK)               |                                            |
| `name`         | `varchar(50)`               | Display name                               |
| `username`     | `varchar(50)` (unique)      | Login credential                           |
| `password`     | `text`                      | bcrypt hash — never returned by API        |
| `avatar`       | `text \| null`              |                                            |
| `telegram`     | `jsonb \| null`             | `{ id, chat_id?, username? }`              |
| `notification` | `jsonb \| null`             | 12 boolean preference keys                 |
| `role`         | `smallint`                  | `UserRoleEnum` (0 = Normal, 1 = Admin)     |
| `is_disable`   | `boolean`                   | Soft-disable flag                          |

### API Endpoints Consumed

| Method   | Endpoint                                  | Request                  | Response                        |
|----------|-------------------------------------------|--------------------------|---------------------------------|
| `GET`    | `/api/admin/user/all`                     | —                        | `UserResponse[]`                |
| `POST`   | `/api/admin/user/table`                   | SearchTable params       | `PaginatedResult<UserResponse>` |
| `GET`    | `/api/admin/user/[id]`                    | —                        | `UserResponse`                  |
| `POST`   | `/api/admin/user/create`                  | `CreateUserRequest`      | `UserResponse` (201)            |
| `PUT`    | `/api/admin/user/[id]`                    | `UpdateUserRequest`      | `UserResponse`                  |
| `POST`   | `/api/admin/user/update-password/[id]`    | `UpdatePasswordRequest`  | `{ message: string }`           |
| `DELETE` | `/api/admin/user/[id]`                    | —                        | `{ message: string }`           |

### TypeScript Interfaces

```typescript
// UserRoleEnum — shared/enums/UserRoleEnum.ts
export const UserRoleEnum = { NormalUser: 0, AdminUser: 1 } as const
export type UserRole = typeof UserRoleEnum[keyof typeof UserRoleEnum]

// Database row shape — never exposed directly to client
interface UserModel {
  id: number
  name: string
  username: string
  password: string        // bcrypt hash
  avatar: string | null
  telegram: { id: string; chat_id?: string; username?: string } | null
  notification: Record<string, boolean> | null
  role: UserRole
  is_disable: boolean
}

// Transformer output — returned by all endpoints
interface UserResponse {
  id: number
  name: string
  username: string
  avatar: string | null
  role: UserRole
  telegram: { id: string; chat_id?: string; username?: string } | null
  is_disable: boolean
}

// POST /api/admin/user/create
interface CreateUserRequest {
  name: string
  username: string
  password: string
  role?: UserRole
  is_disable?: boolean
}

// PUT /api/admin/user/[id]
interface UpdateUserRequest {
  name?: string
  username?: string
  role?: UserRole
  is_disable?: boolean
}

// POST /api/admin/user/update-password/[id]
interface UpdatePasswordRequest {
  current_password: string
  new_password: string
  confirm_password: string  // must equal new_password
}
```

---

## State Management

- A Pinia store (`app/stores/user.ts`) or scoped composable holds the current user list state.
- The paginated table state (page, pageSize, sort, filters) is managed locally within `user-table.vue` or via `useAsyncData` with reactive keys.
- After any mutation (create / update / delete / password change), refresh the table data via the composable.
- No global user list cache required — data is fetched fresh on each table load.

---

## Edge Cases

| Scenario | Expected Behaviour |
|---|---|
| `username` already exists on create | Return `409` with message indicating uniqueness conflict |
| `username` already exists on update (different user) | Return `409` with message indicating uniqueness conflict |
| Update user `id` not found | Return `404` |
| Delete user `id` not found | Return `404` |
| `current_password` mismatch on password change | Return `400`/`401` with descriptive error |
| `new_password` ≠ `confirm_password` | Return `422` with error on `confirm_password` field |
| User being promoted to `AdminUser` already has a private board | Skip board creation — do not duplicate |
| User being promoted to `AdminUser` has no private board | Auto-create one private board |
| Non-admin calls any `/api/admin/user/*` route | `admin` middleware returns `403` |
| Disabled user calls any `/api/admin/user/*` route | `active-user` middleware returns `403` |
| Unauthenticated request | `auth` middleware returns `401` |
| Admin attempts to delete themselves | Not explicitly blocked by spec; implementation may add guard |

---

## Non-Goals

- Self-service profile editing by normal users (separate feature).
- Avatar upload / management (separate feature).
- Telegram chat linking via admin panel (managed by user themselves).
- Bulk user import or export.
- Password reset via email / magic link.
- Session invalidation on role change or disable (handled separately if needed).
- Audit trail for admin actions (see `specs/audit-trail.md`).

---

## Dependencies

| Dependency | Role |
|---|---|
| `specs/authentication.md` | JWT `auth` middleware and token validation |
| `specs/database-schema.md` | `users` table definition and Drizzle schema |
| `specs/validation-schemas.md` | `createUserSchema`, `updateUserSchema`, `updatePasswordSchema` |
| `specs/search-table-service.md` | SearchTable service powering `/api/admin/user/table` |
| `server/utils/password.ts` | `hashPassword`, `verifyPassword` (bcrypt, `SALT_ROUNDS = 10`) |
| `server/utils/userDefaults.ts` | `initializeUserDefaults()` called after user insert |
| `server/transformers/user.transformer.ts` | `transformUser()` — shapes all user API responses |
| `server/middleware/auth.ts` | JWT authentication gate |
| `server/middleware/active-user.ts` | Blocks disabled users |
| `server/middleware/admin.ts` | Restricts to `UserRoleEnum.AdminUser` |
| `shared/enums/UserRoleEnum.ts` | `{ NormalUser: 0, AdminUser: 1 }` |
| BullMQ / notification system | Notification preferences initialised on user create (see `specs/notification-system.md`) |
