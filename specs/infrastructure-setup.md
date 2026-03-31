# Spec: Infrastructure Setup

## Status: Approved

## Purpose
This spec defines the complete infrastructure configuration for the Nuxt 4 task management application. It covers every layer a DevOps engineer or developer must configure to bootstrap the project from scratch: runtime dependencies, Docker services, environment variables, the Nuxt/Nitro configuration, Drizzle ORM setup, CORS policy, file storage, and the full project directory structure. It is the single source of truth for bringing the application from an empty repository to a running, fully-connected system.

## User Stories
- As a developer, I want a single Docker Compose command to start all backing services so that I can begin local development without manually installing PostgreSQL, Redis, or Centrifugo.
- As a DevOps engineer, I want all environment variables documented and validated so that misconfigured deployments fail loudly at startup rather than silently at runtime.
- As a developer, I want the Drizzle ORM client and migration toolchain configured so that I can evolve the database schema using TypeScript and version-controlled migration files.
- As a developer, I want the Nuxt 4 runtime config to expose the correct variables to server-only and public scopes so that secrets never leak to the client bundle.
- As a developer, I want the complete project directory scaffolded so that I know exactly where to place every category of file.
- As a developer, I want a database seeder available so that I can populate a fresh database with baseline data for development and testing.
- As a developer, I want file storage paths configured so that uploaded attachments and avatars are persisted to a predictable location on disk.

## Acceptance Criteria

### Docker & Services
- [ ] AC-1: `compose.dev.yaml` defines services: `app`, `db` (postgres:16-alpine), `redis` (redis:7-alpine), and `centrifugo` (centrifugo/centrifugo:v5).
- [ ] AC-2: The `app` service depends on `db`, `redis`, and `centrifugo` (`depends_on` list).
- [ ] AC-3: The `db` service persists data to a named volume `pgdata` mounted at `/var/lib/postgresql/data`.
- [ ] AC-4: The `centrifugo` service mounts `./centrifugo.json` into the container and starts with `--config=/centrifugo/config.json`.
- [ ] AC-5: Running `docker compose -f compose.dev.yaml up` successfully starts all four services with no errors.
- [ ] AC-6: A `compose.prod.yaml` exists alongside `compose.dev.yaml` for production deployment.
- [ ] AC-7: Docker configuration files are organized under `docker/development/` and `docker/production/`.

### Environment Variables
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

### Nuxt Configuration (`nuxt.config.ts`)
- [ ] AC-19: `compatibilityDate` is set to `'2024-11-01'` and `future.compatibilityVersion` is `4`.
- [ ] AC-20: `ssr` is set to `false` (SPA mode — SSR is disabled).
- [ ] AC-21: The following modules are registered: `@pinia/nuxt`, `@nuxt/image`, `@nuxt/icon`, `nuxt-easy-lightbox`, `@tailwindcss/vite`.
- [ ] AC-22: All server-only secrets (`databaseUrl`, `redisUrl`, `jwtSecret`, `jwtTtl`, `jwtRefreshTtl`, `centrifugoApiUrl`, `centrifugoApiKey`, `centrifugoSecret`, `telegramBotToken`, `storagePath`, `maxFileSize`) are under `runtimeConfig` (not `runtimeConfig.public`).
- [ ] AC-23: Public runtime config (`runtimeConfig.public`) contains only `wsBaseUrl` and `socketDebug`.
- [ ] AC-24: `routeRules` applies CORS headers (`Access-Control-Allow-Origin: *`, `Access-Control-Allow-Methods: *`, `Access-Control-Allow-Headers: *`) to all `/api/**` routes.
- [ ] AC-25: `nitro.preset` is set to `'node-server'`.
- [ ] AC-26: Tailwind CSS is wired via the Vite plugin (`tailwindcss()` in `vite.plugins`).

### Drizzle ORM
- [ ] AC-27: `drizzle.config.ts` sets `schema` to `'./server/database/schema/index.ts'`, `out` to `'./server/database/migrations'`, and `dialect` to `'postgresql'`.
- [ ] AC-28: `drizzle.config.ts` reads `DATABASE_URL` from `process.env` for `dbCredentials.url`.
- [ ] AC-29: `server/database/index.ts` exports a `db` instance created with `drizzle(process.env.DATABASE_URL!, { schema })` using `drizzle-orm/node-postgres`.
- [ ] AC-30: All Drizzle schema files are imported and re-exported from `server/database/schema/index.ts`.
- [ ] AC-31: `npx drizzle-kit generate` produces migration files in `server/database/migrations/` without error.
- [ ] AC-32: `npx drizzle-kit migrate` applies all pending migrations to a running PostgreSQL 16 instance without error.
- [ ] AC-33: `npx drizzle-kit push` applies the schema directly to the database (development use only).
- [ ] AC-34: `npx drizzle-kit studio` opens Drizzle Studio pointing at the configured database.

### Database Seeder
- [ ] AC-35: `server/database/seed.ts` exists and is executable via `npx tsx server/database/seed.ts`.
- [ ] AC-36: The seeder creates at least one admin user, a set of default tags, and sample boards/lists/cards.
- [ ] AC-37: Running the seeder on an already-seeded database does not duplicate records (idempotent or guarded).

### File Storage
- [ ] AC-38: Attachment files are stored under `storage/media/`.
- [ ] AC-39: Avatar files are stored under `storage/avatar/`.
- [ ] AC-40: Files are served by Nitro's static asset handling or a dedicated `/storage/**` route.
- [ ] AC-41: Upload, replace, and delete operations are supported; delete removes the file from disk and the corresponding database record.
- [ ] AC-42: File uploads exceeding `MAX_FILE_SIZE` (default 10 MB) are rejected with an appropriate error response.

### TypeScript & Toolchain
- [ ] AC-43: `tsconfig.json` enables strict mode (`strict: true`).
- [ ] AC-44: `package.json` includes all server dependencies: `drizzle-orm`, `pg`, `jose`, `bcrypt`, `bullmq`, `ioredis`, `zod`, `formidable`.
- [ ] AC-45: `package.json` includes all server dev dependencies: `drizzle-kit`, `@types/pg`, `@types/bcrypt`.
- [ ] AC-46: `package.json` includes all client dependencies: `pinia`, `centrifuge`, `gsap`, `vue-sonner`, `vue-draggable-plus`, `vue-iconsax`, `vue3-persian-datetime-picker`, `hammerjs`.
- [ ] AC-47: The project builds without TypeScript errors (`nuxi build` exits 0).
- [ ] AC-48: ESLint and Prettier pass with zero errors across all project files.

### Project Directory Structure
- [ ] AC-49: The root directory contains `nuxt.config.ts`, `package.json`, `tsconfig.json`, `tailwind.config.js`, `drizzle.config.ts`, `compose.dev.yaml`, `compose.prod.yaml`.
- [ ] AC-50: The `shared/` directory contains `enums/`, `schemas/`, `types/`, and `utils/` subdirectories.
- [ ] AC-51: The `server/` directory contains `api/`, `database/`, `services/`, `jobs/`, `transformers/`, `utils/`, and `plugins/` subdirectories.
- [ ] AC-52: The `server/database/` directory contains `schema/`, `migrations/`, `seed.ts`, and `index.ts`.
- [ ] AC-53: The `app/` directory exists for all client-side code (components, pages, stores, composables, layouts, middleware, plugins, utils, assets, types).
- [ ] AC-54: The `public/` directory contains at minimum `robots.txt` and an `images/` subdirectory.
- [ ] AC-55: The `docker/` directory contains `development/` and `production/` subdirectories.
- [ ] AC-56: The `specs/` directory exists and contains all feature specification files.

## File Structure

```
task-management/
├── nuxt.config.ts                    # Nuxt 4 configuration (modules, runtimeConfig, routeRules, nitro)
├── package.json                      # All server + client dependencies
├── tsconfig.json                     # TypeScript strict-mode config
├── tailwind.config.js                # Tailwind CSS design tokens (dark theme, violet/purple palette)
├── drizzle.config.ts                 # Drizzle ORM + drizzle-kit configuration
├── compose.dev.yaml                  # Docker Compose — development (app, db, redis, centrifugo)
├── compose.prod.yaml                 # Docker Compose — production
├── centrifugo.json                   # Centrifugo server configuration (mounted by Docker)
├── .env                              # Local environment variables (git-ignored)
├── .env.example                      # Environment variable template (committed)
├── docker/
│   ├── development/                  # Development Dockerfile and supporting files
│   └── production/                   # Production Dockerfile and supporting files
├── shared/                           # Shared between server & client
│   ├── enums/                        # TypeScript const enums (e.g., BoardStatusEnum.ts)
│   ├── schemas/                      # Zod validation schemas (e.g., board.schema.ts)
│   ├── types/                        # Shared TypeScript interfaces
│   └── utils/                        # Shared utility functions
├── server/                           # Nitro server (API backend)
│   ├── api/                          # File-based API route handlers
│   │   ├── auth/                     # Authentication routes
│   │   ├── admin/                    # Admin-only routes
│   │   └── client/                   # Client-facing routes
│   ├── database/
│   │   ├── schema/                   # Drizzle ORM table schemas + relations
│   │   │   └── index.ts              # Re-exports all schema files
│   │   ├── migrations/               # Generated by drizzle-kit
│   │   ├── seed.ts                   # Database seeder (run via npx tsx)
│   │   └── index.ts                  # Drizzle client instance export (db)
│   ├── services/                     # Business logic services
│   ├── jobs/                         # BullMQ job definitions
│   ├── transformers/                 # Response transformer functions
│   ├── utils/                        # Server utilities (jwt, password, response, broadcast)
│   ├── middleware/                    # Server middleware (auth.ts, admin.ts, active-user.ts)
│   └── plugins/                      # Nitro plugins (BullMQ worker initialization)
├── app/                              # SPA client (Vue 3)
│   ├── assets/                       # Uncompiled assets (CSS, images, fonts)
│   ├── components/                   # Auto-imported Vue components
│   ├── composables/                  # Auto-imported composables
│   │   └── api/                      # Internal API composables
│   ├── layouts/                      # Layout components
│   ├── middleware/                    # Route middleware
│   ├── pages/                        # File-based routing
│   ├── plugins/                      # Nuxt plugins (GSAP, Centrifugo, etc.)
│   ├── stores/                       # Pinia Setup Stores
│   ├── types/                        # Client-side TypeScript types
│   ├── utils/                        # Auto-imported utility functions
│   ├── app.vue                       # Root component
│   ├── app.config.ts                 # App-level runtime config
│   └── error.vue                     # Error page
├── public/
│   ├── robots.txt
│   └── images/
├── storage/                          # Runtime file storage (git-ignored)
│   ├── media/                        # Card attachment files
│   └── avatar/                       # User avatar images
├── specs/                            # Feature specifications (Spec-Driven Development)
└── tests/
    ├── unit/                         # Vitest unit tests
    └── e2e/                          # Playwright e2e tests
```

## Data Model

### Environment Variables

| Variable | Scope | Default | Description |
|---|---|---|---|
| `DATABASE_URL` | Server | — | PostgreSQL connection string (`postgresql://user:pass@host:port/db`) |
| `REDIS_URL` | Server | — | Redis connection string (`redis://host:port`) |
| `JWT_SECRET` | Server | — | HS256 signing key, minimum 32 characters |
| `JWT_TTL` | Server | — | Access token TTL in nanoseconds (e.g., `110000000000`) |
| `JWT_REFRESH_TTL` | Server | — | Refresh token TTL in minutes (e.g., `20160` = 14 days) |
| `CENTRIFUGO_API_URL` | Server | — | Centrifugo HTTP API base URL (e.g., `http://localhost:8000/api`) |
| `CENTRIFUGO_API_KEY` | Server | — | Centrifugo HTTP API key |
| `CENTRIFUGO_SECRET` | Server | — | Centrifugo HMAC secret for channel token signing |
| `TELEGRAM_BOT_TOKEN` | Server | — | Telegram Bot API token |
| `STORAGE_PATH` | Server | `./storage` | Root path for file storage on disk |
| `MAX_FILE_SIZE` | Server | `10485760` | Maximum upload size in bytes (10 MB) |
| `WS_BASE_URL` | Public | `ws://localhost:8000/connection/websocket` | Centrifugo WebSocket endpoint (exposed to client) |
| `SOCKET_DEBUG` | Public | `false` | Enable Centrifugo client debug logging |

### Nuxt Runtime Config Mapping

```typescript
// nuxt.config.ts — runtimeConfig
runtimeConfig: {
  // Server-only (private)
  databaseUrl: process.env.DATABASE_URL,
  redisUrl: process.env.REDIS_URL,
  jwtSecret: process.env.JWT_SECRET,
  jwtTtl: process.env.JWT_TTL,
  jwtRefreshTtl: process.env.JWT_REFRESH_TTL,
  centrifugoApiUrl: process.env.CENTRIFUGO_API_URL,
  centrifugoApiKey: process.env.CENTRIFUGO_API_KEY,
  centrifugoSecret: process.env.CENTRIFUGO_SECRET,
  telegramBotToken: process.env.TELEGRAM_BOT_TOKEN,
  storagePath: process.env.STORAGE_PATH || './storage',
  maxFileSize: parseInt(process.env.MAX_FILE_SIZE || '10485760'),
  public: {
    wsBaseUrl: process.env.WS_BASE_URL || 'ws://localhost:8000/connection/websocket',
    socketDebug: process.env.SOCKET_DEBUG === 'true',
  },
},
```

### Drizzle Configuration

```typescript
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit'

export default defineConfig({
  schema: './server/database/schema/index.ts',
  out: './server/database/migrations',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
})
```

### Database Client

```typescript
// server/database/index.ts
import { drizzle } from 'drizzle-orm/node-postgres'
import * as schema from './schema'

export const db = drizzle(process.env.DATABASE_URL!, { schema })
```

### API Endpoints Consumed
*This spec covers infrastructure only. No application-level API endpoints are defined here. See individual feature specs for endpoint definitions.*

### TypeScript Interfaces

```typescript
// Nuxt 4 runtime config types (auto-generated from nuxt.config.ts)
interface RuntimeConfig {
  databaseUrl: string
  redisUrl: string
  jwtSecret: string
  jwtTtl: string
  jwtRefreshTtl: string
  centrifugoApiUrl: string
  centrifugoApiKey: string
  centrifugoSecret: string
  telegramBotToken: string
  storagePath: string
  maxFileSize: number
  public: {
    wsBaseUrl: string
    socketDebug: boolean
  }
}
```

## State Management
No Pinia stores are defined at the infrastructure level. Runtime config values are accessed server-side via `useRuntimeConfig()` inside Nitro event handlers and server utilities. Client-side public config values are accessed via `useRuntimeConfig()` in composables and plugins.

## Edge Cases
- **Missing `JWT_SECRET`**: Application must refuse to start (or throw at first JWT operation) if `JWT_SECRET` is absent or shorter than 32 characters.
- **Database unreachable at startup**: Drizzle does not eagerly connect; the first query will fail. Nitro startup should be validated by running `npx drizzle-kit migrate` before starting the app container.
- **`compose.dev.yaml` volume not cleared between schema resets**: Running `docker compose down -v` removes `pgdata`; developers must re-run the seeder after volume removal.
- **`STORAGE_PATH` points to a non-existent directory**: Upload handlers must create missing directories (`mkdirSync` with `{ recursive: true }`) before writing files.
- **`MAX_FILE_SIZE` not an integer**: `parseInt` silently returns `NaN`; validate this at startup and fall back to the 10 MB default.
- **Centrifugo config JSON missing or malformed**: The `centrifugo` Docker service will exit immediately; a `centrifugo.json` template must be committed to the repository.
- **`compose.dev.yaml` vs `.env` port conflicts**: If local PostgreSQL (port 5432) or Redis (port 6379) is already running, the Docker services will fail to bind. Developers must stop conflicting local services or remap ports.
- **`npx drizzle-kit push` in production**: This command bypasses the migration history and must never be run against a production database.

## Non-Goals
- This spec does **not** define any database table schemas — see `specs/database-schema.md`.
- This spec does **not** cover authentication logic (JWT issuance, middleware) — see `specs/authentication.md`.
- This spec does **not** define Centrifugo channel permissions or real-time event types — see `specs/realtime-centrifugo.md`.
- This spec does **not** cover BullMQ queue definitions or job implementations — see individual feature specs.
- This spec does **not** define application-level API routes.
- This spec does **not** cover frontend component architecture — see `specs/frontend-architecture.md`.
- This spec does **not** define Tailwind design tokens or CSS conventions beyond noting Tailwind CSS 4 is used.
- This spec does **not** cover CI/CD pipeline configuration.
- This spec does **not** cover TLS/SSL termination or reverse proxy configuration.

## Dependencies

### Runtime Packages (Server)
| Package | Purpose |
|---|---|
| `drizzle-orm` | PostgreSQL ORM — query builder and schema definition |
| `pg` | Node.js PostgreSQL driver (used by drizzle-orm/node-postgres) |
| `jose` | JWT issuance and verification (HS256) |
| `bcrypt` | Password hashing |
| `bullmq` | Redis-backed job queue |
| `ioredis` | Redis client (used by BullMQ and direct cache access) |
| `zod` | Runtime validation of request bodies and responses |
| `formidable` | Multipart form data parsing for file uploads |

### Dev Packages (Server)
| Package | Purpose |
|---|---|
| `drizzle-kit` | Migration generation, schema push, Drizzle Studio |
| `@types/pg` | TypeScript types for `pg` |
| `@types/bcrypt` | TypeScript types for `bcrypt` |

### Runtime Packages (Client)
| Package | Purpose |
|---|---|
| `pinia` | State management (Setup Stores) |
| `centrifuge` | Centrifugo WebSocket client |
| `gsap` | Animations (ScrollTrigger, ScrollToPlugin) |
| `vue-sonner` | Toast notifications |
| `vue-draggable-plus` | Drag-and-drop for kanban cards/lists |
| `vue-iconsax` | Iconsax icon set for Vue 3 |
| `vue3-persian-datetime-picker` | Persian/Jalali date picker component |
| `hammerjs` | Touch gesture support |

### Nuxt Modules
| Module | Purpose |
|---|---|
| `@pinia/nuxt` | Pinia integration for Nuxt (auto-imports) |
| `@nuxt/image` | Optimized image component |
| `@nuxt/icon` | Icon system integration |
| `nuxt-easy-lightbox` | Image lightbox viewer |
| `@tailwindcss/vite` | Tailwind CSS 4 via Vite plugin |

### External Services
| Service | Image | Purpose |
|---|---|---|
| PostgreSQL 16 | `postgres:16-alpine` | Primary relational database |
| Redis 7 | `redis:7-alpine` | Cache, BullMQ queue backend, JWT token blacklist |
| Centrifugo v5 | `centrifugo/centrifugo:v5` | Real-time WebSocket pub/sub server |
| Telegram Bot API | External | Push notifications via Telegram |

### CLI Tools
| Command | Purpose |
|---|---|
| `npx drizzle-kit generate` | Generate SQL migration file from schema changes |
| `npx drizzle-kit migrate` | Apply all pending migrations to the database |
| `npx drizzle-kit push` | Push schema directly to database (dev only) |
| `npx drizzle-kit studio` | Open Drizzle Studio web UI for database inspection |
| `npx tsx server/database/seed.ts` | Run the database seeder |
