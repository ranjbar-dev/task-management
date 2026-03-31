# TypeScript — Strict Mode Patterns

## Configuration

The project uses strict TypeScript. These flags are non-negotiable:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": false
  }
}
```

## Type Definitions

### Interfaces for object shapes
```typescript
// Use interface for objects — they're extendable and produce better error messages
interface User {
  id: number
  name: string
  email: string
  role: UserRole
}

interface PaginatedResponse<T> {
  data: T[]
  meta: {
    total: number
    page: number
    perPage: number
  }
}
```

### Types for unions, intersections, and aliases
```typescript
// Use type for unions and computed types
type ApiResult<T> = { success: true; data: T } | { success: false; error: string }
type Nullable<T> = T | null

// Derive union types from const enum objects
type UserRole = typeof UserRoleEnum[keyof typeof UserRoleEnum]
```

### Discriminated unions for state
```typescript
// Model async states explicitly
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string }
```

## Banned Patterns

| Pattern | Reason | Alternative |
|---------|--------|------------|
| `any` | Defeats type safety | `unknown` with type guard |
| `@ts-ignore` | Hides real errors | Fix the type or `@ts-expect-error` with issue link |
| `as` type assertion | Unsafe cast | Type guard or `satisfies` |
| `!` non-null assertion | Runtime risk | Explicit null check |
| `enum` | Tree-shaking issues | `const` object + `type` union |

## Enums Alternative

```typescript
// Instead of enum, use const objects with numeric values
// All enums live in shared/enums/
const UserRoleEnum = {
  NormalUser: 0,
  AdminUser: 1,
} as const

type UserRole = typeof UserRoleEnum[keyof typeof UserRoleEnum]

const BoardStatusEnum = {
  Active: 0,
  Archived: 1,
} as const

type BoardStatus = typeof BoardStatusEnum[keyof typeof BoardStatusEnum]
```

## Generic Patterns

```typescript
// Composable return types must be explicit
interface UseApiReturn<T> {
  data: Ref<T | undefined>
  error: Ref<string | undefined>
  isLoading: Ref<boolean>
  execute: () => Promise<void>
}

function useApi<T>(url: string): UseApiReturn<T> {
  // implementation
}
```

## Zod for Runtime Validation

Zod schemas live in `shared/schemas/` and are shared between server and client.
Validate API request bodies on the server and API responses on the client:

```typescript
import { z } from 'zod'

// shared/schemas/board.schema.ts
const CreateBoardSchema = z.object({
  title: z.string().min(1).max(255),
  description: z.string().optional(),
  color: z.string().regex(/^#[0-9a-f]{6}$/i).optional(),
})

type CreateBoardInput = z.infer<typeof CreateBoardSchema>

// Server-side: validate request body
const body = await readValidatedBody(event, CreateBoardSchema.parse)

// Client-side: validate API response
const validated = BoardResponseSchema.parse(apiResponse)
```

## Shared Types

Types used by both server and client go in `shared/types/`.
Enums go in `shared/enums/`. Zod schemas go in `shared/schemas/`.
App-only types live alongside their composable or component.
Server-only types live alongside their route handler or service.

### Drizzle ORM Types

```typescript
import type { InferSelectModel, InferInsertModel } from 'drizzle-orm'
import { boards } from '~/server/database/schema/boards'

type Board = InferSelectModel<typeof boards>
type NewBoard = InferInsertModel<typeof boards>
```
