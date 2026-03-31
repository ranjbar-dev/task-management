# New Page

Scaffold a complete Nuxt 4 page from a spec.

## Context

You are the Frontend agent. You are creating a new page based on an approved spec.

## Inputs

- **Spec file:** {{ spec_path }} (must exist in `specs/`)
- **Page route:** {{ route }} (e.g., `/boards`, `/settings/profile`)

## Pre-Flight Checklist

1. Read the spec file — abort if it doesn't exist or status is `draft`
2. Identify all components listed in the spec's Component Tree
3. Identify all composables and stores needed
4. Check if referenced composables/stores already exist

## Steps

### 1. Create the page file
- Location: `app/pages/` following Nuxt file-based routing
- Include `definePageMeta()` with layout, middleware if specified in spec
- Add spec reference comment: `// Spec: {{ spec_path }}`
- Use `<script setup lang="ts">`

### 2. Create child components
- Location: `app/components/<feature>/`
- Follow kebab-case naming
- Type all props with interfaces
- Add `data-testid` attributes to interactive elements

### 3. Wire up data fetching
- Use existing API composables from `app/composables/api/`
- If new API endpoints are needed, flag for the API agent — do not create them yourself
- Handle loading, error, and empty states in the template

### 4. Wire up state management
- Use existing stores if they cover the domain
- If a new store is needed, flag for the API agent

### 5. Create test skeletons
- Create `tests/unit/components/<feature>/*.test.ts` for each new component
- Create `tests/e2e/<feature>.spec.ts` with test descriptions matching acceptance criteria
- Include `// TODO: implement` in test bodies

## Output Files

```
app/pages/<route>.vue
app/components/<feature>/*.vue
tests/unit/components/<feature>/*.test.ts
tests/e2e/<feature>.spec.ts
```

## Quality Checks

- [ ] All acceptance criteria from spec have corresponding UI elements
- [ ] All interactive elements have `data-testid`
- [ ] TypeScript compiles with zero errors
- [ ] No raw `fetch()` — only composables
- [ ] Tailwind utility classes only — no custom CSS
- [ ] Responsive: works on mobile viewport (375px+)
