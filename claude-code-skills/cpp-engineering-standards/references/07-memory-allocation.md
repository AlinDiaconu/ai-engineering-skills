# 07 — Allocation strategies, allocators, no-allocation policy, stack, data layout

Ownership, RAII, smart pointers, and lifetime rules are in `06-memory-ownership-lifetime.md`.

## 1. How heap allocation works (and why it is a problem)

A general-purpose allocator (`malloc`/`operator new`) must handle any size, any order of
allocation and release, and any thread. The price:

| Cost | Typical value | Consequence |
|---|---|---|
| Fast path (thread cache hit) | 10-50 ns | Acceptable outside hot loops |
| Slow path (central heap, lock, `mmap`/`brk` system call) | µs to ms | Unbounded jitter — not acceptable in real-time code |
| Metadata per block | 8-16 bytes; minimum block 16-32 bytes (glibc 64-bit: 32) | Many small allocations waste memory |
| Internal fragmentation | Rounding up to size classes | Up to ~25 % waste |
| External fragmentation | Free memory split into small holes | Large allocation fails although enough total memory is free |
| Cache/TLB misses | Objects spread across memory | Slow traversal |
| Lock contention | Several threads allocate at once | Poor scaling (modern allocators use thread caches) |

Common allocators: glibc ptmalloc2 (default Linux), jemalloc, tcmalloc (Google), mimalloc
(Microsoft), snmalloc, rpmalloc; real-time allocators: TLSF (O(1), bounded fragmentation), o1heap. Replacing the allocator (link
or `LD_PRELOAD`) can give 10-30 % in allocation-heavy server code without code changes.

**Determinism rule**: real-time code must not call a general-purpose allocator after
initialization. Even "fast" allocators have a non-deterministic slow path.

## 2. `operator new` / `operator delete` in depth

| Form | Use |
|---|---|
| `new T` / `delete p` | Single object |
| `new T[n]` / `delete[] p` | Array (prefer containers) |
| `new (std::nothrow) T` | Returns `nullptr` instead of throwing `std::bad_alloc` — check the result |
| `new (ptr) T` (placement) | Construct in existing storage; no allocation |
| `operator new(size, std::align_val_t)` (C++17) | Over-aligned types (`alignas` > `__STDCPP_DEFAULT_NEW_ALIGNMENT__`, usually 16) — used automatically by `new` |
| Sized `operator delete(void*, std::size_t)` (C++14) | Lets the allocator skip a size lookup |
| Class-specific `T::operator new/delete` | Route one type to a pool |
| Global replacement | Route all allocations (counting, trapping, custom heap) |
| `std::set_new_handler` | Called on allocation failure (free cache memory, log, terminate) |

Rules:
- With `-fno-exceptions`, a failing `new` cannot throw: depending on the toolchain it calls
  `abort`/`std::terminate` or returns null. Know your toolchain's behavior; in real-time code,
  avoid run-time `new` completely.
- A class-specific `operator new` also needs the matching `operator delete` (and array forms if used).
- Global replacement must implement all forms consistently (normal, nothrow, aligned, sized, array).

## 3. Allocation strategies — detailed catalog

| Strategy | Allocation | Free | Fragmentation | Deterministic | Thread-safe | Best for |
|---|---|---|---|---|---|---|
| Static (global/member arrays) | Compile/link time | Never | None | Yes | N/A | Fixed number of long-lived objects |
| Stack (automatic) | O(1) | Scope end | None | Yes | Per thread | Small, short-lived objects |
| Arena / linear / bump | O(1): move a pointer | All at once (`reset`) | None inside a cycle | Yes | No (one per thread) | Per-frame, per-cycle, per-request temporary data |
| Stack allocator (LIFO) | O(1) | O(1), only in reverse order | None | Yes | No | Nested temporary allocations |
| Fixed-size pool (free list) | O(1) | O(1) | None (one size) | Yes | With lock or per thread | Many objects of one type |
| Multi-size pools (size classes) | O(1) | O(1) | Internal only | Yes | With lock | Mixed small sizes |
| Slab | O(1) | O(1) | Low | Yes | Per CPU | Kernel-style object caches |
| Buddy allocator | O(log n) | O(log n) + merge | Internal (power of 2) | Bounded | With lock | Variable-size blocks, page allocators |
| TLSF | O(1) | O(1) + merge | Bounded | Yes | With lock | Real-time general allocation |
| Ring buffer | O(1) | O(1), FIFO order | None | Yes | SPSC lock-free possible | Queues, logs, streaming data |
| Double/triple buffer | None at run time | None | None | Yes | Single writer/reader | Latest-value exchange between tasks |
| General heap | Variable | Variable | Yes | No | Yes | Non-real-time code, unknown sizes |

### 3.1 Arena (monotonic / bump) allocator

```cpp
/// Linear allocator over a caller-provided buffer. Not thread-safe. Release is all-at-once.
class Arena
{
public:
  explicit Arena(std::span<std::byte> buffer) noexcept : m_buffer{buffer} {}

  /// @return Aligned memory, or nullptr when the arena is full.
  [[nodiscard]] void* allocate(std::size_t size, std::size_t alignment = alignof(std::max_align_t)) noexcept
  {
    const auto base = reinterpret_cast<std::uintptr_t>(m_buffer.data());
    const std::uintptr_t current = base + m_offset;
    const std::uintptr_t aligned = (current + (alignment - 1U)) & ~(static_cast<std::uintptr_t>(alignment) - 1U);
    const std::size_t newOffset = static_cast<std::size_t>(aligned - base) + size;
    if (newOffset > m_buffer.size())
    {
      return nullptr;
    }
    m_offset = newOffset;
    m_highWater = std::max(m_highWater, m_offset);
    return m_buffer.data() + (aligned - base);
  }

  template <typename T, typename... Args>
  [[nodiscard]] T* create(Args&&... args)   // only for trivially destructible T: no destructor runs on reset
  {
    static_assert(std::is_trivially_destructible_v<T>, "Arena::create needs trivially destructible types");
    void* const p = allocate(sizeof(T), alignof(T));
    return (p != nullptr) ? ::new (p) T(std::forward<Args>(args)...) : nullptr;
  }

  void reset() noexcept { m_offset = 0U; }
  [[nodiscard]] std::size_t used() const noexcept { return m_offset; }
  [[nodiscard]] std::size_t highWater() const noexcept { return m_highWater; }   // for budget reports

private:
  std::span<std::byte> m_buffer;
  std::size_t m_offset{0U};
  std::size_t m_highWater{0U};
};
```
`alignment` must be a power of two. Track the high-water mark at run time to size the buffer.

### 3.2 Fixed-size object pool with generation handles

Handles (index + generation) instead of pointers detect use-after-free: a stale handle has an old
generation and is rejected.

```cpp
template <typename T, std::size_t Capacity>
class ObjectPool
{
public:
  struct Handle
  {
    std::uint32_t index{kInvalidIndex};
    std::uint32_t generation{0U};
  };

  ObjectPool() noexcept
  {
    for (std::size_t i = 0U; i < Capacity; ++i)
    {
      m_slots[i].nextFree = static_cast<std::uint32_t>(i + 1U);
    }
  }
  ~ObjectPool()
  {
    for (auto& slot : m_slots)
    {
      if (slot.alive) { std::destroy_at(slot.ptr()); }
    }
  }
  ObjectPool(const ObjectPool&) = delete;
  ObjectPool& operator=(const ObjectPool&) = delete;

  template <typename... Args>
  [[nodiscard]] std::optional<Handle> create(Args&&... args)
  {
    if (m_freeHead >= Capacity)
    {
      return std::nullopt;                                    // pool exhausted: caller handles it
    }
    const std::uint32_t index = m_freeHead;
    Slot& slot = m_slots[index];
    m_freeHead = slot.nextFree;
    ::new (static_cast<void*>(slot.storage)) T(std::forward<Args>(args)...);
    slot.alive = true;
    return Handle{index, slot.generation};
  }

  void destroy(Handle h) noexcept
  {
    if (T* const p = get(h))
    {
      Slot& slot = m_slots[h.index];
      std::destroy_at(p);
      slot.alive = false;
      ++slot.generation;                                      // invalidates every old handle
      slot.nextFree = m_freeHead;
      m_freeHead = h.index;
    }
  }

  [[nodiscard]] T* get(Handle h) noexcept
  {
    if ((h.index >= Capacity) || !m_slots[h.index].alive || (m_slots[h.index].generation != h.generation))
    {
      return nullptr;                                         // stale or invalid handle
    }
    return m_slots[h.index].ptr();
  }

private:
  static constexpr std::uint32_t kInvalidIndex{std::numeric_limits<std::uint32_t>::max()};
  struct Slot
  {
    alignas(T) std::byte storage[sizeof(T)];
    std::uint32_t generation{0U};
    std::uint32_t nextFree{kInvalidIndex};
    bool alive{false};
    [[nodiscard]] T* ptr() noexcept { return std::launder(reinterpret_cast<T*>(storage)); }
  };
  std::array<Slot, Capacity> m_slots{};
  std::uint32_t m_freeHead{0U};
};
```
O(1) create/destroy, no heap, no fragmentation, stale-handle detection. Add a mutex or one pool
per thread for concurrent use. Libraries: Boost.Pool.

### 3.3 Choosing a strategy

1. Is the number and size of objects known at design time? → static allocation.
2. Do all temporary objects of a cycle/frame/request die together? → arena, reset per cycle.
3. Many objects of one type created and destroyed at run time? → object pool.
4. Producer/consumer data flow? → ring buffer / bounded queue.
5. Latest-value exchange between tasks? → double/triple buffer.
6. Variable sizes in real-time code? → TLSF or size-class pools, sized from measured high-water marks.
7. None of the above and not real-time? → general heap (consider a better `malloc`).

## 4. The standard allocator model

Containers take an allocator type parameter. A minimal C++17 allocator:

```cpp
template <typename T>
class ArenaAllocator
{
public:
  using value_type = T;

  explicit ArenaAllocator(Arena& arena) noexcept : m_arena{&arena} {}
  template <typename U>
  ArenaAllocator(const ArenaAllocator<U>& other) noexcept : m_arena{other.arena()} {}   // rebind

  [[nodiscard]] T* allocate(std::size_t n)
  {
    if (n > (std::numeric_limits<std::size_t>::max() / sizeof(T)))
    {
      throw std::bad_array_new_length{};                     // or a project fault handler without exceptions
    }
    void* const p = m_arena->allocate(n * sizeof(T), alignof(T));
    if (p == nullptr)
    {
      throw std::bad_alloc{};
    }
    return static_cast<T*>(p);
  }
  void deallocate(T*, std::size_t) noexcept {}                // arena frees everything at reset

  [[nodiscard]] Arena* arena() const noexcept { return m_arena; }
  template <typename U>
  bool operator==(const ArenaAllocator<U>& other) const noexcept { return m_arena == other.arena(); }

private:
  Arena* m_arena;
};
```
- `std::allocator_traits` fills in everything else (`construct`, `destroy`, `rebind`, ...).
- Propagation traits (`propagate_on_container_copy_assignment`, `..._move_assignment`,
  `..._swap`, `is_always_equal`) define what happens when containers with different allocators
  are assigned or swapped. Stateful allocators make these important.
- The allocator is part of the container **type** (`std::vector<int, ArenaAllocator<int>>` ≠
  `std::vector<int>`). This spreads into interfaces. `pmr` (section 5) solves this.

## 5. Polymorphic memory resources (`std::pmr`, C++17)

`std::pmr::polymorphic_allocator` holds a pointer to a `std::pmr::memory_resource` (virtual
`do_allocate`/`do_deallocate`). All `std::pmr::vector<T>` have the same type, whatever resource
they use.

| Resource | Behavior | Use |
|---|---|---|
| `std::pmr::new_delete_resource()` | `operator new`/`delete` | Default |
| `std::pmr::null_memory_resource()` | Always throws `std::bad_alloc` | Upstream that forbids fallback to the heap |
| `std::pmr::monotonic_buffer_resource` | Bump allocation from a buffer; `deallocate` does nothing; memory released at destruction/`release()` | Per-cycle temporary data |
| `std::pmr::unsynchronized_pool_resource` | Pools per size class; single thread | Mixed sizes, reuse, one thread |
| `std::pmr::synchronized_pool_resource` | Same, thread-safe | Mixed sizes, several threads |
| Custom resource (derive from `memory_resource`) | Anything | Counting, limits, specific RAM regions |

Rules and pitfalls:
- `std::pmr::set_default_resource()` changes the process-wide default — useful to forbid
  hidden allocation (set it to a counting or null resource).
- Nested `pmr` containers propagate the resource to their elements (`std::pmr::vector<std::pmr::string>`):
  uses-allocator construction. A `std::pmr::vector<std::string>` does **not** — the strings use the heap.
- The resource must outlive every container that uses it.
- Copying a `pmr` container gives the copy the **default** resource, not the source's (allocator is
  not propagated on copy construction). Move keeps the resource.
- Cost: one virtual call per allocation (usually negligible compared with the savings).

```cpp
// Counting + limit resource for memory budgets.
class BudgetResource final : public std::pmr::memory_resource
{
public:
  BudgetResource(std::size_t limitBytes, std::pmr::memory_resource* upstream) noexcept
    : m_limit{limitBytes}, m_upstream{upstream} {}

  [[nodiscard]] std::size_t inUse() const noexcept { return m_inUse; }
  [[nodiscard]] std::size_t peak() const noexcept { return m_peak; }
  [[nodiscard]] std::size_t allocations() const noexcept { return m_count; }

private:
  void* do_allocate(std::size_t bytes, std::size_t alignment) override
  {
    if ((m_inUse + bytes) > m_limit)
    {
      throw std::bad_alloc{};                                // budget exceeded: a design error
    }
    void* const p = m_upstream->allocate(bytes, alignment);
    m_inUse += bytes;
    m_peak = std::max(m_peak, m_inUse);
    ++m_count;
    return p;
  }
  void do_deallocate(void* p, std::size_t bytes, std::size_t alignment) override
  {
    m_upstream->deallocate(p, bytes, alignment);
    m_inUse -= bytes;
  }
  [[nodiscard]] bool do_is_equal(const std::pmr::memory_resource& other) const noexcept override
  {
    return this == &other;
  }

  std::size_t m_limit;
  std::pmr::memory_resource* m_upstream;
  std::size_t m_inUse{0U};
  std::size_t m_peak{0U};
  std::size_t m_count{0U};
};
```

## 6. No-allocation and real-time memory policy

Applies to real-time and low-latency code (audio, games, trading, robotics, control loops,
device software) and to safety-critical code. The goal: memory use is bounded, known in
advance, and has no unpredictable timing.

### 6.1 Lifecycle policy

| Phase | Allowed |
|---|---|
| Start-up / initialization | Static construction, pool and container sizing, one-time allocation if the project allows |
| Run (operational) | No heap allocation, no deallocation; only pools, arenas, ring buffers, stack, statics |
| Shutdown (if any) | Release in reverse order |

### 6.2 Enforce "no heap after init"

```cpp
// Global replacement: count allocations; trap after initialization. Link only into the real-time binary.
namespace mem
{
  inline std::atomic<bool> g_runPhase{false};          // set to true when initialization ends
  inline std::atomic<std::size_t> g_allocCount{0U};
}

void* operator new(std::size_t size)
{
  if (mem::g_runPhase.load(std::memory_order_relaxed))
  {
    faultHandler(Fault::HeapUseAfterInit);               // project fault reaction; declared [[noreturn]]
  }
  mem::g_allocCount.fetch_add(1U, std::memory_order_relaxed);
  if (void* const p = std::malloc(size == 0U ? 1U : size))
  {
    return p;
  }
  faultHandler(Fault::OutOfMemory);
}
void operator delete(void* p) noexcept { std::free(p); }
void operator delete(void* p, std::size_t) noexcept { std::free(p); }
// Also replace: new[]/delete[], nothrow, and std::align_val_t forms, or link-fail them.
```
Other enforcement methods:
- No calls to `malloc`, `operator new`, or code that allocates implicitly (exceptions, `std::function` with large captures, growing containers, `std::string` growth) in run-phase code.
- With `pmr`: `std::pmr::set_default_resource(std::pmr::null_memory_resource())` after initialization, so that any hidden allocation through the default resource fails at once.
- Real-time threads hand objects that must be freed to a non-real-time thread (deallocation can also block).

### 6.3 Memory budget

- Keep a memory budget per component: static data, stack per thread, pool and arena capacities.
- Size pools and arenas from measured high-water marks plus a margin; document the numbers next to the constants.
- Define what happens when a pool or arena is full (error return, drop with a counter, degraded mode) — never undefined behavior.
- Pools must detect misuse, not only exhaustion: a double release, a release of an item that is not
  checked out, a release to the wrong pool, and use after release. Use handles with a generation
  counter (section 3.2), or an "in use" flag per slot checked on every release. Never accept a
  raw pointer back without validation.
- A pool must not be destroyed while handles are outstanding. Count checked-out items and
  assert/terminate in the pool destructor when the count is not zero, or give handles shared
  ownership of the pool storage. A handle that outlives its pool is a use-after-free.
- Reset recycled items: clear or reinitialize a slot on release or on acquire (ids, lengths,
  flags). Always clear slots that held sensitive data. Document what state an acquired item has.
- A dropped handle (acquired, never released) is a leak of a slot. Prevent it with RAII release,
  or detect it (debug assertion in the handle destructor, leak counter) when RAII release is not
  allowed on that thread (see `10-concurrency.md` section 5).

## 7. The stack in depth

- Each thread has a fixed-size stack (typical defaults: 8 MiB for the main thread on Linux,
  1 MiB on Windows; worker threads and small devices can have much less).
- A stack overflow crashes the program, or silently corrupts memory on systems without guard pages.
- Rules:
  - No unbounded recursion. No recursion at all in real-time and safety-critical code.
  - No large local objects: move large buffers to members, static storage, or pools.
  - No VLAs (not standard C++) and no `alloca`.
  - Watch hidden stack use: large `std::array` locals, `std::function` small buffers, exceptions,
    `printf`-family functions, deep call chains, inlining that merges frames.
  - Set the stack size of a thread explicitly when the default is not known to be enough.
- Know the worst-case stack depth of real-time threads (compiler stack-usage reports, high-water
  marks), and keep a margin.

## 8. Data layout, alignment, and padding

This section covers layout correctness. Layout for speed (SoA/AoS, hot/cold split, indexes
instead of pointers, false sharing, cache latencies) is in `08-performance-optimization.md` section 3.

### 8.1 Alignment rules

- Every type has an alignment (`alignof(T)`); its address must be a multiple of it.
- Struct alignment = largest member alignment; size is rounded up to a multiple of it.
- Misaligned access is UB in C++; on some CPUs it faults (bus error), on others it is slow.
- `alignas(N)` raises alignment (power of two). Over-aligned types need aligned `new` (automatic since C++17).
- `std::assume_aligned<N>(p)` (C++20) tells the optimizer a pointer is aligned — UB if false.

### 8.2 Padding and member order

```cpp
struct Bad            // 24 bytes on 64-bit: 1 + 7 padding + 8 + 1 + 3 padding + 4
{
  std::uint8_t  flag;
  double        value;
  std::uint8_t  mode;
  std::uint32_t count;
};

struct Good           // 16 bytes: 8 + 4 + 1 + 1 + 2 padding
{
  double        value;
  std::uint32_t count;
  std::uint8_t  flag;
  std::uint8_t  mode;
};
static_assert(sizeof(Good) == 16U);
```
- Order members from largest to smallest alignment (unless grouping by access pattern matters more).
- Check sizes with `static_assert` for types used in arrays, messages, or shared memory.

### 8.3 Packed structs and wire formats

- `#pragma pack` / `[[gnu::packed]]` removes padding but creates misaligned members: taking their
  address or reference is dangerous (UB/faults on strict-alignment CPUs).
- Better: keep internal structs naturally aligned and **serialize** explicitly into byte buffers
  with defined byte order (`std::byteswap`, `std::endian`, shift/mask), or use a schema tool
  (protobuf, FlatBuffers, Cap'n Proto, ASN.1).
- Bit-fields: layout and bit order are implementation-defined → do not use them for hardware
  registers or wire formats. Use masks and shifts.

## 9. Shared memory and zero-copy

- **Shared memory between processes/cores**: use only trivially copyable, fixed-layout types
  (no pointers, no `std::string`, no virtual functions); use offsets instead of pointers; use
  `std::atomic` with lock-free types for synchronization; start lifetimes with `std::start_lifetime_as`
  or by construction in place; version the layout.
- **Zero-copy**: pass views (`std::span`) or ownership of buffers (pool handles, `unique_ptr`
  with pool deleter) instead of copying payloads. Many messaging frameworks use this pattern.
- **Memory-mapped files** (`mmap`): fast large reads; handle `SIGBUS` on truncated files; use RAII for the mapping.

## 10. Allocation and memory layout checklist

- [ ] The allocation strategy fits the lifetime pattern (static, arena, pool, ring buffer, heap).
- [ ] No general-purpose heap use in real-time paths or after initialization (REAL-TIME/SAFETY-CRITICAL); enforcement exists.
- [ ] Pools/arenas have defined capacities, exhaustion handling, and measured high-water marks.
- [ ] Custom allocators/resources are correct: alignment, overflow check on `n * sizeof(T)`, matching deallocation, outlive their containers.
- [ ] `pmr` containers propagate the resource where intended (nested `pmr` types).
- [ ] Stack: no recursion, no large frames, no VLAs/`alloca`; worst-case stack depth of real-time threads known.
- [ ] Data layout: correct alignment; no unnecessary padding in large arrays; `static_assert` on sizes of shared/wire types; no packed structs accessed by reference.
- [ ] Shared memory: fixed layout, trivially copyable types, no pointers in shared data.
- [ ] Memory budget (static data, stacks, pools, arenas) documented; full-pool behavior defined.
