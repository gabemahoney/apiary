# Default Test Review Guide

This is a default, language-agnostic checklist used by the Apiary
`test-review` skill (and reused by the `test-writer` subagent for
self-review). It is intentionally opinionated. Edit it or swap it
for your own document; whatever `apiary.md`
`## Documentation Locations` → `Test review guide` points at is what
`test-review` will apply.

## 1. Coverage Gaps

- New functions or behaviors in the changed source with no
  corresponding test
- Missing edge cases: empty inputs, null / none, boundary values,
  max / min
- Missing error / exception paths — only the happy path is
  exercised
- Missing negative tests — things that *should* fail are not
  asserted to fail

## 2. Test Correctness

- Tests that assert the wrong thing (incorrect expectations)
- Tests that would still pass if the code under test were broken
  (vacuous tests)
- Tests that check internal implementation details rather than
  observable behavior
- Mocks / stubs that do not faithfully reflect the real dependency
- Loose assertions (e.g., asserting "not null" when the meaningful
  check is the value)

## 3. Test Structure & Quality

- A single test that exercises multiple behaviors — should be split
- Poor test names — unclear what behavior is being verified
- Missing or incorrect setup / teardown
- Shared mutable state between tests (a common cause of flakiness)
- Tests that depend on execution order

## 4. Bloat & Redundancy (HIGH PRIORITY)

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

## 5. Stale / Obsolete Tests

- Old tests that are no longer valid given current code
- Tests that exercise code paths that no longer exist
- Tests left behind around a feature that has been removed

## 6. Test Anti-patterns

- No assertions — the test always passes
- Catch-all blocks that swallow failures silently
- Hardcoded sleeps / delays — use deterministic synchronization or
  fake clocks
- External I/O (network, filesystem) in unit tests without mocking
- Tests too tightly coupled to internal implementation

## Prioritization

- **Include:** missing coverage for new code, incorrect assertions,
  flaky patterns, stale tests, large parameterization opportunities.
- **Exclude:** minor style issues, naming nitpicks, personal
  preferences, anything a formatter already handles.

It is fine — and expected — for a review to return zero items. Do
not invent issues.
