# 10 — Concurrency and parallelism

## 1. Principles

- A data race is UB (CG CP.2). Two threads access one memory location, at least one writes, and
  no happens-before relation exists → UB, no matter how "benign" it looks.
- Minimize sharing of writable data (CG CP.3). The best concurrency design has no shared mutable
  state: message passing, ownership transfer, immutable snapshots, per-thread data.
- Think in tasks, not threads (CG CP.4). Use thread pools, executors, job systems.
- Design for thread safety from the start; document for every class:
  - **Thread-safe**: all functions may run at the same time.
  - **Thread-compatible**: const functions may run at the same time; mutations need external sync (the standard library default).
  - **Thread-hostile / single-thread**: only one thread may use it.

## 2. Threads

- `std::jthread` (C++20) joins automatically and supports `std::stop_token` cancellation. Prefer it to `std::thread`.
- Never `detach()` (CG CP.26): lifetime and shutdown become uncontrollable.
- Every `std::thread` is joined on every path (RAII wrapper or `jthread`).
- Catch all exceptions in thread entry functions; an escaping exception calls `std::terminate`.
- Pass arguments by value into threads; `std::ref` only when the referent outlives the thread.
- Real-time: set priority, scheduling policy, CPU affinity, and stack size explicitly (platform API). Create threads at initialization, not at run time.

## 3. Mutexes and locks

- Use RAII locks only (CG CP.20): `std::scoped_lock` (C++17) by default, `std::unique_lock` when you need to unlock early, defer, or use condition variables. Never call `lock()`/`unlock()` by hand.
- Always name the lock object (CG CP.44): `std::scoped_lock lock{m_mutex};`. The unnamed form `std::scoped_lock{m_mutex};` locks and unlocks at once.
- Lock several mutexes with one `std::scoped_lock lock{a, b};` (deadlock-avoidance algorithm) (CG CP.21).
- Define one global lock order and document it if you must hold several locks in different places.
- Keep critical sections short. Never call unknown code (callbacks, virtual calls to user code) while you hold a lock (CG CP.22).
- What else must stay out of a critical section depends on the context:
  - All contexts: no I/O, no blocking calls (sleep, waits on other locks, network), no long computations. Do expensive work (formatting, large copies, parsing) before you take the lock or after you release it.
  - GENERAL: small allocations under a lock are acceptable when they are part of the protected operation (for example a node insert in a thread-safe map); move large copies and bulk allocation outside the lock when possible.
  - REAL-TIME / SAFETY-CRITICAL: no allocation and no deallocation under a lock (and in real-time threads, no locks that a non-real-time thread can hold for an unbounded time).
- Put the mutex next to the data it protects and document the relation (CG CP.50). Clang Thread Safety Analysis annotations (`GUARDED_BY`, `REQUIRES`, `ACQUIRE`) let the compiler check it.
- `std::shared_mutex` for many readers / few writers — only when measured; it is slower than `std::mutex` with low contention.
- `std::recursive_mutex` is a design smell; restructure instead.
- Real-time: use priority-inheritance mutexes (for example `PTHREAD_PRIO_INHERIT`) to avoid priority inversion.

## 4. Condition variables

- Always wait with a predicate (CG CP.42) — spurious wake-ups and lost notifications:
  ```cpp
  std::unique_lock lock{m_mutex};
  m_cv.wait(lock, [this] { return !m_queue.empty() || m_stopRequested; });
  ```
- Change the condition state under the mutex, then notify (notifying after unlocking is allowed and can be faster).
- Use `wait_for`/`wait_until` with timeouts in systems that must detect hangs.
- C++20: `std::condition_variable_any::wait(lock, stop_token, pred)` for cancellable waits.

## 5. Atomics and memory order

- `std::atomic<T>` for single variables shared between threads (flags, counters, pointers).
- Default memory order is `seq_cst`. Use weaker orders (`acquire`/`release`/`relaxed`) only when
  both conditions hold:
  1. **Correctness is argued in writing**: a comment next to each atomic operation says which
     operation it pairs with and what data it publishes or protects.
  2. **There is a reason**: a measured performance need, or a well-known pattern whose ordering is
     standard (SPSC ring buffer, reference counting, publish-once flag).
  Weakly ordered hardware (ARM, POWER) shows errors that x86 hides.
  - `relaxed`: counters/statistics with no ordering need.
  - `release` store + `acquire` load: publish data to another thread.
  - Avoid `consume` (no compiler implements it as specified).
- Check `std::atomic<T>::is_always_lock_free` (C++17) for types used in signal handlers or other asynchronous contexts.
- A set of atomics is not atomic as a group. Use a mutex when several values must change together.
- `std::atomic_ref` (C++20) to operate atomically on non-atomic objects for a phase.
- `std::atomic<std::shared_ptr<T>>` (C++20) for snapshots (often not lock-free).
- `std::atomic::wait`/`notify_one` (C++20) for efficient waiting on an atomic.
- `std::atomic_flag` is the only guaranteed lock-free type.
- Lock-free data structures: only when necessary (CG CP.100). Prefer a proven library (for example Boost.Lockfree, folly, moodycamel queues).
- A custom lock-free structure is acceptable when no approved library fits, under these conditions:
  - Use the simplest correct algorithm (SPSC ring buffer before MPMC; fixed capacity).
  - `static_assert(std::atomic<T>::is_always_lock_free)` for every atomic it uses.
  - Each atomic operation has the written memory-order argument described above.
  - The capacity and index arithmetic are checked (no overflow of `capacity + 1`, power-of-two masks validated).
  - Thread roles are enforced by the interface (next rule), not only by comments.
- **Single-producer / single-consumer roles must be enforced by types.** Give the producer and the consumer separate endpoint objects (for example `Producer` and `Consumer` handles created once from the queue/pool); each endpoint is used by exactly one thread and is non-copyable. A function that only the consumer may call must not be callable from the producer endpoint.
- **Every acquire path needs a legal release path on the same thread.** If a producer can acquire an item (for example a job slot) and then fail or not need it, the producer endpoint must offer a way to return it that keeps the single-producer/single-consumer roles (for example a producer-side free list, or submitting the item as "cancelled" to the consumer).
- RAII handles for items of a role-restricted structure must release through the correct endpoint
  **and on the endpoint's thread**. Binding a handle to its endpoint is not enough if the handle can
  move to another thread. Choose one:
  - Make the handle non-movable (or movable only inside the endpoint's API), so it cannot leave
    its thread.
  - Record the owning thread in the endpoint and assert `std::this_thread::get_id()` in the handle
    destructor and in every endpoint operation (debug builds).
  - Make the handle non-RAII **and** make dropping it a detected error (assert in debug, counter in
    release).
  Document the rule on the handle type.

Example — SPSC queue with type-enforced roles and argued memory orders:

```cpp
/// Bounded single-producer / single-consumer queue. Thread roles are enforced by types:
/// only a Producer can push, only a Consumer can pop, and each endpoint exists once.
template <typename T, std::size_t Capacity>
class SpscQueue
{
  static_assert((Capacity >= 2U) && ((Capacity & (Capacity - 1U)) == 0U), "Capacity: power of two >= 2");
  static_assert(std::atomic<std::size_t>::is_always_lock_free);
  static_assert(std::is_nothrow_move_assignable_v<T> && std::is_nothrow_default_constructible_v<T>);

public:
  class Producer
  {
  public:
    Producer(const Producer&) = delete;
    Producer& operator=(const Producer&) = delete;
    Producer(Producer&&) noexcept = default;
    Producer& operator=(Producer&&) noexcept = default;
    ~Producer() = default;

    [[nodiscard]] bool tryPush(T value) noexcept
    {
      const std::size_t head = m_q->m_head.load(std::memory_order_relaxed);        // only this thread writes head
      const std::size_t tail = m_q->m_tail.load(std::memory_order_acquire);        // pairs with release in tryPop
      if ((head - tail) == Capacity)
      {
        return false;                                                               // full
      }
      m_q->m_slots[head & kMask] = std::move(value);
      m_q->m_head.store(head + 1U, std::memory_order_release);                     // publishes the slot to the consumer
      return true;
    }

  private:
    friend class SpscQueue;
    explicit Producer(SpscQueue& q) noexcept : m_q{&q} {}
    SpscQueue* m_q;
  };

  class Consumer
  {
  public:
    Consumer(const Consumer&) = delete;
    Consumer& operator=(const Consumer&) = delete;
    Consumer(Consumer&&) noexcept = default;
    Consumer& operator=(Consumer&&) noexcept = default;
    ~Consumer() = default;

    [[nodiscard]] std::optional<T> tryPop() noexcept
    {
      const std::size_t tail = m_q->m_tail.load(std::memory_order_relaxed);        // only this thread writes tail
      const std::size_t head = m_q->m_head.load(std::memory_order_acquire);        // pairs with release in tryPush
      if (head == tail)
      {
        return std::nullopt;                                                        // empty
      }
      T value = std::move(m_q->m_slots[tail & kMask]);
      m_q->m_tail.store(tail + 1U, std::memory_order_release);                     // gives the slot back to the producer
      return value;
    }

  private:
    friend class SpscQueue;
    explicit Consumer(SpscQueue& q) noexcept : m_q{&q} {}
    SpscQueue* m_q;
  };

  SpscQueue() = default;
  SpscQueue(const SpscQueue&) = delete;
  SpscQueue& operator=(const SpscQueue&) = delete;
  ~SpscQueue() = default;

  /// Creates the two endpoints. Call once, before the threads start; give each endpoint to one thread.
  [[nodiscard]] std::pair<Producer, Consumer> endpoints() noexcept
  {
    assert(!m_endpointsTaken && "endpoints() must be called once");
    m_endpointsTaken = true;
    return {Producer{*this}, Consumer{*this}};
  }

private:
  static constexpr std::size_t kMask{Capacity - 1U};
  std::array<T, Capacity> m_slots{};
  alignas(64) std::atomic<std::size_t> m_head{0U};   // written by the producer only
  alignas(64) std::atomic<std::size_t> m_tail{0U};   // written by the consumer only
  bool m_endpointsTaken{false};
};
```

Usage: `auto [producer, consumer] = queue.endpoints();` then move `producer` into the producer
thread and `consumer` into the consumer thread. A thread that holds only a `Producer` cannot pop.


## 6. `volatile`

`volatile` is not a synchronization tool (CG CP.8). It does not give atomicity or ordering
between threads. Use it only for memory-mapped hardware registers and for variables changed by
signal handlers (`volatile std::sig_atomic_t`). Many compound operations on `volatile` are
deprecated in C++20.

## 7. Higher-level patterns

| Pattern | Use |
|---|---|
| Message queue (SPSC/MPSC/MPMC) | Decouple producers and consumers; bounded capacity in real-time systems |
| Double/triple buffer | One writer, one reader, latest-value semantics (latest measurement, latest configuration snapshot) |
| Immutable snapshot + atomic pointer swap | Configuration and maps read often, changed rarely |
| Thread pool / executor | Many short tasks |
| `std::async` | Avoid in production: launch policy surprises, blocking futures in destructors |
| Futures/promises | One-shot results between threads |
| `std::latch`, `std::barrier`, `std::counting_semaphore` (C++20) | Phase synchronization and resource counting |
| `std::call_once` / function-local static | Thread-safe lazy initialization |
| Actor / active object | Each object has its own thread and message queue; no shared state |
| Senders/receivers (`std::execution`, C++26) | Structured asynchronous composition, when available |

## 8. Signal handlers and real-time threads

- Signal handlers (and other asynchronous handlers, such as interrupt handlers): keep them short — set a flag or push to a lock-free SPSC queue, then return.
- Share data with a handler only through lock-free `std::atomic` types or `volatile std::sig_atomic_t`.
- Call only async-signal-safe functions in a signal handler: no locks, no allocation, no `printf`, no I/O through the C++ library.
- Real-time threads: no blocking calls, no allocation, no unbounded loops; communicate with other threads through lock-free queues or double buffers.
- Avoid priority inversion and unbounded blocking; know the worst-case response time.

## 9. Common bugs

| Bug | Sign | Fix |
|---|---|---|
| Data race | Random wrong values, rare crashes | Mutex or atomic; remove sharing |
| Deadlock | Threads stop forever | One lock order, `scoped_lock` for several mutexes, no callbacks under lock |
| Lost wake-up | Thread sleeps forever | Wait with predicate; change state under the mutex |
| Unnamed lock guard | No protection | Name the lock |
| Check-then-act | `if (!m.contains(k)) m[k] = v;` across threads | One critical section, or atomic operation (`try_emplace` under lock) |
| Dangling capture | Crash after scope exit | Capture by value; join before scope end |
| Static init race (pre-C++11 patterns) | Double init | Function-local static or `std::call_once` |
| False sharing | Poor scaling | Pad/align per-thread data |
| Priority inversion | Missed deadlines | Priority-inheritance mutex, lock-free handoff |

## 10. Concurrency checklist

- [ ] Thread-safety contract documented for every shared class.
- [ ] Every shared mutable variable is protected by one named mutex or is atomic.
- [ ] All locks are RAII and named; multiple locks in one `scoped_lock` or one documented order.
- [ ] No callbacks, I/O, or blocking under a lock; no allocation under a lock in real-time/safety-critical code; expensive work moved outside the lock.
- [ ] SPSC/role-restricted structures enforce thread roles through separate endpoint types; every acquire path has a legal release path on the same thread.
- [ ] Custom lock-free code: lock-free `static_assert`s, checked capacity arithmetic, written memory-order argument per atomic operation.
- [ ] Every wait has a predicate (and a timeout where hangs must be detected).
- [ ] No `volatile` for synchronization; non-default memory orders have a written argument.
- [ ] Every thread is joined; no `detach()`; exceptions caught in thread entry.
