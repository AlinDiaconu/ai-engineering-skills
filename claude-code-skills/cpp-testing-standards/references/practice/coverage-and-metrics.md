# Coverage and metrics (C++)

This file is the single owner of coverage theory, coverage targets, MC/DC, and test metrics.

## What coverage tells you

- It tells you what is **not tested**. It does **not** tell you that the tests are good. A test with no assertion gives coverage and no value.
- High coverage with weak asserts gives false confidence. Add mutation testing (`../types/mutation-tests.md`) and assertion review (`review.md`).

## Coverage types

| Type | Meaning | Strength |
|---|---|---|
| Function | Each function is called | Weakest |
| Line / statement | Each statement runs | Basic |
| Branch / decision | Each decision outcome is taken | Good minimum for logic |
| Condition | Each condition is true and false | Finer |
| **MC/DC** | Each condition independently changes the decision | Strictest practical level |
| Path | Every path | Too many for most code |
| Requirement | Each requirement has a test | Completeness |
| State / transition | Each state and transition is tested | State machines |
| Input / data | Equivalence classes and boundaries are tested | Input handling |

## Targets

Use the **project target**. If the project has none, use these as a start:

| Code | Start with |
|---|---|
| General application code | Statement and branch coverage of changed code |
| Parsers, state machines, decision logic | Branch coverage, plus mutation testing |
| Code under a strict standard or contract | The level that the standard or contract names (it may require MC/DC) |

- Set a **floor** as a guard against decline. Do not reward 100%.
- Measure at the level that the project uses (unit level is common).
- If a standard requires a qualified tool, ask which tool is approved.

## Measuring

| Tool | Notes |
|---|---|
| `gcov` / `gcovr` / `lcov` (GCC) | Line and branch. Build with `--coverage -O0`. |
| `llvm-cov` (Clang) | Line, branch, region. MC/DC with recent Clang (`-fcoverage-mcdc`). |
| Commercial tools (VectorCAST, Bullseye, LDRA, Testwell CTC++, Cantata) | When a standard requires a qualified tool |

- Build with **no optimization** (`-O0`), so the numbers match the source.
- Exclude test code, mocks, and third-party code. State the exclusions.
- Check that header, template, and `constexpr` code is counted.

### Hidden branches in C++

- `&&`, `||`, and `?:` are decisions.
- Exception paths create compiler-made branches. Exclude them only with approval, and state it.
- Range-`for`, `std::visit`, and templates create generated branches. Do not chase them blindly. Check what is real code.
- Impossible `default:` in a `switch` over an enum: document it. Get approval for an exclusion.

## MC/DC

For decision `A && (B || C)`, show for each condition a **pair** of tests where only that condition changes and the decision output changes. Use n + 1 tests for n independent conditions.

| Test | A | B | C | Result |
|---|---|---|---|---|
| 1 | T | T | F | T |
| 2 | F | T | F | F |
| 3 | T | F | F | F |
| 4 | T | F | T | T |

Pair 1–2 shows A. Pair 1–3 shows B. Pair 3–4 shows C. Keep the table in the test file or the report. Keep decisions small: split many conditions into named booleans. For the white-box workflow, see `../types/white-box-tests.md`.

## Uncovered code

For each uncovered line or branch, do **one** of these:

1. Add a test.
2. Show that the code is dead, and tell the user to remove it.
3. Document why it cannot be tested, and get the user's approval.

Do not add tests only to raise the number.

## Good metrics

| Metric | Use |
|---|---|
| Branch coverage of changed code | Pull-request gate |
| Mutation score on critical modules | Test strength |
| Requirement → test traceability | Completeness |
| Escaped defects | Real effectiveness |
| Flaky-test rate | Trust |
| Suite duration per stage | Feedback speed |
| Sanitizer findings per week | Memory and race health |
| Time to fix a red build | Discipline |
| Fuzz coverage and crash count | Parser robustness |
| Timing margin to deadline (maximum observed) | Real-time health |

## Bad metrics (avoid)

Number of tests, 100% line coverage as a goal, tests per developer, pass rate alone, defects found per tester. They reward numbers, not quality (Goodhart's law).

## Traceability

- Link **requirement → test → result → program version**.
- Put the requirement ID in a comment above each test. Do not invent IDs. If a requirement has no ID, tell the user.
- Check both ways: every requirement has a test, every test has a reason.

## Reporting

State the **tool**, the **type**, the **scope**, the **level**, and the **program version**. Give measured numbers, not estimates. List the important gaps with a reason for each.
