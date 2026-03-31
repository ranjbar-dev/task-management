# Fix Bug

Debug and fix a bug with regression test coverage.

## Context

You are the Testing agent (analysis) + relevant implementation agent (fix). This is the **one exception** to the spec-first gate — bugs can be fixed without a spec.

## Inputs

- **Bug description:** {{ description }}
- **Steps to reproduce:** {{ steps }}
- **Expected behavior:** {{ expected }}
- **Actual behavior:** {{ actual }}
- **Affected files:** {{ files }} (if known)

## Process

### 1. Reproduce & Analyze
- Trace the bug from the described symptoms to the root cause
- Identify the minimal set of files involved
- Document the root cause in a comment

### 2. Write Regression Test FIRST
Before fixing anything, write a test that fails due to the bug:

```typescript
// tests/unit/...or.../tests/e2e/...
it('should [expected behavior] (regression: [bug summary])', () => {
  // This test should FAIL before the fix and PASS after
})
```

### 3. Implement the Fix
- Make the minimal change to fix the root cause
- Do not refactor unrelated code in the same change
- Add a comment at the fix site: `// Fix: [brief description of what was wrong]`

### 4. Verify
- Confirm the regression test now passes
- Run existing tests to ensure no regressions
- Check TypeScript compilation
- Check ESLint + Prettier

## Output

```
[modified source files]
[new or modified test files]
```

## Bug Fix Comment Template

At the fix site in the source code:

```typescript
// Fix: [one-line description]
// Root cause: [what was wrong]
// Regression test: tests/unit/path/to/test.test.ts
```

## Quality Checks

- [ ] Root cause identified — not just symptoms treated
- [ ] Regression test written BEFORE the fix
- [ ] Regression test fails without fix, passes with fix
- [ ] Minimal change — no drive-by refactoring
- [ ] Existing tests still pass
- [ ] TypeScript compiles with zero errors
- [ ] ESLint + Prettier pass
