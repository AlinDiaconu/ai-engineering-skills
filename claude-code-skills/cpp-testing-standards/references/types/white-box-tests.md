# White-box tests (C++)

## What

Tests that use **knowledge of the code structure** to choose cases and to measure how much of the code the tests reach (structural coverage).

## Why

- Finds code that **no test reaches**.
- Finds **dead code** and **missing requirements** (code that does something the spec does not say).
- Required by some coding and quality standards (statement, branch, MC/DC targets).
- Finds defects in special paths that black-box tests do not reach.

## When

- After the black-box tests are written. Use white-box analysis to find gaps.
- For code with a structural coverage target.
- For complex logic: many branches, loops, switches, state handlers.
- For error and defensive paths.

## How

1. Write and run the **black-box tests** first.
2. Measure **coverage** (`../practice/coverage-and-metrics.md`).
3. For each uncovered statement, branch, or condition, decide:
   - add a test that reaches it (still through the public interface when possible),
   - show that the code is dead (tell the user to remove it),
   - document why it cannot be reached (get approval).
4. For MC/DC: build the condition pairs table, then write the tests.
5. Check **each new test** has a meaningful assertion.

## Techniques

Coverage types, MC/DC, coverage targets, and tools: `../practice/coverage-and-metrics.md`. In short: statement, branch, condition, and MC/DC coverage, plus loop testing (below), path testing for small critical functions, and data-flow testing for complex data handling.

## Loop testing

| Case | Why |
|---|---|
| Zero iterations | Loop skipped |
| One iteration | Minimal |
| Two iterations | Detects state carried over |
| Typical count | Normal use |
| Max, max−1, max+1 | Boundary and overflow |

## MC/DC

Build the condition-pair table as in `../practice/coverage-and-metrics.md`. Keep it in the test file as a comment. Write one test per row.

## Accessing internals: the rule

- Reach internal paths **through the public interface** by setting the right inputs and fakes.
- If a path cannot be reached through the interface, ask:
  1. Is it dead code? → remove.
  2. Is the class doing too much? → extract a class, test it directly.
  3. Is it a defensive check for an impossible state? → document, get approval for an exclusion.
- Do **not** use `#define private public` or `friend` hacks.
- Do not change production code to reach a path without asking the user.

## C++ specific points

| Topic | Note |
|---|---|
| Short-circuit operators | `&&` and `||` are decisions. Count them for MC/DC. |
| `switch` | Test every `case` and the `default`. For enums, test every value. |
| Ternary `?:` | A decision. Test both sides. |
| Templates | Coverage is per instance. Test the instances that you use. |
| Exceptions | Each `catch` and each `throw` is a path. Exception paths may add compiler-made branches. State how you count them. |
| `constexpr` / compile-time code | Test with `static_assert` and also at run time. |
| Inline and header code | Check that coverage tools count the header lines. |
| Destructors and RAII | Cleanup runs on every exit path (normal, early return, exception). Test those exits. |
| Build for coverage | `-O0 --coverage`. Optimization changes the branches. |

## Best practices

- Coverage is a **search tool**, not a goal. Do not add empty tests to raise a number.
- Every white-box test has a **real assertion** on the result.
- Name tests by **behavior**, not by line numbers.
- Keep **traceability**: white-box tests still link to a requirement. If the path has no requirement, tell the user.
- Re-measure coverage **after each change** to the code.
- Combine with **mutation testing** for critical code.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Tests aim at 100% number only | Check assertions. Use mutation testing. |
| Tests tied to the structure (break on refactoring) | Call the public interface. Check behavior. |
| Unreachable code left with a "cannot test" note | Remove it, or get approval for an exclusion |
| Coverage measured with optimization | Rebuild with `-O0` |
| Test of private function through a hack | Extract a class |

## Checklist

- [ ] Black-box tests were written first.
- [ ] Coverage was measured with a named tool and type.
- [ ] Each gap: test added, code removed, or approved exclusion.
- [ ] MC/DC table exists where the project target needs it.
- [ ] No access hacks for private members.
