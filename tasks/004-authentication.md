# Tasks: Authentication
**Spec:** `specs/authentication.md`
**Status:** pending

---

## Overview

Full-stack JWT authentication covering server-side utilities, API routes, middleware layers, and the client-side auth state machine. Execution must follow the wave order: JWT utility first (other server tasks depend on it), then routes and middleware (can run in parallel), then the client layer, then tests.

---

## Wave 1 — Data Layer (API Agent)

### Task 004-A: JWT Utility & Shared Auth Types
- **Agent:** API
- **Spec:** `specs/authentication.md`
- **Section:** TypeScript Interfaces → `server/utils/jwt.ts` signatures; Redis Keys; AC-3
- **Acceptance Criteria:**
  - [ ] AC-3: `isBlacklisted(jti)` is called inside `verifyToken()` before accepting any token.
  - [ ] AC-3: A blacklisted token returns `401 Unauthorized`.
  - [ ] AC-3: Redis key `jwt:blacklist:{jti}` is written with TTL equal to the token's remaining expiry seconds.
  - [ ] `signToken(userId)` signs with HS256 using `runtimeConfig.jwtSecret`, embeds `sub` (userId) and `jti` (uuid), respects `runtimeConfig.jwtTtl`.
  - [ ] `verifyToken(token)` calls `isBlacklisted(jti)` before returning the payload.
  - [ ] `refreshToken(oldToken)` verifies the old token, issues a new one, and blacklists the old `jti`.
  - [ ] `blacklistToken(jti, ttlSeconds)` writes to Redis; if Redis is unavailable it logs the error and throws so callers can return 500.
  - [ ] `shared/types/auth.types.ts` exports `LoginResponse`, `RefreshResponse`, `JWTPayload`, `CentrifugoChannelResponse` exactly as specified.
- **Dependencies:** `tasks/001-infrastructure-setup.md` (Redis client, `runtimeConfig.jwtSecret` / `jwtTtl` / `jwtRefreshTtl`), `tasks/002-database-schema.md`
- **Output Files:**
  - `server/utils/jwt.ts`
  - `shared/types/auth.types.ts`

---

### Task 004-B: Auth API Routes
- **Agent:** API
- **Spec:** `specs/authentication.md`
- **Section:** AC-1, AC-2, AC-4, AC-11; API Endpoints Consumed; Component Tree
- **Acceptance Criteria:**
  - [ ] AC-1: `POST /api/auth/login` validates body with `loginSchema`; returns `{ token, user: UserModel }` on success; returns `422` on schema failure; returns `401` on bad credentials; uses `server/utils/password.ts` for hash comparison.
  - [ ] AC-2: `POST /api/auth/logout` requires `Authorization: Bearer {token}`; extracts `jti`; calls `blacklistToken(jti, remainingTtl)`; returns `200` on success; returns `500` if Redis blacklisting fails.
  - [ ] AC-4: `POST /api/auth/refresh` accepts a valid non-expired non-blacklisted token; calls `refreshToken()`; returns `{ token }` with the new token; returns `401` on expired, blacklisted, or invalid input.
  - [ ] AC-11: `POST /api/auth/centrifugo/channel` validates `{ channel }` with `centrifugoChannelSchema`; parses channel format `board#{boardId}` or `card#{boardId}`; checks `boardUsers` membership or `role === UserRoleEnum.AdminUser`; returns Centrifugo subscription token on success; returns `403` for non-member non-admins; returns `422` for invalid channel format.
- **Dependencies:** Task 004-A, `tasks/002-database-schema.md` (users table, boardUsers table), `tasks/003-validation-schemas.md` (loginSchema, centrifugoChannelSchema)
- **Output Files:**
  - `server/api/auth/login.post.ts`
  - `server/api/auth/logout.post.ts`
  - `server/api/auth/refresh.post.ts`
  - `server/api/auth/centrifugo/channel.post.ts`

---

### Task 004-C: Server Middleware
- **Agent:** API
- **Spec:** `specs/authentication.md`
- **Section:** AC-5, AC-6, AC-7; Edge Cases (disabled user, non-admin, concurrent logout)
- **Acceptance Criteria:**
  - [ ] AC-5: `server/middleware/auth.ts` intercepts all `/api/**` requests except `/api/auth/login`; missing or invalid token returns `401`; valid token attaches `event.context.user` and `event.context.token`; non-API routes are not affected.
  - [ ] AC-5: `event.context.user` is populated with the full `UserModel` from the database (not just the JWT payload).
  - [ ] AC-6: `server/middleware/active-user.ts` runs after auth middleware on all `/api/**` except `/api/auth/**`; if `event.context.user.isDisable === true` returns `403 Forbidden`; active users pass through.
  - [ ] AC-7: `server/middleware/admin.ts` runs on all `/api/admin/**` routes; if `event.context.user.role !== UserRoleEnum.AdminUser` returns `401 Unauthorized`; admin users pass through.
  - [ ] Edge case: a blacklisted token (e.g., from concurrent logout) is caught by `verifyToken()` inside auth middleware and returns `401`.
- **Dependencies:** Task 004-A, `tasks/002-database-schema.md` (users table for user lookup), `tasks/001-infrastructure-setup.md` (UserRoleEnum path)
- **Output Files:**
  - `server/middleware/auth.ts`
  - `server/middleware/active-user.ts`
  - `server/middleware/admin.ts`

---

## Wave 2 — Frontend (Frontend Agent)

### Task 004-D: Client Auth Layer
- **Agent:** Frontend
- **Spec:** `specs/authentication.md`
- **Section:** AC-8, AC-9, AC-10; State Management; Auth Plugin; Route Middleware; Component Tree (`app/` subtree)
- **Acceptance Criteria:**
  - [ ] AC-8: `app/plugins/0.auth.client.ts` runs client-side only on every app load before routing; reads token via `useAuth().getAuthToken()`; calls `authStore.refreshToken()` then `authStore.fetchUser()` in sequence; on any error calls `authStore.logout()` and redirects to `/auth/log-in`.
  - [ ] AC-8: `authStore.is_authenticated` is `true` only when both `token` and `user` are non-null.
  - [ ] AC-9: `app/middleware/auth.ts` is a global route middleware; checks `authStore.isLoggedIn`; redirects unauthenticated users to `/auth/log-in`; redirects already-authenticated users away from `/auth/log-in`.
  - [ ] AC-10: All `$fetch` / `useFetch` calls include `Authorization: Bearer {token}` automatically (configured at the composable or plugin level); a `401` response from any API call triggers `authStore.logout()` and redirects to `/auth/log-in`.
  - [ ] AC-1 (client side): `authStore.login(credentials)` calls `POST /api/auth/login`, stores the returned token via `useAuth().login(token)` (localStorage), sets `user` and `token` in store.
  - [ ] AC-2 (client side): `authStore.logout()` calls `POST /api/auth/logout`, clears localStorage via `useAuth().logout()`, resets `user`, `token`, `is_authenticated` to initial values, redirects to `/auth/log-in`.
  - [ ] AC-4 (client side): `authStore.refreshToken()` calls `POST /api/auth/refresh`, replaces token in localStorage and in store.
  - [ ] `useAuth()` composable exports `login(token)`, `logout()`, `getAuthToken()`, `isAuthenticated()` as pure localStorage wrappers with no side effects.
  - [ ] `authStore.isLoggedIn` computed returns `true` only when `token !== null && user !== null`.
  - [ ] Edge case: concurrent refresh calls are protected by a lock/queue to prevent duplicate `POST /api/auth/refresh` calls.
- **Dependencies:** Task 004-A, Task 004-B
- **Output Files:**
  - `app/stores/auth.ts`
  - `app/composables/useAuth.ts`
  - `app/middleware/auth.ts`
  - `app/plugins/0.auth.client.ts`

---

## Wave 3 — Tests (Testing Agent)

### Task 004-E: Auth Tests
- **Agent:** Testing
- **Spec:** `specs/authentication.md`
- **Section:** All ACs; Edge Cases table
- **Acceptance Criteria:**
  - [ ] Unit tests for `server/utils/jwt.ts`: happy paths for `signToken`, `verifyToken`, `refreshToken`, `blacklistToken`, `isBlacklisted`; error cases: expired token, blacklisted token, invalid signature, Redis unavailable on blacklist.
  - [ ] Unit tests for `server/middleware/auth.ts`: valid token passes through and sets `event.context`; missing token → 401; invalid token → 401; blacklisted token → 401; non-API route skipped.
  - [ ] Unit tests for `server/middleware/active-user.ts`: active user passes; disabled user → 403; `/api/auth/**` routes are skipped.
  - [ ] Unit tests for `server/middleware/admin.ts`: admin user passes; non-admin → 401.
  - [ ] Unit tests for `server/api/auth/login.post.ts`: valid credentials → 200 with token + user; invalid password → 401; schema failure → 422.
  - [ ] Unit tests for `server/api/auth/logout.post.ts`: valid token → 200 + blacklisted jti; Redis failure → 500.
  - [ ] Unit tests for `server/api/auth/refresh.post.ts`: valid token → 200 with new token + old jti blacklisted; expired token → 401; blacklisted token → 401.
  - [ ] Unit tests for `server/api/auth/centrifugo/channel.post.ts`: board member → 200 with subscription token; admin → 200; non-member → 403; invalid channel format → 422.
  - [ ] E2e test `tests/e2e/auth.spec.ts`: login happy path (valid credentials → redirected to app); login error (wrong password → error shown); logout (token cleared → redirected to login); protected route redirect (unauthenticated → `/auth/log-in`); authenticated user visiting `/auth/log-in` → redirected away.
- **Dependencies:** Task 004-A, Task 004-B, Task 004-C, Task 004-D
- **Output Files:**
  - `tests/unit/server/utils/jwt.test.ts`
  - `tests/unit/server/middleware/auth.test.ts`
  - `tests/unit/server/middleware/active-user.test.ts`
  - `tests/unit/server/middleware/admin.test.ts`
  - `tests/unit/server/api/auth/login.test.ts`
  - `tests/unit/server/api/auth/logout.test.ts`
  - `tests/unit/server/api/auth/refresh.test.ts`
  - `tests/unit/server/api/auth/centrifugo-channel.test.ts`
  - `tests/e2e/auth.spec.ts`
