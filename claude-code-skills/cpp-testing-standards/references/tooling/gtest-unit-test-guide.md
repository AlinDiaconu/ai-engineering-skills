# Unit tests with GoogleTest: principles and a worked example

This file teaches the unit test from the start. It has the **principles**, a **setup that builds**, and one **complete example**: requirement, case table, code, tests, run.
The code in this file was built and run (GCC 9, C++17, GoogleTest v1.14.0, ASan + UBSan, shuffled and repeated).
Rules for all unit tests are in `../types/unit-tests.md`. Macro syntax is in `gtest-gmock-cheatsheet.md`.

## 1. What is a unit

A **unit** is the smallest part of the code that has its own behavior and a public interface: a function, or a class. A unit test:

- calls the unit through its **public interface**,
- replaces **all slow or non-deterministic dependencies** (clock, I/O, hardware) with fakes,
- checks the **result**,
- runs in **milliseconds**.

If a test needs a real file, a real network, a real clock, or many real classes, it is not a unit test. See `../types/integration-and-component-tests.md`.

## 2. Principles of unit testing

| # | Principle | In practice |
|---|---|---|
| 1 | **Test behavior, not implementation** | Check what the unit does. A refactoring must not break the test. |
| 2 | **One behavior per test** | One reason to fail. The name tells what broke. |
| 3 | **Isolate the unit** | Inject the clock, I/O, and hardware. No `sleep`. No globals. |
| 4 | **Be deterministic** | Same result on every run, every machine, every order. |
| 5 | **Be fast** | Milliseconds. A slow unit test is a design signal. |
| 6 | **Be independent** | No shared state. No order dependence. Run shuffled. |
| 7 | **Be readable** | Arrange, Act, Assert. Names and constants that show intent. |
| 8 | **Be able to fail** | Prove it. A test that cannot fail has no value. |
| 9 | **Cover the edges** | Valid, invalid, and boundary values. Error paths. |
| 10 | **Trace to a requirement** | Put the ID in a comment. Do not invent IDs. |

Why these ten exist and the full list (F.I.R.S.T., oracle, testability): `../foundations/principles.md`.

## 3. Setup that builds

Layout:

```
src/clock.h  src/heartbeat_monitor.h  src/heartbeat_monitor.cpp
test/fake_clock.h  test/heartbeat_monitor_test.cpp
CMakeLists.txt
```

`CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.14)
project(gtest_example CXX)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_library(heartbeat_monitor src/heartbeat_monitor.cpp)
target_include_directories(heartbeat_monitor PUBLIC src)

include(FetchContent)
FetchContent_Declare(googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG        v1.14.0)
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

enable_testing()
include(GoogleTest)

add_executable(heartbeat_monitor_test test/heartbeat_monitor_test.cpp)
target_include_directories(heartbeat_monitor_test PRIVATE test)
target_link_libraries(heartbeat_monitor_test PRIVATE heartbeat_monitor GTest::gtest_main)
target_compile_options(heartbeat_monitor_test PRIVATE -Wall -Wextra -Wpedantic -Wconversion -Wshadow -Werror)

gtest_discover_tests(heartbeat_monitor_test PROPERTIES LABELS "unit" TIMEOUT 30)
```

Build and run:

```bash
cmake -S . -B build
cmake --build build -j
ctest --test-dir build --output-on-failure
./build/heartbeat_monitor_test --gtest_shuffle --gtest_repeat=20
```

Notes:

- `GTest::gtest_main` gives you `main()`. You do not write one.
- The test links the **same library** as production code (`heartbeat_monitor`).
- `-Werror` with `-Wshadow`: do not name a lambda parameter `info` in `INSTANTIATE_TEST_SUITE_P`. GoogleTest uses that name inside the macro. Use `p`.
- Use `-DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-sanitize-recover=all"` for a sanitizer build.
- With `TEST_P`, CTest names show the printed parameter (for example `GetParam()=24-byteobject<...>`). Add `PrintTo` or `operator<<` for the parameter type to make them readable.

## 4. Worked example

### 4.1 Requirement (example only)

> **REQ-EXAMPLE-01.** The monitor shall report the peer as lost when no heartbeat arrived for **more than** the timeout. The timeout shall be positive.

This ID is an example. In real work, take the ID from the project. Do not invent it.

### 4.2 The unit

`src/clock.h`:

```cpp
#pragma once
#include <chrono>

using Duration  = std::chrono::milliseconds;
using TimePoint = std::chrono::milliseconds;

class IClock {
public:
    virtual ~IClock() = default;
    virtual TimePoint Now() const = 0;
};
```

`src/heartbeat_monitor.h`:

```cpp
#pragma once
#include "clock.h"

// Reports that the peer is lost when no heartbeat arrives within the timeout.
class HeartbeatMonitor {
public:
    // Throws std::invalid_argument when timeout is zero or negative.
    HeartbeatMonitor(const IClock& clock, Duration timeout);

    void OnHeartbeat();
    bool IsPeerLost() const;

private:
    const IClock& clock_;
    Duration timeout_;
    TimePoint lastHeartbeat_;
};
```

`src/heartbeat_monitor.cpp`:

```cpp
#include "heartbeat_monitor.h"

#include <stdexcept>

HeartbeatMonitor::HeartbeatMonitor(const IClock& clock, Duration timeout)
    : clock_{clock}, timeout_{timeout}, lastHeartbeat_{clock.Now()}
{
    if (timeout <= Duration::zero()) {
        throw std::invalid_argument{"timeout must be positive"};
    }
}

void HeartbeatMonitor::OnHeartbeat() { lastHeartbeat_ = clock_.Now(); }

bool HeartbeatMonitor::IsPeerLost() const
{
    const Duration age = clock_.Now() - lastHeartbeat_;
    return age > timeout_;   // negative age (clock went back) is not "lost"
}
```

The clock is injected through `IClock`. The unit never calls the system clock.

### 4.3 Case table (write this first)

| ID | Technique | State and input | Expected |
|---|---|---|---|
| TC-01 | Requirement | Just constructed | not lost |
| TC-02 | Requirement | Heartbeat every `timeout − 1 tick`, 10 times | never lost |
| TC-03 | Requirement | Wait `timeout`, heartbeat, wait `timeout` | not lost (timer reset) |
| TC-04 | BVA | Age = `timeout − 1 tick` | not lost |
| TC-05 | BVA | Age = `timeout` | not lost (limit is "more than") |
| TC-06 | BVA | Age = `timeout + 1 tick` | **lost** |
| TC-07 | Error guessing | Clock goes backwards | not lost |
| TC-08 | EP invalid | Timeout = 0 | throws `std::invalid_argument` |
| TC-09 | EP invalid | Timeout < 0 | throws `std::invalid_argument` |

Techniques: `../foundations/design-techniques.md`.

### 4.4 The fake

`test/fake_clock.h`:

```cpp
#pragma once
#include "clock.h"

class FakeClock final : public IClock {
public:
    TimePoint Now() const override { return now_; }
    void SetTime(TimePoint t) { now_ = t; }
    void Advance(Duration d) { now_ += d; }
private:
    TimePoint now_{};
};
```

### 4.5 The tests

`test/heartbeat_monitor_test.cpp`:

```cpp
#include "heartbeat_monitor.h"

#include <gtest/gtest.h>

#include <stdexcept>
#include <string>

#include "fake_clock.h"

namespace {

using std::chrono::milliseconds;

constexpr Duration kTimeout{100};
constexpr Duration kOneTick{1};

// Fixture: shared setup. The values that matter stay visible in each test.
class HeartbeatMonitorTest : public ::testing::Test {
protected:
    FakeClock clock_;
    HeartbeatMonitor monitor_{clock_, kTimeout};
};

// Verifies: REQ-EXAMPLE-01 (example ID)
TEST_F(HeartbeatMonitorTest, JustStarted_IsNotLost)
{
    EXPECT_FALSE(monitor_.IsPeerLost());
}

TEST_F(HeartbeatMonitorTest, HeartbeatKeepsComing_IsNeverLost)
{
    for (int i = 0; i < 10; ++i) {
        clock_.Advance(kTimeout - kOneTick);
        monitor_.OnHeartbeat();
        ASSERT_FALSE(monitor_.IsPeerLost()) << "cycle " << i;
    }
}

TEST_F(HeartbeatMonitorTest, HeartbeatResetsTheTimer)
{
    clock_.Advance(kTimeout);
    monitor_.OnHeartbeat();
    clock_.Advance(kTimeout);

    EXPECT_FALSE(monitor_.IsPeerLost());
}

TEST_F(HeartbeatMonitorTest, ClockGoesBackwards_IsNotLost)
{
    clock_.SetTime(milliseconds{500});
    monitor_.OnHeartbeat();
    clock_.SetTime(milliseconds{400});

    EXPECT_FALSE(monitor_.IsPeerLost());
}

// Boundary values around the timeout (BVA): one parameter row per case.
struct AgeCase {
    const char* name;
    Duration age;
    bool lost;
};

class HeartbeatAgeTest : public ::testing::TestWithParam<AgeCase> {};

TEST_P(HeartbeatAgeTest, LostOnlyWhenAgeIsAboveTimeout)
{
    FakeClock clock;
    HeartbeatMonitor monitor{clock, kTimeout};
    monitor.OnHeartbeat();

    clock.Advance(GetParam().age);

    EXPECT_EQ(monitor.IsPeerLost(), GetParam().lost);
}

INSTANTIATE_TEST_SUITE_P(
    TimeoutBoundary, HeartbeatAgeTest,
    ::testing::Values(AgeCase{"OneTickBefore", kTimeout - kOneTick, false},
                      AgeCase{"Exactly",       kTimeout,            false},
                      AgeCase{"OneTickAfter",  kTimeout + kOneTick, true}),
    [](const ::testing::TestParamInfo<AgeCase>& p) { return std::string{p.param.name}; });

TEST(HeartbeatMonitorConstruction, ZeroTimeout_Throws)
{
    FakeClock clock;
    EXPECT_THROW((HeartbeatMonitor{clock, Duration::zero()}), std::invalid_argument);
}

TEST(HeartbeatMonitorConstruction, NegativeTimeout_Throws)
{
    FakeClock clock;
    EXPECT_THROW((HeartbeatMonitor{clock, Duration{-1}}), std::invalid_argument);
}

}  // namespace
```

### 4.6 Run and read the result

```text
100% tests passed, 0 tests failed out of 9
```

Also run `ctest -L unit` (label), and the binary shuffled and repeated. All must pass.

### 4.7 What each part teaches

| Part | Lesson |
|---|---|
| `FakeClock` | Time is injected. The test moves time by hand. No `sleep`. |
| `TEST_F` + fixture | Shared setup. The values that matter (`kTimeout`) stay visible. |
| `ASSERT_FALSE(...) << "cycle " << i` | `ASSERT_` in a loop stops at the first failure. The message says which cycle. |
| `TEST_P` + `AgeCase` table | One body, one row per boundary case. No copy and paste. |
| `kTimeout ± kOneTick` | Boundary values are written from the requirement, not from magic numbers. |
| `EXPECT_THROW((HeartbeatMonitor{...}), ExactType)` | Exact exception type. Extra parentheses protect the comma in the macro. |
| Names `Unit_Condition_Result` | A failing name tells what broke. |
| `// Verifies: REQ-...` | Traceability. |

### 4.8 Check that the tests can fail

Do this once, in a scratch copy. Never leave the change in the code. The results below were measured: each change failed exactly the tests that are listed.

| Change in `IsPeerLost` | Tests that must fail |
|---|---|
| `age > timeout_` → `age >= timeout_` | TC-03 and TC-05 (`Exactly`) |
| `age > timeout_` → `age > timeout_ + kOneTick` | TC-06 (`OneTickAfter`) |
| Remove `lastHeartbeat_ = clock_.Now();` in `OnHeartbeat` | TC-02, TC-03, TC-07 |
| Constructor: remove the `throw` | TC-08, TC-09 |

If a change does not fail any test, a case is missing. See `../practice/review.md` and `../types/mutation-tests.md`.

## 5. Recipe for any class

1. Read the class and find the requirement.
2. List its dependencies. Put each slow or non-deterministic one behind an interface. Write a fake.
3. Write the **case table**: requirement, EP, BVA, state, error guessing.
4. Write the simplest test first (the constructor and the start state).
5. Add one test per row. Use `TEST_P` when the body repeats.
6. Build with warnings as errors. Run. Run shuffled and repeated. Run with sanitizers.
7. Prove that each test can fail (section 4.8).
8. Check coverage for gaps (`../practice/coverage-and-metrics.md`). Report.

## 6. Common first mistakes

| Mistake | Fix |
|---|---|
| Test calls `std::chrono::steady_clock::now()` | Inject `IClock`. Use `FakeClock`. |
| `EXPECT_TRUE(a == b)` | `EXPECT_EQ(a, b)` |
| Test for the exact boundary missing | Add `limit − 1`, `limit`, `limit + 1` |
| `EXPECT_ANY_THROW` | `EXPECT_THROW` with the exact type |
| A test checks two behaviors | Split it |
| Test passes the first time you write it | Make it fail once on purpose |
| Tests share a static counter | Remove the static. Reset state in `SetUp`. |
