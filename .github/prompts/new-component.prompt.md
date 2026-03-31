# New Component

Scaffold a reusable Vue component from a spec or parent page context.

## Context

You are the Frontend agent. You are creating a component that will be used by a page or other components.

## Inputs

- **Spec file:** {{ spec_path }} (must exist in `specs/`)
- **Component name:** {{ component_name }} (kebab-case, e.g., `board-card-item`)
- **Parent:** {{ parent_path }} (page or component that will use this)

## Pre-Flight Checklist

1. Read the spec — confirm this component is listed in the Component Tree
2. Check if a similar component already exists in `app/components/`
3. Identify props, emits, and slots from the spec or parent context

## Template

```vue
<script setup lang="ts">
// Spec: {{ spec_path }}

interface Props {
  // derived from spec
}

interface Emits {
  // derived from spec
}

const props = defineProps<Props>()
const emit = defineEmits<Emits>()

// Composables
// Reactive state
// Computed
// Methods
</script>

<template>
  <div data-testid="{{ component_name }}">
    <!-- Implementation -->
  </div>
</template>
```

## Steps

1. Define the component interface (props, emits, slots) from the spec
2. Implement the template with Tailwind utilities
3. Extract complex logic into composables — keep the component thin
4. Add `data-testid` to all interactive elements
5. Create unit test file at `tests/unit/components/<feature>/{{ component_name }}.test.ts`

## Output Files

```
app/components/<feature>/{{ component_name }}.vue
tests/unit/components/<feature>/{{ component_name }}.test.ts
```

## Quality Checks

- [ ] `<script setup lang="ts">` with typed props/emits
- [ ] No business logic in `<template>`
- [ ] All interactive elements have `data-testid`
- [ ] Responsive: mobile-first Tailwind classes
- [ ] Dark mode variants for all color utilities
- [ ] Template nesting depth ≤ 4 levels
- [ ] Spec reference comment at top of `<script setup>`
