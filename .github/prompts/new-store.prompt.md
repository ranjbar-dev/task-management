# New Store

Create a Pinia Setup Store from a spec.

## Context

You are the API agent. You are creating a Pinia store for a specific domain.

## Inputs

- **Spec file:** {{ spec_path }} (must exist in `specs/`)
- **Domain name:** {{ domain }} (e.g., `auth`, `board`, `notifications`)

## Pre-Flight Checklist

1. Read the spec — confirm state management is needed for this domain
2. Check `app/stores/` for existing stores that overlap — do NOT create duplicates
3. Identify state, getters, and actions from the spec

## Template

```typescript
// app/stores/{{ domain }}.ts
// Spec: {{ spec_path }}

export const use{{ Domain }}Store = defineStore('{{ domain }}', () => {
  // ── State ──────────────────────────────
  const items = ref<Item[]>([])
  const isLoading = ref(false)
  const error = ref<string | undefined>()

  // ── Getters ────────────────────────────
  const count = computed(() => items.value.length)
  const isEmpty = computed(() => items.value.length === 0)

  // ── Actions ────────────────────────────
  async function fetchItems(): Promise<void> {
    isLoading.value = true
    error.value = undefined
    try {
      // Use API composable — not raw fetch
      const { data } = await useItemsApi().list()
      items.value = data.value?.data ?? []
    } catch (e) {
      error.value = e instanceof Error ? e.message : 'Failed to fetch items'
    } finally {
      isLoading.value = false
    }
  }

  function reset(): void {
    items.value = []
    isLoading.value = false
    error.value = undefined
  }

  // ── Public API ─────────────────────────
  return {
    // State (readonly where appropriate)
    items: readonly(items),
    isLoading: readonly(isLoading),
    error: readonly(error),
    // Getters
    count,
    isEmpty,
    // Actions
    fetchItems,
    reset,
  }
})
```

## Output Files

```
app/stores/{{ domain }}.ts
tests/unit/stores/{{ domain }}.test.ts
```

## Quality Checks

- [ ] Setup Store syntax — no Options API
- [ ] Exposed state wrapped in `readonly()` where mutations should go through actions
- [ ] All async actions have try/catch with error state
- [ ] No direct imports of other stores — use composables as mediators
- [ ] Unit test: initial state, action success, action failure
- [ ] Store name matches domain: `defineStore('{{ domain }}')`
- [ ] Spec reference comment at top
