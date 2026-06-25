---
name: tdd
description: Test-driven development for embedded C/C++ (firmware, drivers, HALs, RTOS apps). Use when the user wants to build features or fix bugs test-first, mentions "red-green-refactor", wants unit/integration tests, needs to mock hardware dependencies, or wants test coverage for a module or driver. Supports Unity (default), CppUTest, and Google Test.
---

# Test-Driven Development

## Philosophy

**Core principle**: Tests should verify behavior through public interfaces, not implementation details. Code can change entirely; tests shouldn't.

**Good tests** are integration-style: they exercise real code paths through public APIs. They describe _what_ the system does, not _how_ it does it. A good test reads like a specification - "user can checkout with valid cart" tells you exactly what capability exists. These tests survive refactors because they don't care about internal structure.

**Bad tests** are coupled to implementation. They mock internal collaborators, test private methods, or verify through external means (like querying a database directly instead of using the interface). The warning sign: your test breaks when you refactor, but behavior hasn't changed. If you rename an internal function and tests fail, those tests were testing implementation, not behavior.

See [tests.md](tests.md) for examples and [mocking.md](mocking.md) for mocking guidelines.

## Embedded Mechanics

The discipline above is language-agnostic; these are the embedded C/C++ specifics.

**Framework selection** — default to **Unity**:

| Framework | Language | Best For |
|-----------|----------|----------|
| Unity | C | Resource-constrained targets, pure C codebases |
| CppUTest | C/C++ | Mixed C/C++, need CppUMock for mocking |
| Google Test | C++ | C++ codebases, complex mocking needs |

Setup, assertions, fixtures, and mocking per framework: [unity.md](unity.md),
[cpputest.md](cpputest.md), [googletest.md](googletest.md).

**Ensure testability first.** If the code under test pokes hardware directly (`PERIPHERAL->CTRL = ...`),
it can't run on the host. Before writing tests, refactor the hardware boundary behind an injected
interface (function-pointer struct, single injected function, or link-time substitution) so a mock
can stand in. Full patterns — register mocks, peripheral fakes, controllable time, interrupt/callback
simulation, weak symbols — are in [mocking.md](mocking.md).

**Cover embedded edge cases.** Beyond the behavior under test, exercise boundary values, empty/full
buffers, 32-bit timer rollover (~49.7 days), NULL pointers, invalid states/events, and error paths.
See [edge-cases.md](edge-cases.md).

**Use real referenced facts.** When a test or mock encodes a real register address, bitfield, reset
value, electrical/timing constant, or protocol field, source it via `/reference-lookup` and cite it
— don't hand-write it from memory, or the test will faithfully verify the wrong value.

## Anti-Pattern: Horizontal Slices

**DO NOT write all tests first, then all implementation.** This is "horizontal slicing" - treating RED as "write all tests" and GREEN as "write all code."

This produces **crap tests**:

- Tests written in bulk test _imagined_ behavior, not _actual_ behavior
- You end up testing the _shape_ of things (data structures, function signatures) rather than user-facing behavior
- Tests become insensitive to real changes - they pass when behavior breaks, fail when behavior is fine
- You outrun your headlights, committing to test structure before understanding the implementation

**Correct approach**: Vertical slices via tracer bullets. One test → one implementation → repeat. Each test responds to what you learned from the previous cycle. Because you just wrote the code, you know exactly what behavior matters and how to verify it.

```
WRONG (horizontal):
  RED:   test1, test2, test3, test4, test5
  GREEN: impl1, impl2, impl3, impl4, impl5

RIGHT (vertical):
  RED→GREEN: test1→impl1
  RED→GREEN: test2→impl2
  RED→GREEN: test3→impl3
  ...
```

## Workflow

### 1. Planning

When exploring the codebase, read `CONTEXT.md` (if it exists) so that test names and interface vocabulary match the project's domain language, and respect ADRs in the area you're touching.

Before writing any code:

- [ ] Confirm with user what interface changes are needed
- [ ] Confirm with user which behaviors to test (prioritize)
- [ ] Identify opportunities for deep modules (small interface, deep implementation) — run the `/codebase-design` skill for the vocabulary and the testability checks
- [ ] Ensure the hardware boundary is injectable — if the code accesses registers/peripherals directly, refactor for dependency injection first (see [mocking.md](mocking.md))
- [ ] List the behaviors to test (not implementation steps)
- [ ] Get user approval on the plan

Ask: "What should the public interface look like? Which behaviors are most important to test?"

**You can't test everything.** Confirm with the user exactly which behaviors matter most. Focus testing effort on critical paths and complex logic, not every possible edge case.

### 2. Tracer Bullet

Write ONE test that confirms ONE thing about the system:

```
RED:   Write test for first behavior → test fails
GREEN: Write minimal code to pass → test passes
```

This is your tracer bullet - proves the path works end-to-end.

### 3. Incremental Loop

For each remaining behavior:

```
RED:   Write next test → fails
GREEN: Minimal code to pass → passes
```

Rules:

- One test at a time
- Only enough code to pass current test
- Don't anticipate future tests
- Keep tests focused on observable behavior

### 4. Refactor

After all tests pass, look for [refactor candidates](refactoring.md):

- [ ] Extract duplication
- [ ] Deepen modules (move complexity behind simple interfaces)
- [ ] Apply SOLID principles where natural
- [ ] Consider what new code reveals about existing code
- [ ] Run tests after each refactor step

**Never refactor while RED.** Get to GREEN first.

## Checklist Per Cycle

```
[ ] Test describes behavior, not implementation
[ ] Test uses public interface only
[ ] Test would survive internal refactor
[ ] Code is minimal for this test
[ ] No speculative features added
```
