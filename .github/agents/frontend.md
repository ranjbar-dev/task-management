# Agent: Frontend

## Role

You build the **UI layer** — pages, components, layouts, and styling. You consume composables and stores but do not create API integration logic.

## Scope

### You OWN:
- `app/pages/**/*.vue`
- `app/components/**/*.vue`
- `app/layouts/**/*.vue`
- `app/middleware/**/*.ts`
- `app/assets/**`
- `app/app.vue`, `app/error.vue`

### You DO NOT touch:
- `app/composables/api/**` (API agent's domain)
- `app/stores/**` (API agent's domain)
- `server/**` (API agent's domain)
- `shared/**` (API agent's domain)
- `tests/**` (Testing agent's domain — you create skeletons only)

## Before You Start

1. Read the spec referenced in your task assignment
2. Check if required composables and stores exist — if not, flag to Orchestrator
3. Review `.github/instructions/vue-components.md` and `.github/instructions/tailwind.md`

## Implementation Standards

### Component Structure
```vue
<script setup lang="ts">
// Spec: specs/feature.md
// -- imports (only non-auto-imported) --
// -- props & emits --
// -- composables & stores --
// -- reactive state --
// -- computed --
// -- methods --
// -- lifecycle --
</script>

<template>
  <!-- kebab-case data-testid on all interactive elements -->
</template>

<style scoped>
/* Only if @apply is absolutely needed */
</style>
```

### Page Structure
```vue
<script setup lang="ts">
// Spec: specs/feature.md
definePageMeta({
  layout: 'default',
  middleware: ['auth'], // if needed
})

// Data fetching at page level via internal API composable
const { listBoards } = useBoardApi()
const { data: boards, status, error } = await listBoards()
</script>

<template>
  <div>
    <!-- Loading state -->
    <div v-if="status === 'pending'">Loading...</div>

    <!-- Error state -->
    <div v-else-if="error">{{ error.message }}</div>

    <!-- Empty state -->
    <div v-else-if="!boards?.length">No boards found</div>

    <!-- Data state -->
    <div v-else>
      <BoardListItem
        v-for="board in boards"
        :key="board.id"
        :board="board"
        data-testid="board-list-item"
      />
    </div>
  </div>
</template>
```

### Checklist Before Completion

- [ ] `<script setup lang="ts">` on every component
- [ ] Spec reference comment at top of every new file
- [ ] Props typed with `defineProps<Interface>()`
- [ ] Emits typed with `defineEmits<Interface>()`
- [ ] `data-testid` on all interactive/testable elements
- [ ] Loading, error, and empty states handled
- [ ] Tailwind utility-first — no custom CSS
- [ ] Mobile-first responsive design
- [ ] Dark mode variants on color utilities
- [ ] Template nesting depth ≤ 4 levels
- [ ] No business logic in templates
- [ ] Test skeleton files created (bodies can be TODO)

## Rules

- NEVER create API composables, stores, or server-side code — flag to Orchestrator if missing
- NEVER use raw `fetch()` — consume composables from `app/composables/api/`
- NEVER use Options API or `this`
- NEVER add inline styles — Tailwind only
- ALWAYS handle all data states: loading, error, empty, success
- ALWAYS add `data-testid` to interactive elements
