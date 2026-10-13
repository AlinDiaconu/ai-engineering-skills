# Flaky and failing tests (C++)

## Failing test: first steps

1. Read the **full** failure output. Note the file, line, expected value, and actual value.
2. Run the single test: `<binary> --gtest_filter=Suite.Test`.
3. Decide: is the **test** wrong, or is the **production code** wrong?
   - Check the requirement. The requirement decides, not the current code.
4. If the production code is wrong, **report it**. Do not change the test to hide it.
5. If the test is wrong, fix the test. Say why it was wrong.
6. Never delete or weaken an assertion to make a test pass without telling the user.

## Flaky test: process

1. Run the test 100 times:
   `<binary> --gtest_filter=Suite.Test --gtest_repeat=100 --gtest_break_on_failure`
2. Run the full suite shuffled with the failing seed:
   `<binary> --gtest_shuffle --gtest_random_seed=<seed>`
3. Run it **alone** and **with the suite**. If the result changes, look for shared or static state.
4. Run it **in parallel** and under **CPU load** (for example `stress-ng`). If it fails, look for timing and races.
5. Run it under **ASan**, **UBSan**, **TSan**, and **MSan** (separate builds).
6. Check for: time, random, thread, uninitialized data, container order, pointer values, environment.
7. Fix the **cause**.

Do not:

- add blind retries,
- make waits longer,
- disable the test without an issue and an owner.

## C++ specific causes

| Cause | Sign | Fix |
|---|---|---|
| Uninitialized variable | Fails in some builds or runs | Initialize. Use MSan or Valgrind. |
| Dangling reference or use-after-free | Random crash or wrong value | ASan. Fix the lifetime. |
| Data race | Rare wrong value or crash | TSan. Use proper synchronization, or run single-threaded. |
| Static initialization order | Fails at start | Use function-local statics. Remove global objects. |
| Static or global state from an earlier test | Order-dependent | Reset in `SetUp` and `TearDown`. |
| `unordered_*` iteration order | Different order per run | Sort. Use `UnorderedElementsAre`. |
| Floating-point differences | Last digits differ per platform or flags | Use a tolerance. |
| Signed overflow, UB | Optimizer changes the result | UBSan. Fix the arithmetic. |
| Time or `sleep` | Slow machine fails | Fake clock. |
| Pointer value in a comparison, or hash of pointer | Differs per run | Compare values, not addresses. |
| Port or file already in use | Fails when tests run in parallel | Use port 0, unique temp paths. |
| Debug vs release difference | Passes in Debug only | Look for UB and uninitialized data. Test both build types in CI. |

## Debugging tools

| Tool | Use |
|---|---|
| `--gtest_break_on_failure` + `gdb` / `lldb` | Stop at the failing assertion |
| `SCOPED_TRACE` | Add context to failures in helpers and loops |
| Print the seed and inputs | Reproduce random failures |
| `rr` (record and replay) | Replay a rare failure again and again |
| Core dump | Post-mortem of a crash |
| `ctest --rerun-failed --output-on-failure` | Re-run only failed tests |
| Sanitizer reports | Memory, UB, and race causes |

## Quarantine rules

If you must quarantine a flaky test:

- Move it to a separate, visible suite or label it (for example `FLAKY_`).
- Open an issue with an owner and a date.
- Tell the user.
- It is not a solution. Fix the cause.

## When the test shows a real defect

- Keep the test as it is. It is correct.
- Report: the requirement, the test, the actual result, and your analysis of the cause.
- Do not fix production code without the user's approval.

## Slow tests

| Cause | Fix |
|---|---|
| Real I/O in unit tests | Fakes, in-memory implementations |
| Heavy setup per test | Share **immutable** setup. Build only what you need. |
| Too many system tests | Move checks to lower levels |
| Sequential run | `ctest -j`. Independent tests only. |
| `sleep` | Fake clock, event wait |
| Large data | The smallest data that shows the behavior |
| Slow build | `ccache`, precompiled headers, fewer includes, one test executable per module |

Measure the slowest tests regularly. Keep the fast suite fast.

## Test debt

Signs: many skipped, `DISABLED_`, or commented-out tests. Red builds re-run until green. Tests that nobody understands. A large part of the suite is slow or flaky. Sanitizer builds are not run. Fear of changing code.

Delete a test only when the feature is gone, the test duplicates another, or it cannot fail for a real reason. Tell the user first. Give the reason.
