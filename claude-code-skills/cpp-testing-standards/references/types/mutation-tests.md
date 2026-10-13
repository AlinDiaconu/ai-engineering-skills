# Mutation tests (C++)

## What

A tool makes **small changes** (mutants) to the code, for example `<` becomes `<=`, `+` becomes `-`, a condition is negated, a statement is removed. It runs the tests for each mutant. If a test fails, the mutant is **killed**. If all tests pass, the mutant **survives**.

## Why

- Coverage shows that code **ran**. Mutation testing shows that the tests **would notice a defect**.
- A surviving mutant points to a **missing or weak assertion**, or to **dead code**.

## When

- Critical code (decision logic, limits, state machines, parsers) that already has high coverage.
- After you write tests for a critical module, to check their strength.
- On changed files in a pull request (incremental).
- On a schedule for the whole critical code.

Do not run on the whole code base at each commit. It is slow.

## How

1. Reach a **good coverage level** and a **green** test suite first.
2. Run the mutation tool on **one module**.
3. Read the **surviving mutants**.
4. For each survivor, decide:
   - **Weak test:** add or strengthen an assertion. This is the usual case.
   - **Missing test:** add a case (often a boundary).
   - **Dead or redundant code:** tell the user to remove it.
   - **Equivalent mutant:** the change does not change the behavior. Mark it as ignored with a reason.
5. Re-run. Track the **mutation score**.

Mutation score = killed / (total − equivalent).

## Common mutations

| Mutation | Example | A killing test needs |
|---|---|---|
| Relational operator | `a < b` → `a <= b` | A boundary case |
| Arithmetic operator | `a + b` → `a - b` | A check of the value |
| Logical operator | `&&` → `||` | Cases for each condition |
| Negate condition | `if (x)` → `if (!x)` | Both outcomes with checks |
| Remove statement / call | Delete a call | A check of its effect |
| Constant change | `0` → `1`, `n` → `n+1` | Exact value checks |
| Return value change | `return true` → `return false` | A check of the result |
| Boundary | `>` → `>=` | A test at the exact threshold |

## Tools for C++

| Tool | Notes |
|---|---|
| **Mull** | LLVM-based. Mutates at compiler IR level. Fast. Works with GoogleTest. Needs Clang. |
| **dextool mutate** | Mutation testing for C and C++. Supports coverage-guided reduction. |
| **Manual mutation** | Small, hand-made changes in a scratch copy. For a quick check of a few tests. |

Check the tool with the toolchain of the project. Ask the user before you add a tool.

### Mull example (idea)

```bash
clang++ -fpass-plugin=/path/to/mull-ir-frontend -g -grecord-command-line \
        session_state_machine.cpp session_state_machine_test.cpp -lgtest -lgtest_main -o test_mull
mull-runner-<ver> ./test_mull
```

(Use the commands of your Mull version and the project build system.)

## Best practices

- **Only mutate production code**, not test code.
- **Limit the scope** (one module, or changed lines) so the run is short.
- **Set a target** for critical modules (for example above 80% to 90%), and agree it with the team. Do not chase 100%.
- **Fix weak assertions first.** Most survivors come from tests that call the code but check little.
- **Handle equivalent mutants** by marking them with a reason. Do not add useless tests.
- **Use fast tests.** Mutation runs the suite many times. Use the unit tests only.
- **Use coverage to skip** mutants in code that no test reaches (they survive for a trivial reason).
- **Set a timeout** for mutants that cause infinite loops. A timeout counts as killed in most tools.
- **Run in CI on a schedule**, or on the diff of a pull request.
- **Report**: tool, scope, score, and the list of important survivors with your decision.

## Manual mutation check (quick)

For one new test, prove it can fail:

1. Make a **scratch copy** or use `git stash`. Never leave the change in the code.
2. Change one thing in the production code (flip a comparison, remove a line).
3. Run the test. It **must fail**.
4. Undo the change.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Score chased as a goal | Look at surviving mutants. Add meaning, not noise. |
| Too slow, so never run | Narrow the scope. Run on a schedule. |
| Many equivalent mutants | Mark with reasons. Use a tool that filters them. |
| Tests use real time or I/O, mutation run is flaky | Use deterministic unit tests |
| Mutants survive in unreachable code | Remove the code, or document |

## Checklist

- [ ] The suite is green and has good coverage first.
- [ ] Scope is limited. Time is acceptable.
- [ ] Each survivor has a decision: strengthen, add, remove, or ignore with a reason.
- [ ] Score and tool are reported.
