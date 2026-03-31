# Testing — Vitest + Playwright Conventions

## Test Structure

```
tests/
├── unit/                    # Vitest — mirrors app/ + server/ structure
│   ├── components/
│   │   └── board-card.test.ts
│   ├── composables/
│   │   └── api/
│   │       └── use-board-api.test.ts
│   ├── stores/
│   │   └── auth.test.ts
│   └── server/                # Server-side tests
│       ├── api/
│       ├── services/
│       └── transformers/
└── e2e/                     # Playwright — user-flow based
    ├── auth.spec.ts
    ├── boards.spec.ts
    └── navigation.spec.ts
```

## Vitest — Unit & Component Tests

### File naming
- `<source-file>.test.ts` — mirrors the source file name
- Located in `tests/unit/` mirroring the app structure

### Component test pattern
```typescript
// tests/unit/components/board-card.test.ts
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { describe, it, expect } from 'vitest'
import BoardCard from '~/components/board/board-card.vue'

describe('BoardCard', () => {
  it('renders board name and status', async () => {
    const wrapper = await mountSuspended(BoardCard, {
      props: {
        board: {
          id: 1,
          name: 'Sprint Board',
          status: 1,
          icon: 0,
        },
      },
    })

    expect(wrapper.text()).toContain('Sprint Board')
  })

  it('emits select event on click', async () => {
    const wrapper = await mountSuspended(BoardCard, {
      props: { board: mockBoard },
    })

    await wrapper.trigger('click')
    expect(wrapper.emitted('select')).toHaveLength(1)
  })
})
```

### Composable test pattern
```typescript
// tests/unit/composables/api/use-board-api.test.ts
import { describe, it, expect, vi } from 'vitest'

describe('useBoardApi', () => {
  it('returns typed board data', async () => {
    // Mock $fetch at the Nuxt level
    vi.stubGlobal('$fetch', vi.fn().mockResolvedValue(mockBoardResponse))

    const { listBoards } = useBoardApi()
    const { data } = await listBoards()

    expect(data.value).toBeDefined()
    expect(data.value?.data).toHaveLength(2)
  })

  it('throws on invalid API response shape', async () => {
    vi.stubGlobal('$fetch', vi.fn().mockResolvedValue({ invalid: true }))

    const { listBoards } = useBoardApi()
    // Zod validation should reject malformed response
    await expect(listBoards()).rejects.toThrow()
  })
})
```

### Store test pattern
```typescript
// tests/unit/stores/auth.test.ts
import { setActivePinia, createPinia } from 'pinia'
import { describe, it, expect, beforeEach } from 'vitest'

describe('useAuthStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('starts unauthenticated', () => {
    const store = useAuthStore()
    expect(store.isAuthenticated).toBe(false)
  })

  it('authenticates after login', async () => {
    const store = useAuthStore()
    await store.login({ email: 'test@test.com', password: 'pass' })
    expect(store.isAuthenticated).toBe(true)
  })
})
```

## Playwright — E2E Tests

### File naming
- `<feature>.spec.ts` — one file per user-facing flow
- Located in `tests/e2e/`

### Test pattern
```typescript
// tests/e2e/auth.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Authentication', () => {
  test('user can log in and see dashboard', async ({ page }) => {
    await page.goto('/login')
    await page.fill('[data-testid="email"]', 'user@test.com')
    await page.fill('[data-testid="password"]', 'password')
    await page.click('[data-testid="login-btn"]')

    await expect(page).toHaveURL('/dashboard')
    await expect(page.locator('[data-testid="user-name"]')).toBeVisible()
  })

  test('shows error for invalid credentials', async ({ page }) => {
    await page.goto('/login')
    await page.fill('[data-testid="email"]', 'wrong@test.com')
    await page.fill('[data-testid="password"]', 'wrong')
    await page.click('[data-testid="login-btn"]')

    await expect(page.locator('[data-testid="error-message"]')).toBeVisible()
  })
})
```

## API Mocking

Use MSW (Mock Service Worker) for consistent API mocking across unit and e2e:

```typescript
// tests/mocks/handlers.ts
import { http, HttpResponse } from 'msw'

export const handlers = [
  http.get('*/api/client/board/all', () => {
    return HttpResponse.json({
      data: [mockBoard],
    })
  }),
]
```

## Data Test IDs

All interactive elements must have `data-testid` attributes:

```html
<button data-testid="create-board-btn">Create Board</button>
<input data-testid="board-name-input" />
<div data-testid="board-list">...</div>
```

Naming: `kebab-case`, descriptive: `<context>-<element>` (e.g., `board-form-submit-btn`).

## Minimum Coverage Requirements

Every new feature must include:
- **Happy path test** — the primary use case works
- **One error case** — at least one failure scenario
- **E2e test** — for user-facing flows only (not internal utilities)

## Rules

- Test files mirror source structure in `tests/unit/`
- E2e files are flow-based in `tests/e2e/`
- Mock external APIs with MSW — never hit real backends in tests
- All interactive elements get `data-testid`
- No `test.skip` or `test.todo` in committed code without an issue link
- Tests must be deterministic — no reliance on timing or external state
