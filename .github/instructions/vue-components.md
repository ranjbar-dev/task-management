# Vue Components — Nuxt 4 Patterns

## Component Structure

Every component follows this order:

```vue
<script setup lang="ts">
// 1. Type imports
// 2. Props & Emits
// 3. Composables & stores
// 4. Reactive state
// 5. Computed properties
// 6. Methods
// 7. Lifecycle hooks
// 8. Watchers
</script>

<template>
  <!-- Single root element preferred but not required (Vue 3 fragments OK) -->
</template>

<style scoped>
/* Tailwind @apply only when absolutely necessary */
</style>
```

## Props & Emits

```typescript
// Props — always use an interface
interface Props {
  title: string
  count?: number
  items: ReadonlyArray<Item>
}

const props = withDefaults(defineProps<Props>(), {
  count: 0,
})

// Emits — always typed
interface Emits {
  (e: 'update', value: string): void
  (e: 'delete', id: number): void
}

const emit = defineEmits<Emits>()
```

## Component Naming

- File names: `kebab-case.vue` (e.g., `user-profile-card.vue`)
- Nuxt 4 auto-resolves component names from directory + file path
- Prefix with directory path for clarity: `components/board/card-item.vue` → `<BoardCardItem />`

## Composition Patterns

### Extract logic into composables
```typescript
// BAD — logic in component
const count = ref(0)
const doubled = computed(() => count.value * 2)
function increment() { count.value++ }

// GOOD — extracted composable
const { count, doubled, increment } = useCounter()
```

### Use VueUse where applicable
```typescript
// Prefer VueUse over custom implementations
const { x, y } = useMouse()
const { toggle, value } = useToggle()
const { data, isLoading } = useFetch(url)
```

## Slots

- Type slots when possible using `defineSlots<T>()`
- Prefer named slots over complex prop drilling
- Use scoped slots to expose data to parent

```typescript
defineSlots<{
  default(props: { item: Item }): any
  header(): any
  footer(): any
}>()
```

## Rules

- No Options API — `<script setup>` only
- No `this` keyword
- No mixins — use composables
- No `v-html` unless sanitized
- Keep templates declarative — no complex expressions, extract to computed
- Components must be self-contained: props in, emits out
- Maximum template depth: 4 levels of nesting before extracting a sub-component
