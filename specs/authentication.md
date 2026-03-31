# Spec: Authentication

## Status: Approved

---

## Purpose

Implement full-stack JWT-based authentication for the Nuxt 4 task management application. This covers user login/logout/refresh endpoints, token blacklisting in Redis, three server-side middleware layers (auth, active-user, admin), client-side auth state management via Pinia, localStorage persistence, automatic token restoration on app load, route protection, and Centrifugo WebSocket channel authorization.

---

## User Stories

- **US-1:** As a user, I can log in with my username and password, receive a JWT token, and be redirected to the application.
- **US-2:** As a user, I can log out and have my token invalidated so it cannot be reused.
- **US-3:** As a user, my session is automatically restored when I return to the application (token persisted in localStorage).
- **US-4:** As a user, my token is silently refreshed on app load so I am not interrupted mid-session.
- **US-5:** As an unauthenticated user, I am redirected to `/auth/log-in` when I try to access any protected page.
- **US-6:** As a disabled user, I receive a 403 response on any API request, preventing access to the application.
- **US-7:** As a non-admin user, I receive a 401 response if I attempt to access any `/api/admin/**` route.
- **US-8:** As a board member or admin, I can authorize a Centrifugo WebSocket channel subscription for a board or card.
- **US-9:** As a user who is not a member of a board, I am denied Centrifugo channel access with a 403.

---

## Acceptance Criteria

### AC-1: Login
- `POST /api/auth/login` accepts `{ username, password }` validated by `loginSchema`.
- Returns `{ token: string, user: UserModel }` on success.
- Returns `422` with field errors if validation fails.
- Returns `401` if credentials are invalid.
- Token is stored in `authStore` (Pinia) and localStorage on success.
- User is redirected away from `/auth/log-in` after successful login.

### AC-2: Logout
- `POST /api/auth/logout` requires a valid `Authorization: Bearer {token}` header.
- Extracts the `jti` from the token and adds it to a Redis blacklist with a TTL matching the token's remaining expiry.
- Returns `200` on success.
- `authStore` clears `user`, `token`, and `is_authenticated` on logout.
- localStorage entry is cleared on logout.
- User is redirected to `/auth/log-in` after logout.

### AC-3: Token Blacklisting
- `isBlacklisted(jti)` is checked during `verifyToken()` before any token is accepted.
- A blacklisted token returns `401 Unauthorized` on any subsequent request.
- Blacklist entries in Redis expire automatically when the token would have expired.

### AC-4: Token Refresh
- `POST /api/auth/refresh` accepts a valid (non-expired, non-blacklisted) token.
- Issues a new token with a fresh expiry (within the 2-week refresh TTL window).
- The old token is blacklisted on successful refresh.
- Returns `401` if the token is expired, blacklisted, or invalid.
- On app load, `0.auth.client.ts` calls `refreshToken()` before routing begins; failure triggers logout.

### AC-5: Server Auth Middleware
- All requests to `/api/**` (except `/api/auth/login`) require a valid `Authorization: Bearer {token}` header.
- Missing or invalid token returns `401 Unauthorized`.
- Valid token attaches `event.context.user` and `event.context.token` for downstream handlers.
- Non-API routes (pages, assets) are not affected by the auth middleware.

### AC-6: Active-User Middleware
- Applied to all authenticated API routes except `/api/auth/**`.
- If `event.context.user.isDisable === true`, returns `403 Forbidden`.
- Active users pass through without interruption.

### AC-7: Admin Middleware
- Applied to all `/api/admin/**` routes.
- If `event.context.user.role !== UserRoleEnum.AdminUser`, returns `401 Unauthorized`.
- Admin users pass through without interruption.

### AC-8: Client Auth State Restoration
- On app load, `0.auth.client.ts` reads the token from localStorage via `useAuth().getAuthToken()`.
- If a token exists, calls `authStore.refreshToken()` then `authStore.fetchUser()`.
- If either call fails (e.g., token expired/blacklisted), `authStore.logout()` is called and the user is redirected to `/auth/log-in`.
- `authStore.is_authenticated` is `true` only when both `token` and `user` are populated.

### AC-9: Route Middleware
- `app/middleware/auth.ts` is applied globally to all pages except `/auth/log-in`.
- Checks `authStore.isLoggedIn`; redirects to `/auth/log-in` if `false`.
- Authenticated users visiting `/auth/log-in` are redirected to the application home.

### AC-10: API Request Authorization Header
- All requests made via `useApi` / `$fetch` automatically include `Authorization: Bearer {token}`.
- A `401` response from any API call triggers `authStore.logout()` and redirects to `/auth/log-in`.

### AC-11: Centrifugo Channel Authorization
- `POST /api/auth/centrifugo/channel` accepts `{ channel: string }` validated by `centrifugoChannelSchema`.
- Channel format is `board#{boardId}` or `card#{boardId}` (numeric IDs).
- Checks that the authenticated user is a member of the board (`boardUsers` table) or has `role === UserRoleEnum.AdminUser`.
- Returns a valid Centrifugo subscription token on success.
- Returns `403 Forbidden` if the user is not a board member and is not an admin.
- Returns `422` if channel format is invalid.

---

## Component Tree / File Structure

```
server/
├── api/
│   └── auth/
│       ├── login.post.ts            # POST /api/auth/login
│       ├── logout.post.ts           # POST /api/auth/logout
│       ├── refresh.post.ts          # POST /api/auth/refresh
│       └── centrifugo/
│           └── channel.post.ts      # POST /api/auth/centrifugo/channel
├── middleware/
│   ├── auth.ts                      # JWT validation for all /api/** routes
│   ├── active-user.ts               # 403 gate for disabled users
│   └── admin.ts                     # 401 gate for non-admins on /api/admin/**
└── utils/
    └── jwt.ts                       # signToken, verifyToken, refreshToken, blacklistToken, isBlacklisted

app/
├── composables/
│   └── useAuth.ts                   # localStorage wrapper (login, logout, getAuthToken, isAuthenticated)
├── middleware/
│   └── auth.ts                      # Client route guard — redirects unauthenticated users
├── stores/
│   └── auth.ts                      # Pinia authStore (user, token, is_authenticated, login, logout, refreshToken, fetchUser)
└── plugins/
    └── 0.auth.client.ts             # App-load token restoration and refresh

shared/
└── schemas/
    └── auth.schema.ts               # loginSchema, centrifugoChannelSchema (Zod)
```

---

## Data Model

### Database Tables Referenced

| Table | Columns Used | Purpose |
|-------|-------------|---------|
| `users` | `id`, `username`, `password`, `role`, `isDisable` | Credential validation, user context, role checks |
| `boardUsers` | `userId`, `boardId` | Board membership check for Centrifugo channel auth |

_No new tables are added by this feature. Token blacklisting is stored entirely in Redis._

### Redis Keys

| Key Pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `jwt:blacklist:{jti}` | String/Set | Remaining token TTL | Blacklisted token JTI entries |

---

## API Endpoints Consumed

| Method | Endpoint | Request Body | Response |
|--------|----------|-------------|----------|
| `POST` | `/api/auth/login` | `{ username: string, password: string }` | `{ token: string, user: UserModel }` |
| `POST` | `/api/auth/logout` | _(none — uses Authorization header)_ | `{ message: string }` |
| `POST` | `/api/auth/refresh` | _(none — uses Authorization header)_ | `{ token: string }` |
| `POST` | `/api/auth/centrifugo/channel` | `{ channel: string }` | `{ token: string }` (Centrifugo subscription token) |

---

## TypeScript Interfaces

```typescript
// shared/types/auth.types.ts

interface LoginResponse {
  token: string
  user: UserModel
}

interface RefreshResponse {
  token: string
}

interface JWTPayload {
  sub: number    // userId
  jti: string    // unique token ID used for blacklisting
  iat: number    // issued-at (Unix timestamp)
  exp: number    // expiry (Unix timestamp)
}

interface CentrifugoChannelResponse {
  token: string  // Centrifugo subscription JWT
}
```

```typescript
// server/utils/jwt.ts signatures

async function signToken(userId: number): Promise<string>
async function verifyToken(token: string): Promise<JWTPayload>
async function refreshToken(oldToken: string): Promise<string>
async function blacklistToken(jti: string, ttlSeconds: number): Promise<void>
async function isBlacklisted(jti: string): Promise<boolean>
```

```typescript
// app/stores/auth.ts (Setup Store)

interface AuthState {
  user: UserModel | null
  token: string | null
  is_authenticated: boolean
}

// Computed
const isLoggedIn: ComputedRef<boolean>  // true when token && user are set

// Actions
async function login(credentials: { username: string; password: string }): Promise<void>
async function logout(): Promise<void>
async function refreshToken(): Promise<void>
async function fetchUser(): Promise<void>
```

```typescript
// app/composables/useAuth.ts

function login(token: string): void           // persist token to localStorage
function logout(): void                        // remove token from localStorage
function getAuthToken(): string | null         // read token from localStorage
function isAuthenticated(): boolean            // check if token exists in localStorage
```

---

## State Management

### `authStore` (Pinia Setup Store — `app/stores/auth.ts`)

| State | Type | Initial Value | Description |
|-------|------|---------------|-------------|
| `user` | `UserModel \| null` | `null` | Authenticated user object |
| `token` | `string \| null` | `null` | Current JWT access token |
| `is_authenticated` | `boolean` | `false` | Auth state flag |

| Computed | Type | Description |
|----------|------|-------------|
| `isLoggedIn` | `boolean` | `true` when `token !== null && user !== null` |

| Action | Description |
|--------|-------------|
| `login(credentials)` | Calls `POST /api/auth/login`, stores token in localStorage + Pinia, populates `user` |
| `logout()` | Calls `POST /api/auth/logout`, clears localStorage, resets store state, redirects to `/auth/log-in` |
| `refreshToken()` | Calls `POST /api/auth/refresh`, replaces stored token in localStorage + Pinia |
| `fetchUser()` | Fetches current user data and populates `user` in store |

### Auth Plugin (`app/plugins/0.auth.client.ts`)

- Runs client-side only on every app load (before routing).
- Reads token from localStorage → calls `refreshToken()` → calls `fetchUser()`.
- On any error: calls `logout()`, redirects to `/auth/log-in`.

### Route Middleware (`app/middleware/auth.ts`)

- Runs on every client-side navigation.
- Reads `authStore.isLoggedIn`.
- Redirects to `/auth/log-in` when `false`.
- Redirects authenticated users away from `/auth/log-in`.

---

## Edge Cases

| Scenario | Expected Behavior |
|----------|------------------|
| Token expired before refresh on app load | Plugin catches 401, calls `logout()`, redirects to login |
| Token blacklisted mid-session (e.g., concurrent logout) | Next API request returns 401, triggers auto-logout |
| Disabled user logs in successfully | First authenticated API call returns 403 from `active-user` middleware |
| Non-admin hits `/api/admin/**` | `admin` middleware returns 401 before the route handler runs |
| Concurrent requests during token refresh | All in-flight requests should re-use the new token; a refresh lock/queue should prevent duplicate refresh calls |
| `POST /api/auth/login` with wrong password | Returns 401; no token issued; no store mutation |
| `POST /api/auth/login` with invalid schema | Returns 422 with Zod field errors |
| Centrifugo channel string with invalid format | `centrifugoChannelSchema` validation returns 422 |
| User removed from board while Centrifugo subscription is active | Existing subscription is not revoked server-push; re-authorization on next subscription attempt returns 403 |
| JWT_SECRET rotation | All existing tokens become invalid; all users are effectively logged out |
| Redis unavailable at logout | Blacklisting fails; token remains valid until natural expiry — log the error, return 500 to client |

---

## Non-Goals

- **OAuth / Social Login:** This spec covers only username + password authentication.
- **Multi-factor Authentication (MFA):** Not in scope for this milestone.
- **Token rotation on every request:** Only explicit refresh via `POST /api/auth/refresh` or app-load plugin triggers new token issuance.
- **Session-based auth:** All auth is stateless JWT; server does not maintain session state beyond the Redis blacklist.
- **Password reset via email:** Not covered here; see `client-user-profile.md` for password change (authenticated).
- **Remember Me / extended sessions:** Token TTL is fixed via environment config; no per-login TTL control.
- **Centrifugo server-side push revocation:** Active WebSocket subscriptions are not terminated server-side when board membership changes.

---

## Dependencies

| Dependency | Type | Purpose |
|------------|------|---------|
| `jose` | Server library | JWT signing (`SignJWT`) and verification (`jwtVerify`) with HS256 |
| Redis | Infrastructure | Token blacklist storage (`blacklistToken`, `isBlacklisted`) |
| `bcrypt` / `server/utils/password.ts` | Server utility | Password hash comparison during login |
| `UserRoleEnum` | Shared enum (`shared/enums/`) | Role comparison in `admin` middleware and Centrifugo channel auth |
| `loginSchema` / `centrifugoChannelSchema` | Shared Zod schemas (`shared/schemas/auth.schema.ts`) | Request body validation |
| `UserModel` / `boardUsers` schema | Database (`server/database/schema/`) | User lookup and board membership checks |
| Pinia | Client library | `authStore` state management |
| VueUse | Client library | Composable utilities used in store/plugins if needed |
| `database-schema.md` | Spec dependency | `users` table structure, `boardUsers` junction table |
| `infrastructure-setup.md` | Spec dependency | Redis connection, `runtimeConfig` for `JWT_SECRET`, `JWT_TTL`, `JWT_REFRESH_TTL` |
| `realtime-centrifugo.md` | Spec dependency | Centrifugo subscription token format expected by `channel.post.ts` |
