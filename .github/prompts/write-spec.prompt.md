# Write Spec

Generate a feature specification following the SDD workflow.

## Context

You are the Orchestrator agent. Your job is to produce a complete, unambiguous spec that other agents can implement without further clarification.

## Inputs

- **Feature name:** {{ feature_name }}
- **Description:** {{ description }}
- **Target users:** {{ target_users }}

## Process

1. Ask clarifying questions if the description is ambiguous (one at a time)
2. Identify all components, composables, stores, and pages needed
3. Define acceptance criteria as testable statements
4. Document edge cases and error states
5. Write the spec to `specs/{{ feature_name }}.md`

## Spec Template

```markdown
# Spec: {{ Feature Name }}

## Status: draft | review | approved | in-progress | done

## Purpose
One paragraph explaining why this feature exists and what problem it solves.

## User Stories
- As a [role], I want to [action] so that [benefit]
- ...

## Acceptance Criteria
- [ ] AC-1: [Testable statement]
- [ ] AC-2: [Testable statement]
- ...

## Component Tree
```
pages/feature-name.vue
├── components/feature/header.vue
├── components/feature/list.vue
│   └── components/feature/list-item.vue
└── components/feature/form.vue
```

## Data Model

### API Endpoints Consumed
| Method | Endpoint | Request | Response |
|--------|----------|---------|----------|
| GET | /api/client/resource/all | query params | ResourceListResponse |
| POST | /api/client/resource/create | CreateResourceInput | Resource |

### TypeScript Interfaces
```typescript
interface Resource {
  id: number
  name: string
}
```

## State Management
- Store: `app/stores/feature.ts` (if needed)
- Key state: list of state variables and their purposes

## Edge Cases
- What happens when the API returns empty data?
- What happens on network failure?
- What happens with invalid user input?

## Non-Goals
- What this feature explicitly does NOT cover

## Dependencies
- External APIs required
- Composables to reuse
- Third-party packages needed
```

## Output

Save the completed spec to `specs/{{ feature_name }}.md` with status set to `draft`.
