# Tasks: Infrastructure Setup
**Spec:** `specs/infrastructure-setup.md`
**Status:** pending

## Overview
Bootstrap the complete project from an empty repository to a fully connected, running system. This covers Docker Compose service definitions (PostgreSQL 16, Redis 7, Centrifugo v5), all environment variable declarations and runtime config mapping in `nuxt.config.ts`, Drizzle ORM client and migration toolchain, a database seeder, a file storage server utility with upload/delete/serve support, TypeScript strict-mode toolchain, full package dependency declarations (`package.json`), and the complete project directory scaffold. No application-level API endpoints or database table schemas are defined here — those belong to subsequent task files.

---

## Wave 1 — Infrastructure & Configuration (API Agent)

### Task 001-A: Docker Compose & Centrifugo Configuration
- **Agent:** API
- **Spec:** `specs/infrastructure-setup.md`
- **Section:** Docker & Services (AC-1 to AC-7)
- **Acceptance Criteria:**
  - [ ] AC-1: `compose.dev.yaml` defines services: `app`, `db` (postgres:16-alpine), `redis` (redis:7-alpine), and `centrifugo` (centrifugo/centrifugo:v5).
  - [ ] AC-2: The `app` service depends on `db`, `redis`, and `centrifugo` (`depends_on` list).
  - [ ] AC-3: The `db` service persists data to a named volume `pgdata` mounted at `/var/lib/postgresql/data`.
  - [ ] AC-4: The `centrifugo` service mounts `./centrifugo.json` into the container and starts with `--config=/centrifugo/config.json`.
  - [ ] AC-5: Running `docker compose -f compose.dev.yaml up` successfully starts all four services with no errors.
  - [ ] AC-6: A `compose.prod.yaml` exists alongside `compose.dev.yaml` for production deployment.
  - [ ] AC-7: Docker configuration files are organized under `docker/development/` and `docker/production/`.
- **Dependencies:** none
- **Output Files:**
  - `compose.dev.yaml`
  - `compose.prod.yaml`
  - `centrifugo.json`
  - `docker/development/Dockerfile`
  - `docker/production/Dockerfile`

---

### Task 001-B: Environment Variables & `.env.example`
- **Agent:** API
- **Spec:** `specs/infrastructure-setup.md`
- **Section:** Environment Variables (AC-8 to AC-18)
- **Acceptance Criteria:**
  - [ ] AC-8: A `.env.example` file is present listing every required environment variable with placeholder values.
  - [ ] AC-9: `DATABASE_URL` is set in the form `postgresql://<user>:<password>@<host>:<port>/<dbname>`.
  - [ ] AC-10: `REDIS_URL` is set in the form `redis://<host>:<port>`.
  - [ ] AC-11: `JWT_SECRET` is at least 32 characters long; the application refuses to start if it is shorter.
  - [ ] AC-12: `JWT_TTL` (access token TTL in nanoseconds) and `JWT_REFRESH_TTL` (refresh TTL in minutes) are both present and parsed as numbers.
  - [ ] AC-13: `CENTRIFUGO_API_URL`, `CENTRIFUGO_API_KEY`, and `CENTRIFUGO_SECRET` are present and used by server-side Centrifugo utilities.
  - [ ] AC-14: `WS_BASE_URL` is exposed as a public runtime config value (`runtimeConfig.public.wsBaseUrl`).
  - [ ] AC-15: `TELEGRAM_BOT_TOKEN` is present and injected only into server-side runtime config.
  - [ ] AC-16: `STORAGE_PATH` defaults to `./storage` if not set.
  - [ ] AC-17: `MAX_FILE_SIZE` defaults to `10485760` (10 MB) if not set and is parsed as an integer.
  - [ ] AC-18: `SOCKET_DEBUG` is exposed as `runtimeConfig.public.socketDebug` (boolean, defaults to `false`).
- **Dependencies:** none
- **Output Files:**
  - `.env.example`

---

### Task 001-C: Nuxt Configuration (`nuxt.config.ts`)
- **Agent:** API
- **Spec:** `specs/infrastructure-setup.md`
- **Section:** Nuxt Configuration (AC-19 to AC-26)
- **Acceptance Criteria:**
  - [ ] AC-19: `compatibilityDate` is set to `'2024-11-01'` and `future.compatibilityVersion` is `4`.
  - [ ] AC-20: `ssr` is set to `false` (SPA mode — SSR is disabled).
  - [ ] AC-21: The following modules are registered: `@pinia/nuxt`, `@nuxt/image`, `@nuxt/icon`, `nuxt-easy-lightbox`, `@tailwindcss/vite`.
  - [ ] AC-22: All server-only secrets (`databaseUrl`, `redisUrl`, `jwtSecret`, `jwtTtl`, `jwtRefreshTtl`, `centrifugoApiUrl`, `centrifugoApiKey`, `centrifugoSecret`, `telegramBotToken`, `storagePath`, `maxFileSize`) are under `runtimeConfig` (not `runtimeConfig.public`).
  - [ ] AC-23: Public runtime config (`runtimeConfig.public`) contains only `wsBaseUrl` and `socketDebug`.
  - [ ] AC-24: `routeRules` applies CORS headers (`Access-Control-Allow-Origin: *`, `Access-Control-Allow-Methods: *`, `Access-Control-Allow-Headers: *`) to all `/api/**` routes.
  - [ ] AC-25: `nitro.preset` is set to `'node-server'`.
  - [ ] AC-26: Tailwind CSS is wired via the Vite plugin (`tailwindcss()` in `vite.plugins`).
- **Dependencies:** Task 001-B (all env var names must be known before mapping to `runtimeConfig`)
- **Output Files:**
  - `nuxt.config.ts`

---

### Task 001-D: TypeScript Config & Package Dependencies
- **Agent:** API
- **Spec:** `specs/infrastructure-setup.md`
- **Section:** TypeScript & Toolchain (AC-43 to AC-48)
- **Acceptance Criteria:**
  - [ ] AC-43: `tsconfig.json` enables strict mode (`strict: true`).
  - [ ] AC-44: `package.json` includes all server runtime dependencies: `drizzle-orm`, `pg`, `jose`, `bcrypt`, `bullmq`, `ioredis`, `zod`, `formidable`.
  - [ ] AC-45: `package.json` includes all server dev dependencies: `drizzle-kit`, `@types/pg`, `@types/bcrypt`.
  - [ ] AC-46: `package.json` includes all client runtime dependencies: `pinia`, `centrifuge`, `gsap`, `vue-sonner`, `vue-draggable-plus`, `vue-iconsax`, `vue3-persian-datetime-picker`, `hammerjs`.
  - [ ] AC-47: The project builds without TypeScript errors (`nuxi build` exits 0).
  - [ ] AC-48: ESLint and Prettier pass with zero errors across all project files.
- **Dependencies:** none
- **Output Files:**
  - `package.json`
  - `tsconfig.json`
  - `tailwind.config.js`
  - `eslint.config.mjs`
  - `.prettierrc`

---

### Task 001-E: Drizzle ORM Configuration & Database Client
- **Agent:** API
- **Spec:** `specs/infrastructure-setup.md`
- **Section:** Drizzle ORM (AC-27 to AC-34)
- **Acceptance Criteria:**
  - [ ] AC-27: `drizzle.config.ts` sets `schema` to `'./server/database/schema/index.ts'`, `out` to `'./server/database/migrations'`, and `dialect` to `'postgresql'`.
  - [ ] AC-28: `drizzle.config.ts` reads `DATABASE_URL` from `process.env` for `dbCredentials.url`.
  - [ ] AC-29: `server/database/index.ts` exports a `db` instance created with `drizzle(process.env.DATABASE_URL!, { schema })` using `drizzle-orm/node-postgres`.
  - [ ] AC-30: All Drizzle schema files are imported and re-exported from `server/database/schema/index.ts`.
  - [ ] AC-31: `npx drizzle-kit generate` produces migration files in `server/database/migrations/` without error.
  - [ ] AC-32: `npx drizzle-kit migrate` applies all pending migrations to a running PostgreSQL 16 instance without error.
  - [ ] AC-33: `npx drizzle-kit push` applies the schema directly to the database (development use only).
  - [ ] AC-34: `npx drizzle-kit studio` opens Drizzle Studio pointing at the configured database.
- **Dependencies:** Task 001-D (requires `drizzle-orm` and `pg` in `package.json`)
- **Cross-References:** `tasks/002-database-schema.md` — individual schema files (e.g., `users.ts`, `boards.ts`) imported by `schema/index.ts` are defined there; `index.ts` must be kept in sync as schemas are added.
- **Output Files:**
  - `drizzle.config.ts`
  - `server/database/index.ts`
  - `server/database/schema/index.ts`

---

### Task 001-F: Database Seeder
- **Agent:** API
- **Spec:** `specs/infrastructure-setup.md`
- **Section:** Database Seeder (AC-35 to AC-37)
- **Acceptance Criteria:**
  - [ ] AC-35: `server/database/seed.ts` exists and is executable via `npx tsx server/database/seed.ts`.
  - [ ] AC-36: The seeder creates at least one admin user, a set of default tags, and sample boards/lists/cards.
  - [ ] AC-37: Running the seeder on an already-seeded database does not duplicate records (idempotent or guarded).
- **Dependencies:** Task 001-E (requires the `db` instance from `server/database/index.ts`); `tasks/002-database-schema.md` (requires `users`, `tags`, `boards`, `boardLists`, and `cards` table schemas to be defined before inserting rows)
- **Output Files:**
  - `server/database/seed.ts`

---

### Task 001-G: File Storage Server Utility & Serve Route
- **Agent:** API
- **Spec:** `specs/infrastructure-setup.md`
- **Section:** File Storage (AC-38 to AC-42)
- **Acceptance Criteria:**
  - [ ] AC-38: Attachment files are stored under `storage/media/`.
  - [ ] AC-39: Avatar files are stored under `storage/avatar/`.
  - [ ] AC-40: Files are served by a dedicated `/storage/**` Nitro route.
  - [ ] AC-41: Upload, replace, and delete operations are supported; delete removes the file from disk.
  - [ ] AC-42: File uploads exceeding `MAX_FILE_SIZE` (default 10 MB) are rejected with an appropriate error response.
- **Dependencies:** Task 001-C (requires `storagePath` and `maxFileSize` from `runtimeConfig`)
- **Notes:**
  - `saveFile(buffer, subdir, filename)` must call `mkdirSync({ recursive: true })` before writing to handle a missing `STORAGE_PATH` directory (edge case from spec).
  - `parseInt(MAX_FILE_SIZE)` must fall back to `10485760` if the result is `NaN` (spec edge case).
- **Output Files:**
  - `server/utils/storage.ts`
  - `server/api/storage/[...path].get.ts`
  - `storage/media/.gitkeep`
  - `storage/avatar/.gitkeep`
  - `.gitignore` (add `storage/**/*`, exempt `.gitkeep` files)

---

### Task 001-H: Project Directory Scaffolding
- **Agent:** API
- **Spec:** `specs/infrastructure-setup.md`
- **Section:** Project Directory Structure (AC-49 to AC-56)
- **Acceptance Criteria:**
  - [ ] AC-49: Root directory contains `nuxt.config.ts`, `package.json`, `tsconfig.json`, `tailwind.config.js`, `drizzle.config.ts`, `compose.dev.yaml`, `compose.prod.yaml`.
  - [ ] AC-50: The `shared/` directory contains `enums/`, `schemas/`, `types/`, and `utils/` subdirectories.
  - [ ] AC-51: The `server/` directory contains `api/`, `database/`, `services/`, `jobs/`, `transformers/`, `utils/`, `middleware/`, and `plugins/` subdirectories.
  - [ ] AC-52: The `server/database/` directory contains `schema/`, `migrations/`, `seed.ts`, and `index.ts`.
  - [ ] AC-53: The `app/` directory exists for all client-side code (components, pages, stores, composables, layouts, middleware, plugins, utils, assets, types).
  - [ ] AC-54: The `public/` directory contains at minimum `robots.txt` and an `images/` subdirectory.
  - [ ] AC-55: The `docker/` directory contains `development/` and `production/` subdirectories.
  - [ ] AC-56: The `specs/` directory exists and contains all feature specification files.
- **Dependencies:** none (can run in parallel with all Wave 1 tasks)
- **Output Files:**
  - `shared/enums/.gitkeep`
  - `shared/schemas/.gitkeep`
  - `shared/types/.gitkeep`
  - `shared/utils/.gitkeep`
  - `server/api/auth/.gitkeep`
  - `server/api/admin/.gitkeep`
  - `server/api/client/.gitkeep`
  - `server/services/.gitkeep`
  - `server/jobs/.gitkeep`
  - `server/transformers/.gitkeep`
  - `server/utils/.gitkeep`
  - `server/middleware/.gitkeep`
  - `server/plugins/.gitkeep`
  - `server/database/migrations/.gitkeep`
  - `app/assets/.gitkeep`
  - `app/components/.gitkeep`
  - `app/composables/api/.gitkeep`
  - `app/layouts/.gitkeep`
  - `app/middleware/.gitkeep`
  - `app/pages/.gitkeep`
  - `app/plugins/.gitkeep`
  - `app/stores/.gitkeep`
  - `app/types/.gitkeep`
  - `app/utils/.gitkeep`
  - `public/robots.txt`
  - `public/images/.gitkeep`
  - `tests/unit/.gitkeep`
  - `tests/e2e/.gitkeep`

---

## Wave 2 — Client App Shell (Frontend Agent)

### Task 001-I: App Entry Files & Tailwind Base Styles
- **Agent:** Frontend
- **Spec:** `specs/infrastructure-setup.md`
- **Section:** Project Directory Structure (AC-53), TypeScript & Toolchain (AC-47, AC-48)
- **Acceptance Criteria:**
  - [ ] AC-53 (partial): `app/app.vue` exists as the root component with a `<NuxtLayout>` / `<NuxtPage>` outlet.
  - [ ] AC-53 (partial): `app/error.vue` exists as the global Nuxt error page.
  - [ ] AC-53 (partial): `app/app.config.ts` exists with a minimal typed configuration block.
  - [ ] AC-47 (partial): All `app/` entry files compile without TypeScript errors.
  - [ ] AC-48 (partial): ESLint and Prettier pass on all `app/` entry files.
- **Dependencies:** Task 001-C (requires `nuxt.config.ts` with modules declared), Task 001-D (requires `tailwind.config.js` and `package.json` in place)
- **Output Files:**
  - `app/app.vue`
  - `app/error.vue`
  - `app/app.config.ts`
  - `app/assets/css/main.css`

---

## Wave 3 — Tests (Testing Agent)

### Task 001-J: Infrastructure Tests
- **Agent:** Testing
- **Spec:** `specs/infrastructure-setup.md`
- **Section:** All ACs
- **Acceptance Criteria:**
  - [ ] AC-11 (env validation): A unit test confirms that passing a `JWT_SECRET` shorter than 32 characters throws or triggers a startup error; a valid 32-character secret does not throw.
  - [ ] AC-17 (max file size): A unit test confirms that a non-integer `MAX_FILE_SIZE` env value falls back to `10485760`.
  - [ ] AC-22, AC-23 (runtimeConfig scoping): A unit test asserts that no server-only key (`databaseUrl`, `redisUrl`, `jwtSecret`, etc.) appears in `runtimeConfig.public`, and that `wsBaseUrl` and `socketDebug` are present in `runtimeConfig.public`.
  - [ ] AC-29 (db instance): An integration test confirms the `db` export from `server/database/index.ts` is a valid Drizzle instance; a `SELECT 1` query completes without error against a live PostgreSQL 16 instance.
  - [ ] AC-37 (seeder idempotency): An integration test runs `seed.ts` twice and asserts that the `users` and `tags` tables contain no duplicate rows (same count both times).
  - [ ] AC-38, AC-39, AC-41 (storage paths & delete): Unit tests confirm `saveFile()` writes to `storage/media/` for attachments and `storage/avatar/` for avatars; `deleteFile()` removes the file from disk.
  - [ ] AC-40 (storage serve route): A unit test confirms the `/storage/**` route resolves a stored file path and returns the correct content-type header.
  - [ ] AC-42 (max file size rejection): A unit test confirms that a payload exceeding `MAX_FILE_SIZE` is rejected with an HTTP 413 or equivalent error response.
  - [ ] AC-47 (build smoke): `nuxi build` exits 0 with no TypeScript errors (verified via CI or manual run).
  - [ ] AC-48 (lint smoke): ESLint passes across all files with zero errors (verified via CI or manual run).
- **Dependencies:** Task 001-A, Task 001-B, Task 001-C, Task 001-D, Task 001-E, Task 001-F, Task 001-G, Task 001-H, Task 001-I
- **Output Files:**
  - `tests/unit/infrastructure/env-validation.test.ts`
  - `tests/unit/infrastructure/runtime-config.test.ts`
  - `tests/unit/infrastructure/drizzle-client.test.ts`
  - `tests/unit/infrastructure/seeder.test.ts`
  - `tests/unit/infrastructure/storage.test.ts`
  - `tests/unit/infrastructure/storage-route.test.ts`
  - `tests/e2e/infrastructure/docker-smoke.spec.ts`
