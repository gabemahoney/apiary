# Default Engineering Best Practices

This is a default, language-agnostic checklist of engineering best
practices used by the Apiary `code-review` skill (and any human or
agent doing review work). It is intentionally opinionated. Edit it or
swap it for your own project-specific document; whatever
`apiary.md` `## Documentation Locations` → `Engineering best practices`
points at is what the review skills will apply.

The checklist is organized by category. For each category, flag any
deviation in the changed code as a review work item.

## 1. Dead / Obsolete Code

- Commented-out code that should be removed
- Unused functions, variables, imports, or parameters
- Old implementations left behind alongside their replacements
- Debugging artifacts (print statements, `console.log`, scratch
  TODOs, breakpoints)

## 2. Architecture & Design

- Inconsistent interfaces — does this match existing patterns in the
  same codebase?
- Inappropriate mixing of concerns — business logic, transport / API,
  and data access should be separable
- Unnecessary abstractions (YAGNI — You Aren't Gonna Need It); three
  similar lines beats a premature abstraction
- Inconsistent patterns relative to the rest of the codebase
- Half-finished implementations or stubs that are not flagged as such

## 3. Security & Correctness (CRITICAL)

- **Input validation:** every external input (user, network, file,
  message) is validated at the system boundary
- **Database queries:** parameterized queries only; never assemble
  queries by string interpolation of untrusted values
- **File paths:** validated against an expected root; do not trust
  caller-supplied paths
- **Secrets:** loaded from environment / config / secret manager;
  never hardcoded; never logged
- **Authentication / authorization:** every protected entry point
  performs the required checks
- **Error responses:** no sensitive data (stack traces, internal
  paths, secrets) leaked to callers
- **Concurrency:** shared mutable state is protected; race conditions
  are considered

## 4. Code Quality

- Long / complex functions (rough heuristic: >50 lines, deep nesting
  >3 levels) — extract helpers
- Repeated code blocks (DRY violations)
- Magic numbers / strings — should be named constants
- Poor variable / function names — name should describe purpose, not
  type
- Missing comments where the *why* is non-obvious (hidden
  constraints, subtle invariants); avoid comments that just restate
  the *what*

## 5. Error Handling

- Catch-all / bare exception clauses that swallow specific failures
- Resources not properly cleaned up — files, connections, locks must
  use the language's RAII / context-manager / `defer` equivalent
- Missing error handling in critical paths (writes, transactions,
  external calls)
- Poor error messages — must be actionable for the caller, not just
  "something went wrong"
- Errors silently dropped to keep the happy path green

## 6. Performance

- N+1 query patterns — database / RPC calls inside loops that should
  be batched or joined
- Loading entire large files / datasets into memory when streaming is
  possible
- Missing connection pooling or reusing of expensive clients
- Synchronous / blocking I/O inside async / event-loop code
- Missing or incorrect cache invalidation

## Prioritization

Focus on substance over style.

- **Include in review output:** security vulnerabilities, logic
  errors, missing tests, architecture problems, dead code, broken
  error handling.
- **Exclude:** trivial style issues, minor naming nitpicks, personal
  preferences, anything a formatter / linter already enforces.

It is fine — and expected — for a review to return zero items. Do
not invent issues.
