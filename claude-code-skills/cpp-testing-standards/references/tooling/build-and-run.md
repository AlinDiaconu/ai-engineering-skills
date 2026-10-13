# Build, run, and tools (C++)

Use the framework, build system, and tools that the project already uses. Check `CMakeLists.txt`, presets, and CI files first. Add a new tool only with the user's approval.

## Unit-test frameworks

| Framework | Style | Mocking | Notes |
|---|---|---|---|
| **GoogleTest + GoogleMock** | Macros, fixtures, `TEST_P`, typed tests | Built in | Default in this skill. Syntax: `gtest-gmock-cheatsheet.md` |
| **Catch2** | `TEST_CASE`, `SECTION`, BDD macros | None (use Trompeloeil, FakeIt) | A `SECTION` re-runs the test case for each leaf |
| **doctest** | Like Catch2, very low compile cost | None | Good for small libraries |
| **Boost.Test** | Macros, fixtures, data-driven | None | Fits Boost-based projects |
| **CppUTest** | xUnit style, built-in leak check | CppUMock | Made for embedded C and C++. Small footprint. |
| **Unity + CMock / Ceedling** | C style | CMock generates mocks from headers | For C code on small targets |

Mocking add-ons: Trompeloeil (modern, header-only), FakeIt (fluent, no macros). Property, fuzz, mutation, and benchmark tools are in the matching `../types/` files.

Do not mix two unit-test frameworks in one test executable. The rules in this skill hold for any framework. Only the syntax changes: Catch2 `REQUIRE` ≈ `ASSERT_*`, `CHECK` ≈ `EXPECT_*`.

## CMake and CTest

```cmake
enable_testing()
include(GoogleTest)

add_executable(session_state_machine_test session_state_machine_test.cpp)
target_link_libraries(session_state_machine_test PRIVATE session_state_machine GTest::gtest_main GTest::gmock)
target_compile_options(session_state_machine_test PRIVATE -Wall -Wextra -Wpedantic -Wconversion -Wshadow -Werror)

gtest_discover_tests(session_state_machine_test PROPERTIES LABELS "unit" TIMEOUT 30)
```

- `gtest_discover_tests`: each test is a separate CTest entry.
- Set `TIMEOUT` on every test. A hang must fail, not block the build.
- Set `LABELS` by type: `unit`, `smoke`, `integration`, `system`, `perf`, `hil`. Select with `ctest -L unit`, `ctest -L "unit|smoke"`, `ctest -LE "hil|perf"`.
- One test executable per module (fast builds, clear failures).
- Link tests to the **same library target** as production. Do not re-compile production sources into the test.
- Use `CMakePresets.json` for `debug`, `release`, `asan-ubsan`, `tsan`, `coverage`.
- Get dependencies with the project method (`FetchContent`, `find_package`, Conan, vcpkg). Pin versions.
- Do not fix a warning by silencing it in a test. Fix the cause.

## Layout and naming

| Item | Convention |
|---|---|
| Test directory | `test/` or `tests/`, mirror of `src/` |
| Test file | `<unit>_test.cpp` |
| Suite name | The unit (`SessionStateMachine`), or `SessionStateMachineTest` for a fixture |
| Shared fakes, builders, matchers | `test/support/` library |
| Test data | `test/data/`. Path comes from the build system. |

Follow the project layout. Do not move tests to a new layout without approval.

## Commands

| Task | Command |
|---|---|
| Configure | `cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug` |
| Build | `cmake --build build -j` |
| Run all | `ctest --test-dir build --output-on-failure` |
| Run by name | `ctest --test-dir build -R <regex>` |
| Re-run failures | `ctest --test-dir build --rerun-failed --output-on-failure` |
| Parallel | `ctest --test-dir build -j <n>` (tests must be independent) |
| One test | `<binary> --gtest_filter=Suite.Test` |
| List tests | `<binary> --gtest_list_tests` |
| Shuffle | `<binary> --gtest_shuffle --gtest_random_seed=<n>` |
| Repeat | `<binary> --gtest_repeat=100 --gtest_break_on_failure` |
| XML report | `<binary> --gtest_output=xml:report.xml` |

Before you report a pass for a new test, run it **shuffled** and **repeated**.

## Sanitizers

Use a **separate** build for each. Do not combine ASan and TSan. Use `-O1` or `-O0` for readable reports.

| Sanitizer | Flags | Finds |
|---|---|---|
| **ASan** (+ LSan) | `-fsanitize=address -fno-omit-frame-pointer` | Out-of-bounds, use-after-free, double free, leaks |
| **UBSan** | `-fsanitize=undefined -fno-sanitize-recover=all` | Signed overflow, bad shift, null deref, misaligned access, bad enum |
| **TSan** | `-fsanitize=thread` | Data races, lock-order problems |
| **MSan** | `-fsanitize=memory` (Clang, instrumented standard library) | Uninitialized reads |

- A sanitizer report is a **test failure**.
- A clean run is not a proof. It shows that no issue occurred in the tested paths.
- Valgrind (`memcheck`, `helgrind`) is a slower alternative.
- Run sanitizers on the **host**. They may not work on the target.

## Static analysis

`clang-tidy`, `cppcheck`, compiler warnings, and the project's coding-standard checker. Run them on test code too (same rules, or an agreed relaxed set).

## Host and target

| Where | Use for |
|---|---|
| **Host** (developer machine, CI) | Unit, component, sanitizers, coverage, fault injection with fakes |
| **Target** (real device or target-like board) | Timing, real I/O, platform effects (endianness, alignment, floating point), HIL |

- Keep most tests on the host. Run a smaller set on the target.
- Same source, same cases. Only the build and the I/O layer differ (`../types/sil-hil-tests.md`).
- On small targets: CppUTest or Unity when GoogleTest does not fit (no exceptions, no RTTI, small RAM). Match the production flags (`-fno-exceptions`, `-fno-rtti`).
- Do not run tests on shared or real hardware without the user's approval.

## CI

- Stages and timing: `../foundations/strategy.md`.
- Store test reports, coverage reports, seeds, and logs as CI artifacts.
- A red build blocks the merge.
