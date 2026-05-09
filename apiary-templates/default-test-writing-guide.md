# Default Test Writing Guide

This is a default, language-agnostic guide for writing unit and
integration tests in an Apiary project. It is consumed by the
`test-writer` subagent (and `test-review` when no project-specific
guide is configured). Edit it or swap it for your project's own
document; whatever `apiary.md` `## Documentation Locations` →
`Test writing guide` points at is what the test-related skills will
apply.

## Scope of a test

- A unit test exercises a single behavior of a single unit. If you
  cannot summarize what it tests in one sentence, split it.
- An integration test exercises a real interaction between
  components. State that explicitly (in the test's name or location)
  so readers know it is not a pure unit test.

## Naming

- The test name describes the behavior under test, not the
  implementation. `describes_what_user_sees`, not
  `calls_method_x_with_y`.
- Reading the names of all tests in a module should give you a clean
  outline of the unit's contract.

## Structure

- Arrange / Act / Assert (or Given / When / Then). Keep the three
  phases visually separable.
- One logical assertion per test. Multiple assertion lines are fine
  if they all describe the same behavior; multiple unrelated
  assertions usually mean you are testing two things in one test.
- Setup that is shared across many tests belongs in fixtures /
  helpers / `beforeEach` — not copy-pasted.

## Isolation

- No shared mutable state between tests. Each test must be
  independently runnable and order-independent.
- No real network, filesystem, clock, or random source in unit
  tests. Mock or inject. Integration tests that intentionally hit
  real dependencies should be marked and runnable in a separate
  pass.
- No `sleep` to wait for things — use deterministic synchronization
  or a controllable fake clock.

## Coverage

- Cover the happy path, the obvious edge cases (empty, null /
  none, boundary, max / min), and at least one failure mode.
- New or changed behavior in source code should arrive with a test
  that would fail without the change. If you cannot write one, say
  why in the PR / Task notes.

## Parameterization over duplication

- If three tests differ only by inputs and expected outputs,
  collapse them into one parameterized test. Use the project's
  parameterization mechanism (table-driven, parametrize, `each`,
  etc.).
- Parameterized cases should still have descriptive names so failure
  output points at the failing case.

## Anti-patterns to avoid

- Tests with no assertions (always pass).
- `try` / `except` / `catch` blocks that swallow failures.
- Asserting on internal implementation details (private methods,
  call counts of internal helpers) instead of on observable
  behavior.
- Mocks that drift from the real dependency — pin them to the same
  contract the real thing exposes.
- Vacuous tests that would pass even if the code were broken
  (e.g., asserting `result is not None` when the meaningful check
  is the value).
- Tests that depend on execution order, environment, or global
  state.
