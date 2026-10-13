# Test strategy and the test pyramid (C++)

## The test pyramid

```
        /\        System / scenario / HIL   few, slow, expensive
       /--\
      /    \      Integration / component   some, medium speed
     /------\
    /        \    Unit (host)               many, fast, cheap
   /----------\
```

- Many fast unit tests. Few slow system tests.
- Lower tests are fast, stable, and show the exact location of a defect.
- Upper tests are realistic, but slow and more likely to be flaky.

### Variants

| Model | Shape | Use when |
|---|---|---|
| **Pyramid** | Mostly unit tests | Logic-heavy code, libraries, embedded (default choice) |
| **Trophy / diamond** | Thick integration layer | Many components with thin logic, middleware |
| **Ice-cream cone** (anti-pattern) | Mostly system and manual tests | Avoid. Move checks down. |

## Risk-based testing

1. List components and functions.
2. Rate **probability of failure** (new code, complexity, change rate, past bugs) and **impact** (data loss, security, availability, cost).
3. Multiply. Test the high scores first and deepest.
4. Review when code or use changes.

Typical high-impact areas: parsers and message decoders, state machines, error handling, ownership code, and concurrency code.

## Test levels

| Level | Scope | Dependencies | Use for |
|---|---|---|---|
| Unit | One class or function | Replaced by fakes or mocks | Logic, limits, state transitions |
| Component | Group of classes behind one interface | Only external I/O replaced | Interface contracts, data flow |
| Integration | Several real components | Simulated environment | Timing, message flow, start-up and shutdown |
| System / scenario | The whole program | Simulated or test environment | End-to-end behavior, start-up, recovery |
| SIL / PIL / HIL | Host build, target CPU, or real device with a simulated environment | Simulated peripherals | Real timing, real hardware interfaces |

Test at the lowest level that can show the defect. Do not mix levels in one test file.

## What to test where

| Concern | Best level | Type file |
|---|---|---|
| Algorithm, calculation, parsing | Unit | `../types/unit-tests.md` |
| Branches, error paths, limits | Unit (+ white-box) | `../types/white-box-tests.md` |
| Message format, wire layout | Unit + golden + compile-time | `../types/golden-and-snapshot-tests.md`, `../types/compile-time-tests.md` |
| Interface between parts | Integration | `../types/integration-and-component-tests.md` |
| Start-up, wiring, configuration | Integration / smoke | `../types/smoke-tests.md` |
| Error handling and recovery | Unit + component + scenario | `../types/fault-injection-tests.md`, `../types/system-tests.md` |
| Real timing and I/O | HIL | `../types/sil-hil-tests.md` |
| Speed and memory | Benchmark, memory tests | `../types/benchmark-and-performance-tests.md`, `../types/memory-and-resource-tests.md` |
| Untrusted input | Fuzz + property-based | `../types/fuzz-tests.md`, `../types/property-based-tests.md` |
| Threads and locks | Concurrency + TSan | `../types/concurrency-tests.md` |

## Environments

| Environment | Purpose | Notes |
|---|---|---|
| **Developer host** | Fast feedback: unit, component, sanitizers | Fakes for I/O and time |
| **CI host** | Every change: build matrix, unit, sanitizers, coverage | Clean, reproducible container |
| **SIL (simulator)** | Scenarios and function chain | Fixed scenario files and seeds |
| **Target / HIL bench** | Timing, real I/O, platform effects | Known bench state. Approval needed. |

Rules:

- Make environments reproducible (containers, scripts, pinned tool versions).
- Each test builds and cleans its own data.
- Do not use real personal data in tests.

## Build matrix

Test on the combinations that you ship. Use a CI matrix.

| Axis | Values |
|---|---|
| Compiler | GCC, Clang (and the target compiler) |
| Version | Oldest and newest supported |
| Build type | Debug and Release (and the production flags) |
| Sanitizers | None, ASan+UBSan, TSan (separate builds) |
| Platform | Host, target (or emulator) |
| Standard | The project's C++ standard (for example C++17) |

Many defects show only in Release (optimizer) or on one compiler. Test both build types.

## CI pipeline stages

| Stage | Content | Target time |
|---|---|---|
| 1. Pre-commit | Format, lint, build, fast unit tests | seconds |
| 2. Commit | Build matrix, all unit tests, static analysis, `smoke` | minutes |
| 3. Merge | Integration tests, sanitizer builds, coverage | 10–30 min |
| 4. Nightly | Scenario (SIL), fuzz (time-boxed), mutation (critical modules), benchmarks, long runs | hours |
| 5. Release | Full HIL, timing, soak, traceability report | per plan |

- Fail fast. Put cheap checks first.
- A red build blocks the merge. Fix or revert at once.
- Label tests (`unit`, `smoke`, `integration`, `system`, `perf`, `hil`) and select by label (`ctest -L`).
- Save test reports, coverage reports, seeds, and logs as artifacts.

## Shift-left and shift-right

- **Shift left:** compiler warnings as errors, static analysis (`clang-tidy`, `cppcheck`, the project coding-standard checker), tests with the code, review.
- **Shift right:** field data, logging, and diagnostics feed new tests. Each field issue becomes a regression test or a scenario.

## Test plan (minimum content)

1. Scope: what is in and out.
2. Risks and priorities.
3. Test types and levels, with the reason.
4. Environments, tools, and how to run them.
5. Entry criteria (when testing can start).
6. Exit criteria (when testing is complete).
7. Traceability: requirement → test → result.
8. Roles, schedule, and reporting.
9. Defect handling.

### Entry and exit criteria: examples

- **Entry:** build passes, smoke tests pass, test bench is ready, requirements are stable.
- **Exit:** planned tests run, all critical tests pass, no open blocker, coverage target met, sanitizers clean, timing target met, known gaps written down and accepted.

## Defect handling

- Report: steps to reproduce, expected, actual, software version, environment, logs, seed.
- Add a regression test for each fixed defect (`../types/regression-and-characterization-tests.md`).
- Track trends. A cluster shows where to test more.
