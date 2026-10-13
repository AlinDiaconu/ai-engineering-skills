# C++ test types: overview and choice

Read this file first. Then read the file of the type that you choose.
For the basic principles and the test strategy, see `../foundations/principles.md` and `../foundations/strategy.md`.

## Map

| Type | File | One-line purpose |
|---|---|---|
| Unit | `unit-tests.md` | One class or function. Fast. Most tests. |
| Mock-based | `mock-tests.md` | Check an interaction that is the requirement |
| Black-box | `black-box-tests.md` | Test from the specification, through the interface |
| White-box | `white-box-tests.md` | Use the code structure to choose cases and coverage |
| Smoke | `smoke-tests.md` | Quick check that the build starts and the core path works |
| Integration and component | `integration-and-component-tests.md` | Real parts working together |
| System and scenario | `system-tests.md` | Full function chain in a simulated world |
| Regression and characterization | `regression-and-characterization-tests.md` | Keep fixed bugs fixed. Make legacy code safe to change |
| Property-based | `property-based-tests.md` | A rule that holds for many generated inputs |
| Fuzz | `fuzz-tests.md` | Random input to find crashes and memory errors |
| Mutation | `mutation-tests.md` | Measure how strong the tests are |
| Benchmark and performance | `benchmark-and-performance-tests.md` | Speed, timing, resource use |
| Concurrency | `concurrency-tests.md` | Races, deadlocks, ordering |
| Memory and resource | `memory-and-resource-tests.md` | Leaks, allocation, lifetime, resource limits |
| Golden and snapshot | `golden-and-snapshot-tests.md` | Compare large output to a saved copy |
| Compile-time | `compile-time-tests.md` | `static_assert`, concepts, type traits, "must not compile" |
| SIL and HIL | `sil-hil-tests.md` | Code on host or on target with a simulated plant |
| Fault injection | `fault-injection-tests.md` | Cause faults. Check detect, react, end state |

## Choose the type

| Situation | Type |
|---|---|
| Logic, limits, state transitions in one class | Unit (+ black-box and white-box design) |
| The call to a collaborator is the requirement ("send one alarm") | Mock-based |
| New build. Is it alive? | Smoke |
| Two parts exchange data | Integration |
| A group of classes behind one interface | Component |
| End-to-end behavior in a defined scenario | System / scenario |
| A bug was reported | Regression (test first) |
| Old code, no tests, must change it | Characterization |
| Parser, decoder, serializer, math with an invariant | Property-based + fuzz |
| Code that reads untrusted or external data | Fuzz |
| High coverage, but doubt about test strength | Mutation |
| A time or speed requirement | Benchmark / performance |
| Shared data, locks, threads | Concurrency (TSan) |
| Resource owner, cyclic path with no heap | Memory and resource |
| Large structured output | Golden |
| Template, concept, trait, constexpr, API of types | Compile-time |
| Real timing, real I/O, real device | HIL |
| Error handling and recovery | Fault injection |

## Best-practice rules for all types

1. Pick the **lowest level** that can show the defect.
2. Each test traces to a **requirement** or a **risk**.
3. Each test is **deterministic**, **independent**, and **can fail**.
4. Keep the **fast** tests (unit, component) separate from the **slow** tests (system, HIL, fuzz long run, mutation).
5. Label tests (CTest labels or GoogleTest suites) by type: `unit`, `smoke`, `integration`, `system`, `perf`, `hil`.
6. Run `unit` and `smoke` at each commit. Run `integration` at each merge. Run `system`, `perf`, `fuzz`, `mutation`, and `hil` on a schedule or before release.

## Where to run each type

Label tests by type (`unit`, `smoke`, `integration`, `system`, `perf`, `hil`) and select by label. See `../tooling/build-and-run.md`. Pipeline stages and the test pyramid: `../foundations/strategy.md`.
