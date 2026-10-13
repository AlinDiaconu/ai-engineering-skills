# Glossary

Short meaning of terms and abbreviations used in this skill. Details are in the file that is named in the last column.

## Test terms

| Term | Meaning | Details |
|---|---|---|
| **UUT / SUT** | Unit (system) under test | `foundations/principles.md` |
| **Oracle** | The source of the expected result (requirement, model, property) | `foundations/principles.md` |
| **Seam** | A place where you change behavior without editing the code | `foundations/tdd-and-legacy.md` |
| **Fixture** | The fixed state that tests need | `practice/writing-tests.md` |
| **AAA** | Arrange, Act, Assert | `practice/writing-tests.md` |
| **FIRST** | Fast, Independent, Repeatable, Self-validating, Timely | `foundations/principles.md` |
| **Double** (dummy, stub, spy, mock, fake) | A replacement for a real dependency | `foundations/test-doubles.md` |
| **EP / BVA** | Equivalence partitioning / boundary value analysis | `foundations/design-techniques.md` |
| **Pairwise** | Test every pair of parameter values, not every full combination | `foundations/design-techniques.md` |
| **MC/DC** | Modified condition/decision coverage. Each condition independently changes the decision. | `practice/coverage-and-metrics.md` |
| **Mutant** | A small deliberate change in the code. A good test suite fails on it. | `types/mutation-tests.md` |
| **Golden file** | Saved known-good output that a test compares to | `types/golden-and-snapshot-tests.md` |
| **Characterization test** | A test that records what legacy code does now | `types/regression-and-characterization-tests.md` |
| **Fault injection** | Causing a fault on purpose to check the reaction | `types/fault-injection-tests.md` |
| **Flaky test** | A test that passes and fails with no code change | `practice/flaky-and-debugging.md` |
| **Smoke test** | Small, fast check that the build is alive | `types/smoke-tests.md` |
| **Corpus** | The saved set of inputs for a fuzzer | `types/fuzz-tests.md` |
| **Traceability** | The link from a requirement to its tests and results | `practice/coverage-and-metrics.md` |

## Simulation levels

| Term | Meaning |
|---|---|
| **MIL** | Model-in-the-loop. The function model runs in a modeling tool. |
| **SIL** | Software-in-the-loop. The real C++ source, built for the host, with a simulated environment. |
| **PIL** | Processor-in-the-loop. The real code on the target CPU (or emulator), with a simulated environment. |
| **HIL** | Hardware-in-the-loop. The real code on the real device, with a simulated environment. |

Details: `types/sil-hil-tests.md`.

## C++ and tool terms

| Term | Meaning |
|---|---|
| **RAII** | Resource acquisition is initialization. An object owns a resource and releases it in its destructor. |
| **UB** | Undefined behavior. The C++ standard gives no rule for the result. |
| **ABI** | Application binary interface: layout and calling rules between compiled parts. |
| **ASan / UBSan / TSan / MSan** | Address, undefined-behavior, thread, and memory sanitizers (`tooling/build-and-run.md`). |
| **LSan** | Leak sanitizer. Part of ASan on Linux. |
| **ULP** | Unit in the last place. The distance between two neighboring floating-point numbers. `EXPECT_DOUBLE_EQ` allows 4 ULP. |
| **CRC** | Cyclic redundancy check. A checksum for messages. |
| **WCET** | Worst-case execution time. |
| **Jitter** | Variation of a period (for example cycle start times). |
| **CTest** | The CMake test runner. Runs test executables and selects them by name or label. |
| **STE** | Simplified Technical English (ASD-STE100). The writing standard for comments and reports. |
