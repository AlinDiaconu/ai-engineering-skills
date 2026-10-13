# Testing principles (C++)

## Why we test C++ code

- Find defects early. C++ has undefined behavior, lifetime errors, and data races that the compiler does not always find.
- Give confidence to refactor and to upgrade compilers and libraries.
- Show that requirements are met. Give evidence for a release.
- Prevent regression. A fixed bug stays fixed.
- Document real behavior. A good test shows how to use the class.

Tests do not prove that code is correct. They show that the tested cases work.

## The seven classic principles (ISTQB), in C++ terms

| # | Principle | Meaning in C++ practice |
|---|---|---|
| 1 | Testing shows the presence of defects, not their absence | A green run with sanitizers is not "no UB". Report what you tested. |
| 2 | Exhaustive testing is impossible | Use equivalence classes, boundaries, and properties. Do not loop over all inputs. |
| 3 | Test early (shift left) | Write tests with the code. Use warnings and static analysis at compile time. |
| 4 | Defects cluster | Parsers, state machines, ownership code, and concurrency code hold most defects. Test them deeper. |
| 5 | Pesticide paradox | Old tests find fewer new bugs. Add new cases, fuzzing, and mutation testing. |
| 6 | Testing depends on context | Real-time code, a library, and a desktop tool need different tests. |
| 7 | Absence-of-defects fallacy | Code that meets a wrong requirement still fails. Check the requirement too. |

## F.I.R.S.T. (a good unit test is…)

| Letter | Property | C++ meaning |
|---|---|---|
| **F** | Fast | Milliseconds. No real I/O, no sleep. |
| **I** | Independent | No test depends on another. No shared static state. |
| **R** | Repeatable | Same result on every machine, compiler, and run. Fake clock, fixed seed. |
| **S** | Self-validating | The test says pass or fail. No reading of logs. |
| **T** | Timely | Written with the code (or before). |

## Core concepts

| Term | Meaning |
|---|---|
| **UUT / SUT** | Unit (system) under test |
| **Test oracle** | The source of the expected result: requirement, specification, reference model, property. No oracle, no valid test. |
| **Test case** | Inputs + preconditions + expected result |
| **Fixture** | The fixed state that a test needs (`TEST_F` class) |
| **AAA** | Arrange, Act, Assert (Given, When, Then) |
| **Black-box / white-box / grey-box** | Test without / with / with partial knowledge of the code |
| **Verification** | Built right? (matches the specification) |
| **Validation** | Built the right thing? (meets the user need) |
| **Error, defect, failure** | A person makes an error. It creates a defect. A running defect can cause a failure. |
| **Regression** | A feature that worked and now does not |
| **Seam** | A place where you can change behavior without editing the code (interface, template parameter, link-time swap) |

## Behavior, not implementation

- A test describes **what** the code does, not **how**.
- Good: "when the cart has two items of 5, the total is 10".
- Bad: "the method calls `Sum()` once and then `Round()`".
- Test through the **public interface**. A behavior-preserving refactoring must not break a test.
- If a refactoring breaks a test, the test is tied to the implementation. Fix the test.

## One reason to fail

- Each test checks one behavior.
- The name tells what broke.
- Many `EXPECT_*` lines are fine when they check **one** behavior (for example all fields of one result).

## DRY vs DAMP

- Production code: DRY (do not repeat).
- Test code: **DAMP** (Descriptive And Meaningful Phrases). A reader must understand a test without jumping between many files.
- Share setup in builders and helpers. Keep the **important values** visible in the test.

## Testability (C++)

C++ code is easy to test when:

- Dependencies come in from outside (constructor parameter or template parameter). They are not created inside.
- Logic is separate from I/O, hardware, and framework code.
- Time, random values, and the environment come through an interface.
- Functions are small, with clear inputs and outputs.
- There are no singletons, global objects, or hidden static state.
- Ownership is clear (RAII, smart pointers, values).

Hard-to-test code is a **design signal**. Tell the user. Suggest a small design change. Do not use `#define private public`, `friend` hacks, or other tricks.

## Cost, value, and risk

- Each test has a cost (write, run, maintain) and a value (the risk it reduces).
- Test more where **probability × impact** of failure is high (parsers, state machines, ownership, concurrency).
- Test less where it is low (trivial getters, framework code).
- Delete tests that give no value. Tell the user first.

## Test oracles: common types

| Oracle | Example | Weakness |
|---|---|---|
| Requirement / specification | "limit is 130 km/h" | May be wrong or incomplete |
| Reference implementation | Old or slow but correct version | Same bug in both |
| Property / invariant | `Decode(Encode(x)) == x` | Does not check every detail |
| Golden file | Saved earlier output | Locks in old bugs. Review each change. |
| Consistency | Two paths give the same result | Both may be wrong the same way |
| Human judgment | Exploratory test, visual check | Not repeatable |

## C++ specific risk list

Use this list when you decide where to test deeper:

- Undefined behavior (overflow, out-of-bounds, null, misaligned, use-after-free).
- Object lifetime and ownership (dangling references, moved-from objects).
- Exception safety and `noexcept`.
- Integer and floating-point handling (conversion, rounding, NaN).
- Data races and lock order.
- Initialization order and uninitialized data.
- Template instantiation with unusual types.
- ABI and layout of shared structs.
- Platform differences (endianness, alignment, `long` size, floating-point mode).
- Compiler and optimization differences (Debug vs Release).
