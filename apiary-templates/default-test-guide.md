# Default Test Guide

This is a default, language-agnostic guide used by the Apiary `test-writer`
subagent and `test-review` skill. It covers both how to write good tests and
what to flag during review. It is intentionally opinionated. Edit it or swap
it for your own document; whatever `apiary.md` `## Documentation Locations` →
`Test guide` points at is what the test-related skills will apply.

---

## Writing Tests

### Scope of a test

- A unit test exercises a single behavior of a single unit. If you
  cannot summarize what it tests in one sentence, split it.
- An integration test exercises a real interaction between
  components. State that explicitly (in the test's name or location)
  so readers know it is not a pure unit test.

### Naming

- The test name describes the behavior under test, not the
  implementation. `describes_what_user_sees`, not
  `calls_method_x_with_y`.
- Reading the names of all tests in a module should give you a clean
  outline of the unit's contract.

### Structure

- Arrange / Act / Assert (or Given / When / Then). Keep the three
  phases visually separable.
- One logical assertion per test. Multiple assertion lines are fine
  if they all describe the same behavior; multiple unrelated
  assertions usually mean you are testing two things in one test.
- Setup that is shared across many tests belongs in fixtures /
  helpers / `beforeEach` — not copy-pasted.

### Isolation

- No shared mutable state between tests. Each test must be
  independently runnable and order-independent.
- No real network, filesystem, clock, or random source in unit
  tests. Mock or inject. Integration tests that intentionally hit
  real dependencies should be marked and runnable in a separate
  pass.
- No `sleep` to wait for things — use deterministic synchronization
  or a controllable fake clock.

### Coverage

- Cover the happy path, the obvious edge cases (empty, null /
  none, boundary, max / min), and at least one failure mode.
- New or changed behavior in source code should arrive with a test
  that would fail without the change. If you cannot write one, say
  why in the PR / Task notes.

### Parameterization over duplication

- If three tests differ only by inputs and expected outputs,
  collapse them into one parameterized test. Use the project's
  parameterization mechanism (table-driven, parametrize, `each`,
  etc.).
- Parameterized cases should still have descriptive names so failure
  output points at the failing case.

### Anti-patterns to avoid

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

---

## Reviewing Tests

### 1. Coverage Gaps

- New functions or behaviors in the changed source with no
  corresponding test
- Missing edge cases: empty inputs, null / none, boundary values,
  max / min
- Missing error / exception paths — only the happy path is
  exercised
- Missing negative tests — things that *should* fail are not
  asserted to fail

### 2. Test Correctness

- Tests that assert the wrong thing (incorrect expectations)
- Tests that would still pass if the code under test were broken
  (vacuous tests)
- Tests that check internal implementation details rather than
  observable behavior
- Mocks / stubs that do not faithfully reflect the real dependency
- Loose assertions (e.g., asserting "not null" when the meaningful
  check is the value)

### 3. Test Structure & Quality

- A single test that exercises multiple behaviors — should be split
- Poor test names — unclear what behavior is being verified
- Missing or incorrect setup / teardown
- Shared mutable state between tests (a common cause of flakiness)
- Tests that depend on execution order

### 4. Bloat & Redundancy (HIGH PRIORITY)

This is one of the most common and damaging problems in test suites.
Be aggressive here.

- **Parameterization opportunities:** multiple test functions that
  run the same logic with different inputs should be collapsed into
  a single parameterized test. Even two or three near-identical
  tests are a candidate.
- **Duplicate coverage:** tests that assert behavior already covered
  elsewhere — find and flag them by name and line.
- **Copy-paste tests:** blocks of nearly identical setup or
  assertion code repeated across tests — extract a fixture or
  helper.
- **Over-specified tests:** tests that re-assert things already
  covered by other tests; trim to what is unique.

When flagging any of the above, always cite the specific test names
and line numbers so the fix is unambiguous.

### 5. Stale / Obsolete Tests

- Old tests that are no longer valid given current code
- Tests that exercise code paths that no longer exist
- Tests left behind around a feature that has been removed

### 6. Test Anti-patterns

- No assertions — the test always passes
- Catch-all blocks that swallow failures silently
- Hardcoded sleeps / delays — use deterministic synchronization or
  fake clocks
- External I/O (network, filesystem) in unit tests without mocking
- Tests too tightly coupled to internal implementation

### Prioritization

- **Include:** missing coverage for new code, incorrect assertions,
  flaky patterns, stale tests, large parameterization opportunities.
- **Exclude:** minor style issues, naming nitpicks, personal
  preferences, anything a formatter already handles.

It is fine — and expected — for a review to return zero items. Do
not invent issues.
