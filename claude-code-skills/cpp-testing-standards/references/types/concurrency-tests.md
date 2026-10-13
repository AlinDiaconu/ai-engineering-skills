# Concurrency tests (C++)

## What

Tests for code that uses **threads, locks, atomics, queues, or callbacks from other threads**. They look for data races, deadlocks, lost updates, wrong ordering, and unsafe shutdown.

## Why

- Concurrency defects are **rare, random, and hard to reproduce**.
- A passing run does **not** prove that the code is correct. Tests raise the chance of finding defects. Design and review are also needed.

## When

- Shared data between threads or tasks.
- Locks, condition variables, atomics, lock-free queues.
- Thread pools, async tasks, callbacks, timers.
- Start-up and shutdown of worker threads.

## How

### Step 1: separate logic from threading

- Put the **logic** in a class that runs in **one thread** and is driven by calls (`Step()`, `Handle(event)`).
- Put the **threading** in a thin wrapper.
- Test the logic with normal unit tests (deterministic).
- Test the wrapper with the concurrency tests below.

### Step 2: stress and sanitize

1. Write a test that runs the code from **several threads** with many iterations.
2. Build with **ThreadSanitizer** (`-fsanitize=thread`).
3. Run **many times** (`--gtest_repeat=100`) and under **CPU load**.
4. A TSan report is a **test failure**.

```cpp
TEST(SharedCounter, ConcurrentIncrements_NoLostUpdates)   // Verifies: REQ-XXX
{
    SharedCounter counter;
    constexpr int kThreads = 8;
    constexpr int kPerThread = 10000;

    std::vector<std::thread> threads;
    for (int t = 0; t < kThreads; ++t) {
        threads.emplace_back([&] {
            for (int i = 0; i < kPerThread; ++i) { counter.Increment(); }
        });
    }
    for (auto& th : threads) { th.join(); }

    EXPECT_EQ(counter.Value(), kThreads * kPerThread);
}
```

### Step 3: force interleavings (do not hope)

- Use **latches and barriers** (`std::latch`, `std::barrier`) to start threads at the same time.
- Use **promises and futures** or **condition variables** to make a thread wait for an exact point.
- Use **test hooks** (injected callbacks) at the critical points, when the design allows it.
- **Never use `sleep`** to force an order.

```cpp
std::latch ready{kThreads};
std::latch go{1};
// each worker: ready.count_down(); go.wait(); then act
ready.wait();
go.count_down();
```

### Step 4: no hangs

- Put a **timeout** on every wait in a test (`wait_for`, `future.wait_for`, CTest `TIMEOUT`).
- A deadlock must **fail** the test, not hang the build.

## What to test

| Area | Cases |
|---|---|
| Data races | Concurrent read and write of shared data (TSan) |
| Atomicity | No lost updates, no torn reads |
| Deadlock | Lock order, nested locks, callbacks under lock |
| Liveness | Every request is eventually served. No starvation. |
| Ordering | Message order is kept when the contract says so |
| Start and stop | Stop while busy, stop twice, stop before start, destroy while running |
| Cancellation | Cancel at each step |
| Queue limits | Full queue, empty queue, wait and wake-up, producers faster than consumers |
| Callbacks | Called from the right thread. No call after unregister. |
| Memory ordering | Lock-free code: use TSan and a model checker or formal review. Tests alone are not enough. |

## Tools

| Tool | Use |
|---|---|
| **ThreadSanitizer** | Data races, lock-order inversion |
| **Helgrind / DRD** (Valgrind) | Races and lock errors. Slow. |
| **ASan** | Use-after-free after a bad shutdown |
| **`stress-ng`, `taskset`** | CPU load and pinning to change timing |
| **CDSChecker, Relacy, CppMem** | Memory-model checking for lock-free code |
| **Static analysis** (clang thread-safety annotations, `-Wthread-safety`) | Check lock rules at compile time |
| **Deterministic scheduler or simulation** | Run tasks in a controlled order |

## Best practices

- **Prefer designs that are easy to test:** share little, use message passing, use immutable data, keep one owner per object.
- **Use Clang thread-safety annotations** (`GUARDED_BY`, `REQUIRES`) and `-Wthread-safety` to find lock errors at build time.
- **Run TSan in CI** on the concurrency tests. Fix every report. Do not suppress without a reason and approval.
- **Repeat the tests** many times and with different load. Print the seed if random delays are used.
- **Use a fake clock** for timers and timeouts. Do not wait in real time.
- **Test the shutdown path** as much as the normal path.
- **Keep the tests short**, so you can run them many times.
- **Document the thread-safety guarantee** of each class (not safe, safe for const, fully safe). Test the stated guarantee.
- **Do not claim** that the code is thread-safe because the tests pass. Say what you tested.

## Pitfalls

| Pitfall | Fix |
|---|---|
| `sleep` to wait for a thread | Latch, future, condition variable |
| Test hangs on a deadlock | Timeouts on all waits |
| TSan not run | Add a TSan build |
| Test passes on a fast machine only | Run under CPU load and with fewer cores |
| `volatile` used as a sync tool | Use `std::atomic` or a mutex |
| Real timers in tests | Inject a clock and a timer service |
| Race in the test itself | Review the test code with the same care |

## Checklist

- [ ] Logic is tested single-threaded. Wrapper is tested with threads.
- [ ] TSan build is clean.
- [ ] Interleavings are forced with latches, not sleeps.
- [ ] Every wait has a timeout.
- [ ] Shutdown, double stop, and stop-while-busy are tested.
- [ ] The thread-safety statement is tested.
