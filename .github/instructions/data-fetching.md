# Data Fetching — Internal API Patterns

## Architecture

This is a full-stack Nuxt 4 app. The Nitro server handles all API routes. Client-side composables call internal Nitro routes using relative paths.

```
┌─ Client Side ─────────────────────────────────────────────────────┐
│ Component → Composable (app/composables/api/) → /api/... route    │
│                                                     ↓             │
│                                              Zod validates        │
│                                                     ↓             │
│                                              Typed data returned  │
└───────────────────────────────────────────────────────────────────┘

┌─ Server Side ─────────────────────────────────────────────────────┐
│ Nitro Route Handler → Drizzle ORM Query → Transformer → Response │
│      (defineEventHandler)    (PostgreSQL)      (shape output)     │
└───────────────────────────────────────────────────────────────────┘
```

## Client-Side Composable Pattern

Every API domain gets its own composable in `app/composables/api/`. All routes are internal Nitro endpoints — use relative paths:

```typescript
// app/composables/api/use-board-api.ts
// Spec: specs/board-management.md

import { z } from 'zod'

const BoardSchema = z.object({
  id: z.number(),
  name: z.string(),
  description: z.string(),
  status: z.number(),
  icon: z.number(),
})

const BoardListSchema = z.object({
  data: z.array(BoardSchema),
})

type Board = z.infer<typeof BoardSchema>
type BoardListResponse = z.infer<typeof BoardListSchema>

export function useBoardApi() {
  // Reads — useFetch with unique keys (relative paths to internal Nitro routes)
  function listBoards() {
    return useFetch<BoardListResponse>('/api/client/board/all', {
      key: 'board-list',
      transform: (raw) => BoardListSchema.parse(raw),
    })
  }

  function getBoard(id: number) {
    return useFetch<Board>(`/api/client/board/show/${id}`, {
      key: `board-${id}`,
      transform: (raw) => BoardSchema.parse(raw),
    })
  }

  // Writes — $fetch (not useFetch)
  async function createBoard(payload: { name: string; description: string }): Promise<Board> {
    const raw = await $fetch('/api/admin/board/create', { method: 'POST', body: payload })
    return BoardSchema.parse(raw)
  }

  async function updateBoard(id: number, payload: Partial<Board>): Promise<Board> {
    const raw = await $fetch(`/api/admin/board/${id}`, { method: 'PUT', body: payload })
    return BoardSchema.parse(raw)
  }

  async function deleteBoard(id: number): Promise<void> {
    await $fetch(`/api/admin/board/${id}`, { method: 'DELETE' })
  }

  return { listBoards, getBoard, createBoard, updateBoard, deleteBoard }
}
```

## Server-Side Route Handler Pattern

Nitro route handlers in `server/api/` use file-based routing:

```typescript
// server/api/client/board/all.get.ts → GET /api/client/board/all
import { db } from '~/server/database'
import { boards, boardUsers } from '~/server/database/schema'
import { transformBoard } from '~/server/transformers/board.transformer'

export default defineEventHandler(async (event) => {
  const user = event.context.user

  const userBoards = await db.query.boardUsers.findMany({
    where: eq(boardUsers.userId, user.id),
    with: { board: true },
  })

  return {
    data: userBoards.map((bu) => transformBoard(bu.board)),
  }
})
```

## useFetch vs useAsyncData

| Use Case | Tool | Why |
|----------|------|-----|
| Simple GET with URL | `useFetch` | Syntactic sugar, handles URL reactivity |
| Complex logic before fetch | `useAsyncData` | Full control over the fetch function |
| Non-GET mutations | `$fetch` + manual state | useFetch is for reads, not writes |

### Nuxt 4 Defaults

In Nuxt 4, `useFetch` and `useAsyncData` return `undefined` (not `null`) for default data and error values. Always check for `undefined`:

```typescript
const { data, error } = await useFetch('/api/client/board/all')

// CORRECT — Nuxt 4
if (data.value === undefined) { /* still loading or failed */ }

// WRONG — Nuxt 3 pattern
if (data.value === null) { /* won't catch undefined */ }
```

## Rules

- NEVER use raw `fetch()` in components — use `useFetch`, `useAsyncData`, or `$fetch`
- NEVER hardcode API URLs — use relative paths for internal Nitro routes
- ALWAYS validate API responses with Zod schemas
- ALWAYS provide unique `key` values to `useFetch`/`useAsyncData` to prevent data collision
- Use `useFetch` for reads, `$fetch` for mutations (POST/PUT/DELETE)

### Shared Keys

In Nuxt 4, reactive data from `useFetch`/`useAsyncData` using the **same key** shares a single reactive ref. Use unique keys to avoid cross-contamination:

```typescript
// Unique keys prevent data collision
const { data: boards } = useFetch('/api/client/board/all', { key: 'boards-list' })
const { data: stats } = useFetch('/api/client/board/stats', { key: 'boards-stats' })
```

## Error Handling

Wrap API errors consistently:

```typescript
// app/composables/api/use-api-error.ts
interface ApiError {
  status: number
  message: string
  details?: Record<string, string[]>
}

export function useApiError() {
  function handleError(error: unknown): ApiError {
    if (error instanceof z.ZodError) {
      return { status: 422, message: 'Invalid response from server', details: {} }
    }
    if (error && typeof error === 'object' && 'statusCode' in error) {
      const e = error as { statusCode: number; message?: string }
      return { status: e.statusCode, message: e.message || 'Request failed' }
    }
    return { status: 500, message: 'Unexpected error' }
  }

  return { handleError }
}
```

## Mutations (POST, PUT, DELETE)

Use `$fetch` directly for mutations — not `useFetch`:

```typescript
async function createBoard(payload: CreateBoardInput): Promise<Board> {
  const raw = await $fetch('/api/client/board/create', {
    method: 'POST',
    body: payload,
  })
  return BoardSchema.parse(raw)
}
```

## Rules

- No raw `fetch()` in components
- All API calls go through `app/composables/api/`
- All response shapes validated with Zod at the boundary
- Use relative paths for internal Nitro routes — no `apiBaseUrl`
- Unique keys for every `useFetch`/`useAsyncData` call
- Error handling is mandatory — no unhandled promise rejections
