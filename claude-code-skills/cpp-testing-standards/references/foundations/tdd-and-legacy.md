# TDD, BDD, and legacy code (C++)

## TDD (test-driven development)

**Cycle: Red, Green, Refactor.**

1. **Red:** write one small test for the next behavior. Build and run. It **must fail**, for the right reason. (A compile error is a valid first red in C++: the interface does not exist yet.)
2. **Green:** write the **simplest** code that passes.
3. **Refactor:** clean the code and the test. Keep all tests green.
4. Repeat in small steps (minutes).

Why: tests exist for all code, the design stays simple and testable, feedback is fast.

When it helps: logic, algorithms, new classes, bug fixes, clear requirements.
When it is hard: unclear requirements (make a throwaway spike, then rewrite with tests), heavy hardware dependence.

Rules:

- No production code without a failing test (when you use TDD).
- Write only enough test to fail. Write only enough code to pass.
- Do not skip the refactor step.
- See the test fail **before** you make it pass.

### TDD in C++: practical points

- Keep the **build fast**: small test executables per module, header hygiene, forward declarations, precompiled headers, `ccache`. Slow builds kill the TDD rhythm.
- Use a **watch loop** (`ctest --rerun-failed --output-on-failure`, or an IDE test runner).
- Start with the **interface** in the test. The test shows how a user calls the class.
- Use `constexpr` and `static_assert` for the first red when the behavior is a compile-time property.
- Keep each cycle small enough to finish in a few minutes.

### Two styles

- **Inside-out (classic):** start with the units, build up. Use fakes for the edge.
- **Outside-in:** start with a component or acceptance test. Drive inner units with doubles at the boundaries.

## BDD (behavior-driven) in C++

- Describe behavior as **Given / When / Then**. You can do this in plain GoogleTest with comments and good names:

```cpp
TEST_F(SessionTest, GivenConnected_WhenHeartbeatLost_ThenStartsReconnect)
{
    // Given
    machine_.StartConnected();
    // When
    LoseHeartbeat();
    // Then
    EXPECT_EQ(machine_.State(), SessionState::Reconnecting);
}
```

- Frameworks: Catch2 has `SCENARIO`, `GIVEN`, `WHEN`, `THEN` macros. GoogleTest has no BDD macros. Cucumber-cpp exists for Gherkin files.
- Use Gherkin only when non-programmers read the scenarios. It adds cost.
- Keep scenarios about **business rules**, not about function calls.

## ATDD and specification by example

- Agree **acceptance criteria** with the customer or the system engineer before coding.
- Make them automated tests (scenario or component tests).
- The feature is done when these tests pass.

## Legacy code with no tests

Goal: change code safely.

1. **Find the seam** (a place to change behavior without editing the code).
2. **Break the dependency** with the smallest, safest change. Ask the user before you change production code.
3. Write **characterization tests**: call the code, record what it does now, assert that.
4. Make the change. Keep the tests green.
5. Add real tests for the new behavior.
6. Refactor in small steps.

### C++ seams

| Seam | How | Cost |
|---|---|---|
| **Object seam** | Extract an interface. Inject it. Subclass in tests. | A virtual call. Small code change. |
| **Template seam** | Make the dependency a template parameter. Use a fake type in tests. | Code moves to the header. |
| **Link seam** | Link the test with a fake implementation of the same symbols (stub library) | No code change. Build work. Works for C APIs. |
| **Preprocessor seam** | `#ifdef UNIT_TEST` replaces a call | Last resort. Hard to maintain. Ask first. |
| **Parameter seam** | Pass the dependency as a function argument | Changes the signature |
| **Wrapper (adapter)** | Wrap a global function or a third-party API in a thin class | Small new class |

Common "break the dependency" moves:

- **Extract and override:** move the hard-to-test call to a protected virtual method. Override it in a test subclass.
- **Parameterize constructor:** add a constructor that takes the dependency. Keep the old one as a delegate.
- **Extract interface:** make a pure virtual class for the role.
- **Wrap global access:** replace direct use of a singleton by an injected reference.
- **Sprout method / sprout class:** write new logic in a new, tested function or class. Call it from the old code with a minimal edit.

Notes:

- Characterization tests record **actual** behavior, even if it looks wrong. Mark suspected bugs. Tell the user.
- Do not rewrite a large area without tests around it.
- Start where code changes most and breaks most.
- Golden-master tests cover large output quickly (`../types/golden-and-snapshot-tests.md`).
- See `../types/regression-and-characterization-tests.md`.

## Bug-fix workflow

Test first, then fix. Steps and examples: `../types/regression-and-characterization-tests.md`.

## Refactoring with tests

- Tests must be green **before** you start.
- Change structure only. Do not change behavior.
- Run the tests after each small step.
- If a test breaks, you changed behavior, or the test is too coupled. Find out which.

## Test-first vs test-after

- Test-first gives the best design feedback.
- Test-after is acceptable when the tests come **soon** and with the **same care**.
- Avoid "tests later". They often never come.

## Definition of done (test view)

- New behavior has tests at the right level and type.
- All tests pass in CI, shuffled, in the build matrix.
- Sanitizer builds are clean.
- Changed code meets the coverage target of the project.
- No new flaky, skipped, or `DISABLED_` tests.
- Fixed bugs have regression tests.
- Known gaps are written down.
