# Skill: Nuxt 4 Patterns & Gotchas

## Quick Reference for Agents

This skill contains Nuxt 4-specific patterns, breaking changes from Nuxt 3, and common pitfalls. All agents should reference this when working with Nuxt-specific APIs.

---

## Directory Structure (Nuxt 4)

Nuxt 4 uses `app/` for client code and `server/` for the API backend:

```
project-root/
├── app/                              # SPA client (Vue 3)
│   ├── components/
│   ├── composables/
│   │   └── api/                      # Internal API composables
│   ├── layouts/
│   ├── middleware/
│   ├── pages/
│   ├── plugins/
│   ├── stores/                       # Pinia stores
│   ├── types/
│   ├── utils/
│   ├── app.vue
│   ├── app.config.ts
│   └── error.vue
├── server/                           # Nitro server (API backend)
│   ├── api/                          # Route handlers (file-based routing)
│   │   ├── auth/
│   │   ├── admin/
│   │   └── client/
│   ├── database/                     # Drizzle ORM
│   │   ├── schema/
│   │   ├── migrations/
│   │   └── index.ts
│   ├── middleware/                    # Server middleware (auth, admin, active-user)
│   ├── services/
│   ├── transformers/
│   ├── jobs/
│   ├── utils/
│   └── plugins/
├── shared/                           # Shared between server & client
│   ├── enums/
│   ├── schemas/                      # Zod validation schemas
│   ├── types/
│   └── utils/
├── public/
├── nuxt.config.ts
├── drizzle.config.ts
└── tailwind.config.js
```

**Key change:** The `~` and `@` aliases point to `app/` in Nuxt 4, not the project root.

---

## Auto-Imports

Nuxt 4 auto-imports from these sources — do NOT manually import them:

### Vue Core
- `ref`, `reactive`, `computed`, `watch`, `watchEffect`
- `onMounted`, `onUnmounted`, `onBeforeMount`, `onBeforeUnmount`
- `provide`, `inject`
- `nextTick`
- `toRef`, `toRefs`, `unref`, `isRef`
- `readonly`, `shallowRef`, `shallowReactive`

### Nuxt
- `useRuntimeConfig`, `useAppConfig`
- `useRoute`, `useRouter`, `navigateTo`
- `useFetch`, `useAsyncData`, `$fetch`
- `useCookie`, `useRequestHeaders`
- `definePageMeta`, `defineNuxtComponent`
- `createError`, `showError`, `clearError`
- `useState`
- `useHead`, `useSeoMeta`
- `NuxtLink`, `NuxtPage`, `NuxtLayout`

### VueUse (if installed)
- All VueUse composables are auto-imported

### Pinia
- `defineStore`, `storeToRefs`

### Project
- Everything in `app/composables/` — auto-imported by filename
- Everything in `app/utils/` — auto-imported by filename
- Everything in `shared/utils/` — auto-imported

---

## Breaking Changes from Nuxt 3

### 1. Default values: `undefined` not `null`

```typescript
// Nuxt 4: data and error default to undefined
const { data, error } = await useFetch('/api/resource')

// ✅ Correct
if (data.value === undefined) { /* no data yet */ }

// ❌ Wrong (Nuxt 3 pattern)
if (data.value === null) { /* won't work */ }
```

### 2. Shared reactive refs by key

```typescript
// In Nuxt 4, same key = same reactive ref
const { data: a } = useFetch('/api/foo', { key: 'my-data' })
const { data: b } = useFetch('/api/bar', { key: 'my-data' })
// a and b point to the SAME ref — data collision!

// ✅ Always use unique keys
const { data: a } = useFetch('/api/foo', { key: 'foo-data' })
const { data: b } = useFetch('/api/bar', { key: 'bar-data' })
```

### 3. Component naming from path

```
app/components/board/card-item.vue → <BoardCardItem />
app/components/ui/base-button.vue  → <UiBaseButton />
```

Nuxt 4 standardizes component names from the full directory + file path. This applies everywhere component names are used as strings (e.g., `KeepAlive`'s `include`).

### 4. `getCachedData` fires on refetch

In Nuxt 4, `getCachedData` fires for refetches triggered by watchers and `refreshNuxtData`, not just initial loads. It now receives a context argument:

```typescript
useFetch('/api/data', {
  getCachedData(key, nuxtApp, ctx) {
    // ctx.cause tells you why: 'initial' | 'watcher' | 'refresh'
    if (ctx.cause === 'refresh') return undefined // force re-fetch
    return nuxtApp.payload.data[key]
  },
})
```

### 5. Error creation — Web API naming

Nuxt 4 is transitioning to Web API naming conventions (ahead of Nuxt 5):

```typescript
// ✅ New (Nuxt 4+)
throw createError({ status: 404, statusText: 'Not Found' })

// ⚠️ Deprecated (still works but will break in Nuxt 5)
throw createError({ statusCode: 404, statusMessage: 'Not Found' })
```

---

## Page Patterns

### definePageMeta

```typescript
definePageMeta({
  layout: 'dashboard',
  middleware: ['auth'],
  title: 'Boards',
  // Nuxt 4: can use function form for dynamic meta
})
```

### Error Page

```vue
<!-- app/error.vue -->
<script setup lang="ts">
import type { NuxtError } from '#app'

const props = defineProps<{
  error: NuxtError
}>()

function handleClear() {
  clearError({ redirect: '/' })
}
</script>

<template>
  <div>
    <h1>{{ error.status }} — {{ error.statusText }}</h1>
    <button @click="handleClear">Go Home</button>
  </div>
</template>
```

### Middleware

```typescript
// app/middleware/auth.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const authStore = useAuthStore()

  if (!authStore.isAuthenticated) {
    return navigateTo('/login')
  }
})
```

---

## Performance Patterns

### Lazy Components

Prefix with `Lazy` to defer loading:

```html
<LazyBoardOverview v-if="showOverview" :data="boardData" />
```

### Client-Only Rendering

For browser-only components:

```html
<ClientOnly>
  <BoardOverview :data="boardData" />
  <template #fallback>
    <div class="animate-pulse h-64 bg-gray-200 rounded" />
  </template>
</ClientOnly>
```

### Prefetching

`NuxtLink` automatically prefetches linked pages on viewport visibility. Disable for heavy pages:

```html
<NuxtLink to="/heavy-page" :prefetch="false">Heavy Page</NuxtLink>
```

---

## Server-Side Patterns (Nitro)

### Route Handler

```typescript
// server/api/client/board/all.get.ts → GET /api/client/board/all
export default defineEventHandler(async (event) => {
  const user = event.context.user

  const userBoards = await db.query.boardUsers.findMany({
    where: eq(boardUsers.userId, user.id),
    with: { board: true },
  })

  return successResponse(userBoards.map((bu) => transformBoard(bu.board)))
})
```

### Server Middleware

```typescript
// server/middleware/auth.ts
export default defineEventHandler(async (event) => {
  if (!getRequestURL(event).pathname.startsWith('/api/')) return
  if (getRequestURL(event).pathname.startsWith('/api/auth/login')) return

  const token = getHeader(event, 'authorization')?.replace('Bearer ', '')
  if (!token) throw createError({ status: 401, statusText: 'Unauthorized' })

  const payload = await verifyToken(token)
  const user = await db.query.users.findFirst({ where: eq(users.id, payload.sub) })
  if (!user) throw createError({ status: 401, statusText: 'Unauthorized' })

  event.context.user = user
})
```

### Transformer Pattern

```typescript
// server/transformers/board.transformer.ts
export function transformBoard(board: typeof boards.$inferSelect) {
  return {
    id: board.id,
    name: board.name,
    description: board.description,
    status: board.status,
    icon: board.icon,
    created_at: board.createdAt,
  }
}
```

---

## Docker Deployment

The full-stack app is deployed via Docker with `compose.dev.yaml` and `compose.prod.yaml`.

Runtime config values can be overridden via environment variables at container startup using the `NUXT_PUBLIC_` prefix.
