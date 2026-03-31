# State Management — Pinia Conventions

## Store Location

All Pinia stores live in `app/stores/`.
Nuxt auto-imports stores from this directory.

## Setup Store Syntax (Mandatory)

Always use the Setup Store syntax. Do not use the Options Store syntax.

```typescript
// app/stores/auth.ts
// Spec: specs/auth.md

export const useAuthStore = defineStore('auth', () => {
  // State
  const user = ref<User | null>(null)
  const token = ref<string | null>(null)

  // Getters (computed)
  const isAuthenticated = computed(() => !!token.value)
  const userName = computed(() => user.value?.name ?? 'Guest')

  // Actions
  async function login(credentials: LoginCredentials): Promise<void> {
    const response = await $fetch<AuthResponse>('/api/auth/login', {
      method: 'POST',
      body: credentials,
    })
    user.value = response.user
    token.value = response.token
  }

  function logout(): void {
    user.value = null
    token.value = null
    navigateTo('/login')
  }

  // Expose only what consumers need
  return {
    // State (readonly where possible)
    user: readonly(user),
    token: readonly(token),
    // Getters
    isAuthenticated,
    userName,
    // Actions
    login,
    logout,
  }
})
```

## One Store Per Domain

Each business domain gets exactly one store:

```
app/stores/
├── auth.ts          # Authentication & user session
├── board.ts         # Board data & operations
├── notifications.ts # Toast & notification state
└── ui.ts            # UI state (sidebar, modals, theme)
```

Do not create micro-stores for every component. Do not create god-stores that handle everything.

## Store Dependencies — No Direct Cross-References

Stores must NOT import other stores directly. Use composables as mediators:

```typescript
// BAD — store importing store
export const useBoardStore = defineStore('board', () => {
  const authStore = useAuthStore() // ❌ Direct dependency
})

// GOOD — composable mediates
// app/composables/use-board-access.ts
export function useBoardAccess() {
  const authStore = useAuthStore()
  const boardStore = useBoardStore()

  const canManageBoard = computed(() => authStore.isAuthenticated && boardStore.isReady)

  return { canManageBoard }
}
```

## Component Access Pattern

Components should prefer composables over direct store access for complex logic:

```typescript
// Simple read — direct store access is fine
const authStore = useAuthStore()
const isLoggedIn = authStore.isAuthenticated

// Complex logic — wrap in a composable
const { canManageBoard, archiveBoard } = useBoardActions()
```

## Persistence

If a store needs persistence (e.g., auth tokens), use `pinia-plugin-persistedstate`:

```typescript
export const useAuthStore = defineStore('auth', () => {
  // ... setup store code
}, {
  persist: {
    pick: ['token'],
    storage: piniaPluginPersistedstate.localStorage(),
  },
})
```

## Rules

- Setup Store syntax only — no Options syntax
- One store per domain
- No store-to-store imports — use composables as mediators
- Expose `readonly()` refs for state that shouldn't be mutated externally
- All actions that call APIs must handle errors
- Store file names match the domain: `auth.ts`, not `useAuthStore.ts`
- Every store file references its spec: `// Spec: specs/<domain>.md`
