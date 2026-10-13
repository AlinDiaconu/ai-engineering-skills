---
name: cpp-testing-standards
description: Complete C++ testing skill. Use when the user asks to write, design, review, plan, or debug tests for C++ code; asks which type of test to use and how (unit, mock, smoke, black-box, white-box, integration, system, regression, property-based, fuzz, mutation, benchmark, concurrency, memory, golden, compile-time, fault injection, SIL/HIL); asks about test principles, strategy, TDD, test doubles, coverage and MC/DC, GoogleTest/GoogleMock, sanitizers, CMake/CTest, or flaky tests. It is an index. It tells you which reference file to read. For how to write production C++, use cpp-engineering-standards.
---

# C++ Testing

Use ASD-STE100 for test comments and reports. Follow existing project test conventions first.
This file is the **index**. Read only the reference files that the task needs. Do not read all of them.

## Structure

| Part | Folder | Answers |
|---|---|---|
| 1. Foundations | `references/foundations/` | **Why** and **what**: principles, strategy, case design, doubles, TDD |
| 2. Test types | `references/types/` | **Which** test, **when**, **how**: one file per type |
| 3. Practice | `references/practice/` | **How well**: writing rules, determinism, coverage, review, debugging |
| 4. Tooling | `references/tooling/` | **With what**: first unit test, frameworks, CMake, sanitizers, GoogleTest syntax |

## Where to look

### 1. Foundations

| The task is about… | Read |
|---|---|
| Why we test. Principles. F.I.R.S.T. Oracle. Testability. C++ risk list. | `references/foundations/principles.md` |
| Test levels. Pyramid. Risk. Build matrix. CI stages. Test plan. Exit criteria. | `references/foundations/strategy.md` |
| How to choose cases (EP, BVA, decision table, states, pairwise). Numeric and interface rules. | `references/foundations/design-techniques.md` |
| Dummy, stub, spy, mock, fake. Seams. Fake clock. Contract tests. | `references/foundations/test-doubles.md` |
| TDD, BDD, ATDD. Legacy code. Seams and dependency breaking. | `references/foundations/tdd-and-legacy.md` |

### 2. Test types (start with the overview)

| The task is about… | Read |
|---|---|
| **Which type to use.** Choice table. Rules for all types. | `references/types/00-overview.md` |
| Unit tests (special members, templates, death tests) | `references/types/unit-tests.md` |
| Mock tests (interaction, GoogleMock, strict vs nice) | `references/types/mock-tests.md` |
| Smoke tests | `references/types/smoke-tests.md` |
| Black-box tests (from the specification) | `references/types/black-box-tests.md` |
| White-box tests (structure, loops, reaching paths) | `references/types/white-box-tests.md` |
| Integration and component tests | `references/types/integration-and-component-tests.md` |
| System and scenario tests | `references/types/system-tests.md` |
| Regression and characterization tests. Bug-fix workflow. | `references/types/regression-and-characterization-tests.md` |
| Fault injection and error-path tests | `references/types/fault-injection-tests.md` |
| Property-based tests | `references/types/property-based-tests.md` |
| Fuzz tests | `references/types/fuzz-tests.md` |
| Mutation tests | `references/types/mutation-tests.md` |
| Benchmark and performance tests | `references/types/benchmark-and-performance-tests.md` |
| Concurrency tests | `references/types/concurrency-tests.md` |
| Memory and resource tests (leaks, RAII, no heap) | `references/types/memory-and-resource-tests.md` |
| Golden and snapshot tests | `references/types/golden-and-snapshot-tests.md` |
| Compile-time tests (`static_assert`, concepts) | `references/types/compile-time-tests.md` |
| Simulation and target tests (SIL, PIL, HIL) | `references/types/sil-hil-tests.md` |

### 3. Practice

| The task is about… | Read |
|---|---|
| How a test looks: structure, naming, fixtures, assertions, anti-patterns | `references/practice/writing-tests.md` |
| Determinism. Time, random, threads. Real-time properties. | `references/practice/determinism-and-realtime.md` |
| Coverage types and targets. MC/DC. Metrics. Traceability. | `references/practice/coverage-and-metrics.md` |
| Review checklist. Prove a test can fail. Weak tests. | `references/practice/review.md` |
| Failing or flaky tests. Slow tests. Test debt. | `references/practice/flaky-and-debugging.md` |

### 4. Tooling

| The task is about… | Read |
|---|---|
| **First unit test with GoogleTest:** unit-test principles, a setup that builds, a complete worked example, a recipe | `references/tooling/gtest-unit-test-guide.md` |
| Frameworks. CMake and CTest. Layout. Commands. Sanitizers. Host and target. | `references/tooling/build-and-run.md` |
| GoogleTest and GoogleMock macros, matchers, mocks | `references/tooling/gtest-gmock-cheatsheet.md` |

If a task matches several rows, read each matching file. For the meaning of an abbreviation (MC/DC, SIL, HIL, seam, RAII, and so on), read `references/glossary.md`.

## Core rules (always apply)

1. Find the **requirement** first. Do not invent requirement IDs. If none exists, tell the user.
2. Write the **case table** before the code.
3. Test through the **public interface**. Never test private members.
4. A test must be able to fail. Prove it (`references/practice/review.md`).
5. **No `sleep`**, no system clock, no real network or files in unit tests. Inject a fake clock.
6. **No `==` on floating point.** Use a tolerance, and state why.
7. Tests are independent, deterministic, and pass in shuffled order.
8. Test the error paths: code that can fail has **fault tests** (detected, reacted to, end state).
9. Test at the **lowest level** that can show the defect.
10. Every bug fix starts with a test that fails for that bug.
11. Do not change production code to make it testable without asking the user.
12. Never weaken, skip, or delete a check to make a test pass without telling the user.
13. Report only results that you saw. Run the tests.
14. Do not run tests on shared or real hardware without the user's approval.

## Workflow

1. **Read** the unit under test: interface, inputs, outputs, state, side effects.
2. **Find the requirement** or the risk (design note, bug report). If none exists, tell the user.
3. **Pick the type and level** (`references/types/00-overview.md`, `references/foundations/strategy.md`).
4. **Design the cases** and write the case table (`references/foundations/design-techniques.md`, `references/types/fault-injection-tests.md`).
5. **Isolate** dependencies (`references/foundations/test-doubles.md`).
6. **Write** the tests (`references/practice/writing-tests.md`, `references/tooling/gtest-gmock-cheatsheet.md`).
7. **Build and run.** Run shuffled and repeated. Run sanitizers (`references/tooling/build-and-run.md`).
8. **Check strength and coverage** (`references/practice/review.md`, `references/practice/coverage-and-metrics.md`).
9. **Report.**

If the build or test tool is not available, say so. Do not claim a pass that you did not see.

## Report format

Short. Caveman style. ASD-STE100.

- **Unit tested:** name and file.
- **Cases:** number of tests. Give the case table when the user asks.
- **Result:** real pass and fail count from the real run. Failing output.
- **Coverage / mutation:** numbers, tool, type. Only if measured.
- **Gaps:** what is not tested, and why.
- **Defects found:** production issues you saw. Do not fix them without approval.
- **Not run:** checks that you could not run (sanitizers, target hardware, timing).
