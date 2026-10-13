# Determinism and real-time

A test must give the same result on every run, on every machine.

## Sources of non-determinism

| Source | Rule |
|---|---|
| **Time** | Inject a clock. Never call `sleep`, `std::this_thread::sleep_for`, or the system clock in a test. Advance a fake clock by hand. |
| **Randomness** | Use a fixed seed. Print the seed on failure. |
| **Threads** | Do not depend on thread timing. Use latches, barriers, or condition variables with a fake clock. Run the unit step by step when you can. |
| **Files and network** | Do not use them in unit tests. Use in-memory fakes. |
| **Order** | Tests must not depend on the order of other tests. Run with `--gtest_shuffle` and `--gtest_repeat` to prove it. |
| **Static state** | Reset all static or global state in `SetUp` and `TearDown`. |
| **Uninitialized data** | Initialize all data. Run the memory sanitizer or Valgrind to find the rest. |
| **Hash and container order** | Do not depend on `unordered_*` iteration order. Sort or use matchers like `UnorderedElementsAre`. |
| **Pointer values** | Do not compare or print raw addresses. |
| **Environment** | Set locale, time zone, and environment variables in the fixture. Restore them after. |
| **Floating point** | Use a tolerance. Know that results can differ by compiler flags (`-ffast-math`, FMA) and platform. |

## Testing time-based logic

- Drive the unit with **logical time**: set the time, call `Step()`, check the output. Repeat.
- Test around each timeout and each threshold: just below, exactly at, just above.
- Test **cycle jitter**: a step that comes early or late.
- Test **time wrap-around** and **time going backward** if possible.
- Test **debounce and hold times**: a glitch shorter than the limit must not trigger. A fault longer than the limit must trigger.
- Test **stale data**: age of the last message grows with the clock. Check the reaction at the exact age limit.

## Testing threaded code

- Prefer to test the **logic** single-threaded, and test the **threading wrapper** separately.
- For races: run many iterations under **ThreadSanitizer**. A passing run is not a proof.
- Use `std::latch`, `std::barrier`, promises and futures, or condition variables to force an interleaving. Never use `sleep` to force one.
- Use a **timeout** on every wait in a test, so a deadlock fails the test and does not hang the build.
- Test shutdown: stop while busy, stop twice, stop before start.

## Real-time properties

| Property | How to test |
|---|---|
| **Logical timing** (required reaction in N cycles) | Fake clock and step count. Fully deterministic. Use in unit tests. |
| **Execution time** (WCET, cycle time) | Measure on a target-like machine under load. Report the maximum, not the mean. Mark as a timing test. Keep it **out of** unit tests. |
| **Jitter** | Record cycle start times. Report the spread and the maximum. |
| **Deadline miss reaction** | Inject a late task or a long step in a fake. Check the reaction (watchdog, degrade, flag). |
| **No allocation in the cyclic path** | Allocation counter or hook (see below). |
| **No blocking in the cyclic path** | Review plus test with fakes that fail the test if a blocking call happens. |
| **Bounded memory** | Run long with fixed input. Check that memory use stays flat. |

## Checking for allocation

Count `operator new` calls in a separate test binary. The method and code are in `../types/memory-and-resource-tests.md`.

## Seeds and logs

- Print the random seed at the start of any random test.
- On failure, print the inputs and the state that are needed to reproduce the failure.
- Allow the seed to be set by an environment variable or a flag.

## Rules for timing test results

- Never put a wall-time check (for example "less than 2 ms") in a unit test. Machines differ.
- Run timing tests on a defined machine under a defined load.
- Report: the machine, the load, the number of runs, the minimum, the median, the maximum.
