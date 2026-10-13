# Test doubles (C++)

A **test double** replaces a real dependency in a test.
This file owns the **theory** and the **C++ seam techniques**. For the mock test type and GoogleMock use, see `../types/mock-tests.md`. For macro syntax, see `../tooling/gtest-gmock-cheatsheet.md`.

## Types

| Type | What it does | Use when |
|---|---|---|
| **Dummy** | Fills a parameter. Never used. | The signature needs an argument, the test does not care |
| **Stub** | Returns fixed answers | The test needs controlled **input** from a dependency |
| **Spy** | A stub that records calls | You check **after** the Act what was called |
| **Mock** | Has expectations set **before** the Act. Fails if they are not met. | The **interaction** is the requirement |
| **Fake** | A simple but working implementation (in-memory store, fake clock) | The real thing is slow, has I/O, or is not deterministic |

Rule: **state verification** (check the result) over **interaction verification** (check the calls). Use a mock only when the call is the behavior.

## When to replace a dependency

Replace it when it is slow, not deterministic (time, random, threads), hard to set up (hardware, driver), dangerous or costly (actuators, payments, outgoing commands), or not yet available.

Do **not** replace: value types, simple data structures, pure fast functions, standard library types, or the unit under test.

## Classical vs mockist

| | Classical (state-based) | Mockist (interaction-based) |
|---|---|---|
| Collaborators | Real, when cheap | Mocks |
| Verification | State, result | Calls |
| Strength | Tests survive refactoring | Drives role-based design |
| Weakness | A failure can come from a collaborator | Tests tied to the implementation |

Default: **classical**. Use doubles at the **boundaries**: hardware, OS, network, clock, other processes, other components with their own tests.

## Rules

1. **Do not mock what you do not own.** Wrap the third-party type in a thin adapter. Replace the adapter.
2. **Do not mock value objects.** Build real ones with a builder.
3. **Do not over-specify.** Check only what the requirement states.
4. **Keep doubles simple.** A fake with logic needs its own tests.
5. **Make a double behave like the real thing.** Run the same **contract tests** on the fake and the real class (below).
6. **One level of doubles.** A mock that returns a mock means too much coupling.
7. **Inject** dependencies. Avoid global state and singletons.

## C++ seams: how to inject a double

| Technique | Use when | Cost |
|---|---|---|
| **Constructor injection of an abstract interface** (pure virtual class, virtual destructor) | Default choice | A virtual call |
| **Template parameter** (policy type) | Real-time code where a virtual call or heap use is not wanted | Code in headers. Longer compile time. |
| **Function object** (`std::function` or a lighter callable) | A single callback | `std::function` may allocate. Check for real-time code. |
| **Link seam** (test links a fake object file or library with the same symbols) | C APIs, legacy code | Build work. No source change. |
| **Preprocessor seam** (`#ifdef`) | Last resort for legacy code | Hard to maintain. Ask the user first. |

Rules:

- Inject through the constructor. Do not use globals or singletons in tests. If production code needs one, ask the user before you change production code.
- Keep the interface small. One role per interface.
- In real-time code, do not add a virtual interface only for tests without checking the cost. Prefer a template seam.
- Wrap hardware access, OS calls, and third-party drivers in an adapter. Test the adapter at integration or HIL level. Test everything above it with fakes.
- More seam types and legacy moves: `tdd-and-legacy.md`.

## Control time, random, and environment

| Source | Technique |
|---|---|
| Time | Inject a clock. Move it by hand. Never `sleep`. |
| Random | Inject the generator or a seed. Print the seed. |
| Environment variables, locale, files | Set and restore in the fixture |
| Network and serial | Fake endpoint or in-memory pipe |
| Threads | Deterministic step mode, latches, or single-thread logic |
| IDs and counters | Inject a generator |
| Hardware registers | Adapter interface with a fake |
| Allocation | Fake allocator that counts or fails on demand |

### Fake clock

```cpp
class FakeClock final : public IClock {
public:
    TimePoint Now() const override { return now_; }
    void SetTime(TimePoint t) { now_ = t; }
    void Advance(Duration d) { now_ += d; }
private:
    TimePoint now_{};
};
```

Advance time by hand. Step the unit after each advance. Test wrap-around and backward jumps if the real clock can do that. More: `../practice/determinism-and-realtime.md`.

### Fake for message input and output

- An in-memory queue or recorder. The test can **push** messages (timestamp, counter, checksum) and **read** all sent messages.
- It must follow the real rules that matter (order, loss, delay).

## Contract tests for doubles

Write one test suite for an **interface**. Run it against the real class and the fake.

```cpp
template <typename T>
class MessageQueueContract : public ::testing::Test {
protected:
    T queue_;
};

using Implementations = ::testing::Types<RealMessageQueue, FakeMessageQueue>;
TYPED_TEST_SUITE(MessageQueueContract, Implementations);

TYPED_TEST(MessageQueueContract, KeepsFifoOrder)
{
    this->queue_.Push(1);
    this->queue_.Push(2);
    EXPECT_EQ(this->queue_.Pop(), 1);
    EXPECT_EQ(this->queue_.Pop(), 2);
}
```

## Signs of too many doubles

| Sign | Fix |
|---|---|
| Most lines are mock setup | Use a fake or the real object |
| A small refactoring breaks many tests | Test behavior through the public interface |
| Tests pass but integration fails | Add integration and contract tests |
| Mock returns a mock | Simplify the design |
| The test repeats the production code | Check the result, not the steps |
