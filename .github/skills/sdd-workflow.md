# Skill: Spec-Driven Development (SDD) Workflow

## Overview

Every feature in this project follows the SDD lifecycle. No code is written without an approved spec. This skill defines the spec format, lifecycle, and validation process.

---

## Spec Lifecycle

```
draft → review → approved → in-progress → done
  │        │         │            │          │
  │        │         │            │          └─ All ACs implemented + tested
  │        │         │            └─ Agents are actively implementing
  │        │         └─ Developer approved, ready for implementation
  │        └─ Spec written, awaiting developer review
  └─ Initial creation by Orchestrator
```

### Status Rules

| Status | Who Sets It | What Happens |
|--------|------------|--------------|
| `draft` | Orchestrator | Spec created, needs developer review |
| `review` | Orchestrator | Spec complete, flagged for developer |
| `approved` | Developer | Developer confirms, implementation can begin |
| `in-progress` | Orchestrator | Tasks assigned to agents, work underway |
| `done` | Orchestrator | All ACs pass, tests pass, validated |

**Hard rule:** No agent may begin implementation on a spec with status `draft` or `review`.

---

## Spec File Format

All specs live in `specs/` at the project root.

Filename: `specs/<feature-name>.md` (kebab-case)

```markdown
# Spec: <Feature Name>

**Status:** draft | review | approved | in-progress | done
**Created:** YYYY-MM-DD
**Author:** <name or agent>
**Last Updated:** YYYY-MM-DD

---

## Purpose

One paragraph: what this feature does and why it exists.

## User Stories

- As a [role], I want to [action] so that [benefit]

## Acceptance Criteria

- [ ] AC-1: <Testable statement>
- [ ] AC-2: <Testable statement>
- [ ] AC-3: <Testable statement>

Each AC must be:
- **Specific** — no ambiguity
- **Testable** — can be verified with a unit or e2e test
- **Independent** — doesn't depend on other ACs to be meaningful

## Component Tree

```
app/pages/<route>.vue
├── app/components/<feature>/header.vue
├── app/components/<feature>/list.vue
│   └── app/components/<feature>/list-item.vue
└── app/components/<feature>/form.vue
```

## Data Model

### API Endpoints Consumed

| Method | Endpoint | Request Body/Params | Response Type |
|--------|----------|-------------------|---------------|
| GET    | /api/client/resource/all | `?page=1&perPage=20` | ResourceListResponse |
| POST   | /api/client/resource/create | CreateResourceInput | Resource |

### Zod Schemas

```typescript
const ResourceSchema = z.object({
  id: z.number(),
  name: z.string(),
  // ...
})
```

### TypeScript Interfaces

```typescript
interface Resource {
  id: number
  name: string
}

interface CreateResourceInput {
  name: string
}
```

## State Management

- **Store:** `app/stores/<domain>.ts` (if needed)
- **State variables:**
  - `items: Resource[]` — the list of resources
  - `isLoading: boolean` — fetch in progress
  - `error: string | undefined` — last error message

If no store is needed, state can live in composables or page-level refs.

## Edge Cases

- Empty state: API returns zero results
- Error state: API returns 4xx/5xx
- Loading state: slow network
- Stale data: user navigates away and back
- Validation: invalid user input

## Non-Goals

Explicitly list what this feature does NOT include.

## Dependencies

- **External APIs:** list endpoints
- **Existing composables:** list reusable composables
- **New packages:** list any new npm packages needed (with justification)

## Task Breakdown (Orchestrator fills this)

| # | Task | Agent | Status | Output Files |
|---|------|-------|--------|-------------|
| 1 | Create Drizzle schema | API | pending | server/database/schema/resource.ts |
| 2 | Create Zod schemas + shared types | API | pending | shared/schemas/resource.schema.ts, shared/types/resource.types.ts |
| 3 | Create Nitro route handlers | API | pending | server/api/client/resource/*.ts |
| 4 | Create transformer | API | pending | server/transformers/resource.transformer.ts |
| 5 | Create API composable | API | pending | app/composables/api/use-resource-api.ts |
| 6 | Create store | API | pending | app/stores/resource.ts |
| 7 | Create page + components | Frontend | pending | app/pages/resource.vue, app/components/resource/*.vue |
| 8 | Write unit + e2e tests | Testing | pending | tests/unit/**, tests/e2e/resource.spec.ts |

```

---

## How Agents Reference Specs

Every file created as part of a spec must include a reference comment:

```typescript
// Spec: specs/feature-name.md
```

```vue
<script setup lang="ts">
// Spec: specs/feature-name.md
</script>
```

This creates traceability from code back to requirements.

---

## Validation Checklist (Orchestrator runs this at the end)

```markdown
## Validation: specs/<feature-name>.md

### Acceptance Criteria
- [ ] AC-1: Implemented in [file] — Tested in [test file]
- [ ] AC-2: Implemented in [file] — Tested in [test file]
- [ ] AC-3: Implemented in [file] — Tested in [test file]

### Quality Gates
- [ ] TypeScript compiles with zero errors
- [ ] ESLint + Prettier pass with zero errors
- [ ] All unit tests pass
- [ ] All e2e tests pass
- [ ] No `any` types in new code
- [ ] No raw `fetch()` calls
- [ ] All API responses validated with Zod
- [ ] All interactive elements have `data-testid`

### Sign-off
- [ ] Orchestrator validated completeness
- [ ] Spec status updated to `done`
```

---

## Quick Start for Developers

### Starting a new feature:

1. Run the `write-spec` prompt with your feature idea
2. Review the generated spec in `specs/`
3. Change status to `approved` when satisfied
4. Orchestrator takes over: breaks into tasks and delegates

### Fixing a bug:

1. Run the `fix-bug` prompt with reproduction steps
2. No spec needed — Testing agent writes regression test first
3. Implementation agent applies the fix
4. Orchestrator verifies regression test passes

### Checking progress:

Look at the Task Breakdown table in the spec file — each task has a status column updated by the assigned agent.
