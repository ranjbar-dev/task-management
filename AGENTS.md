# Agents — Nuxt 4 Project

## Agent Roster

| Agent | Role | Scope | Writes Code? |
|-------|------|-------|-------------|
| **Orchestrator** | Plans, delegates, validates | Specs, task breakdown, review | No |
| **Frontend** | UI implementation | Components, pages, layouts, styling | Yes |
| **API** | Data layer | Server routes, database schemas, services, transformers, composables, types | Yes |
| **Testing** | Quality assurance | Unit tests, e2e tests, coverage | Yes |

## Workflow: Spec-Driven Development (SDD)

```
Developer Request
       │
       ▼
┌─────────────┐     No spec found
│ Orchestrator │──────────────────► Create spec (write-spec prompt)
│  reads spec  │                           │
└──────┬───────┘                           ▼
       │                          Spec created in specs/
       │ Spec exists                       │
       ▼                           ◄───────┘
┌─────────────┐
│ Orchestrator │
│ breaks into  │
│   tasks      │
└──────┬───────┘
       │
       ├──► Frontend Agent (UI tasks)
       │         │
       ├──► API Agent (data layer tasks)
       │         │
       └──► Testing Agent (after implementation)
                 │
                 ▼
       Orchestrator validates
       completeness against spec
```

## Rules of Engagement

### 1. Spec-First Gate (Hard Rule)
- No agent writes code without a spec in `specs/`
- Exception: `fix-bug` tasks skip the spec gate
- Every new file must reference its spec: `// Spec: specs/<name>.md`

### 2. Orchestrator Authority
- Orchestrator is the single entry point for all feature work
- Orchestrator assigns tasks — agents do not self-assign
- Orchestrator validates completeness: all spec acceptance criteria must be covered

### 3. Agent Boundaries
- **Frontend** does not write API composables, stores, or server-side code
- **API** does not write components or pages
- **Testing** does not modify implementation code — only test files
- If a task crosses boundaries, Orchestrator splits it into sub-tasks

### 4. Communication Protocol
- Each agent reads the spec before starting work
- Each agent documents decisions as code comments when deviating from the spec
- When an agent encounters ambiguity, it stops and asks rather than assumes

### 5. Completion Criteria
A task is complete when:
- All acceptance criteria from the spec are implemented
- TypeScript compiles with zero errors
- ESLint + Prettier pass with zero errors
- Tests cover happy path + at least one error case
- Orchestrator has reviewed and approved

## Agent Definitions

Detailed agent instructions are in `.github/agents/`:
- [Orchestrator](.github/agents/orchestrator.md)
- [Frontend](.github/agents/frontend.md)
- [API](.github/agents/api.md)
- [Testing](.github/agents/testing.md)
