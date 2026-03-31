# New Composable

Create a typed composable from a spec.

## Context

You are the API agent. You are creating a composable that encapsulates reusable logic.

## Inputs

- **Spec file:** {{ spec_path }} (must exist in `specs/`)
- **Composable name:** {{ composable_name }} (e.g., `use-board-api`, `use-form-validation`)
- **Type:** {{ type }} — `api` (external API wrapper) | `logic` (reusable business logic) | `ui` (UI behavior)

## Pre-Flight Checklist

1. Read the spec — confirm this composable is referenced
2. Check `app/composables/` for existing composables that overlap
3. Identify the return type interface

## Template

```typescript
// app/composables/{{ path }}/{{ composable_name }}.ts
// Spec: {{ spec_path }}

interface {{ ReturnType }} {
  // all public members
}

export function {{ composableName }}(/* params */): {{ ReturnType }} {
  // Implementation

  return {
    // Only expose what consumers need
  }
}
```

### For API composables specifically:

```typescript
// app/composables/api/{{ composable_name }}.ts
// Spec: {{ spec_path }}

import { z } from 'zod'

// 1. Define Zod schemas for API responses (or import from shared/schemas/)
const ResponseSchema = z.object({ /* ... */ })
type ResponseType = z.infer<typeof ResponseSchema>

// 2. Export composable
export function {{ composableName }}() {
  // 3. Read operations use useFetch with relative internal routes
  function list(params?: ListParams) {
    return useFetch<ResponseType>('/api/client/{{ resource }}/all', {
      query: params,
      key: '{{ resource }}-list',
      transform: (raw) => ResponseSchema.parse(raw),
    })
  }

  // 4. Write operations use $fetch
  async function create(payload: CreateInput): Promise<ResponseType> {
    const raw = await $fetch('/api/client/{{ resource }}/create', {
      method: 'POST',
      body: payload,
    })
    return ResponseSchema.parse(raw)
  }

  return { list, create }
}
```

## Output Files

```
app/composables/{{ path }}/{{ composable_name }}.ts
tests/unit/composables/{{ path }}/{{ composable_name }}.test.ts
```

## Quality Checks

- [ ] Explicit return type interface
- [ ] Zod validation on API data (API composables)
- [ ] Relative paths for internal Nitro routes — no `apiBaseUrl` (API composables)
- [ ] Unique keys for `useFetch`/`useAsyncData`
- [ ] Error handling — no unhandled rejections
- [ ] Unit test with happy path + error case
- [ ] Spec reference comment at top
