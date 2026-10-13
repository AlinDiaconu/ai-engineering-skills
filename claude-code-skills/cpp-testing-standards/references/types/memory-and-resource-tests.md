# Memory and resource tests (C++)

## What

Tests that check **how code uses memory and other resources**: leaks, invalid access, lifetime, allocation in the cyclic path, stack and heap size, handle and file counts, and clean-up on every exit path.

## Why

- Memory errors are a main source of crashes and security defects in C++.
- Real-time code may **not allocate** after start-up, and has fixed memory limits.
- Resource leaks cause failures only after a long run.

## When

- Classes that own a resource (memory, file, socket, handle, lock, hardware).
- The cyclic (real-time) path.
- Code with manual memory management, custom allocators, or pools.
- Long-running services.
- Any code that handles buffers and indexes.

## Tools

| Tool | Finds | Note |
|---|---|---|
| **AddressSanitizer (ASan)** | Out-of-bounds, use-after-free, double free, stack and global overflow | Default. Fast. |
| **LeakSanitizer (LSan)** | Leaks | Part of ASan on Linux |
| **UndefinedBehaviorSanitizer (UBSan)** | Misaligned access, bad shifts, overflow, null deref | Run in a separate build |
| **MemorySanitizer (MSan)** | Use of uninitialized memory | Clang. Needs an instrumented standard library. |
| **Valgrind memcheck** | Similar to ASan and MSan. Slow. | When sanitizers are not available |
| **Valgrind massif / heaptrack** | Heap use over time | Footprint |
| **Custom `operator new` counter** | Allocation in a region | For "no heap in the cyclic path" |
| **`-fstack-usage`, `-Wstack-usage`, `-Wframe-larger-than`** | Stack size per function | Compile time |
| **Stack watermark on target** | Real stack use | HIL or target |
| **Static analysis (clang-tidy, cppcheck)** | Lifetime and ownership problems | Use with tests |

## How

### 1. Sanitizer builds

- Build and run the whole unit suite with **ASan+UBSan**, and separately with **TSan** (see `../tooling/build-and-run.md`).

### 2. Ownership and RAII tests

Use a counting fake to prove that every acquire has a release:

```cpp
struct CountingResource {
    static inline int alive = 0;
    CountingResource()  { ++alive; }
    CountingResource(const CountingResource&) { ++alive; }
    CountingResource(CountingResource&&) noexcept { ++alive; }
    ~CountingResource() { --alive; }
};

TEST(Pool, ReleaseOnAllExitPaths)   // Verifies: REQ-XXX
{
    ASSERT_EQ(CountingResource::alive, 0);
    {
        Pool<CountingResource> pool;
        pool.Add(CountingResource{});
        EXPECT_THROW(pool.AddAndFail(), PoolError);   // exception path
    }
    EXPECT_EQ(CountingResource::alive, 0);
}
```

Test the exits: normal return, early return, **exception**, move-from, copy, assignment, self-assignment, destroy.

### 3. No allocation in the cyclic path

```cpp
// In the test binary only:
static std::atomic<bool> g_armed{false};
static std::atomic<int>  g_allocs{0};

void* operator new(std::size_t n)
{
    if (g_armed.load()) { g_allocs.fetch_add(1); }
    if (void* p = std::malloc(n)) { return p; }
    throw std::bad_alloc{};
}
void operator delete(void* p) noexcept { std::free(p); }
void operator delete(void* p, std::size_t) noexcept { std::free(p); }

TEST(CyclicPath, NoHeapAllocationAfterInit)   // Verifies: REQ-XXX
{
    App app{MakeConfig()};
    app.Init();
    app.Step();                 // warm up

    g_allocs = 0;
    g_armed = true;
    for (int i = 0; i < 1000; ++i) { app.Step(); }
    g_armed = false;

    EXPECT_EQ(g_allocs.load(), 0);
}
```

- Put the replacement `operator new` in a **separate test executable**, so it does not affect other tests.
- Add the array and aligned versions of `new` and `delete` when the code uses them.
- This does not see `malloc` calls from C libraries. Use a `malloc` hook for those.

### 4. Bounds and buffers

- Test sizes: 0, 1, size−1, size, size+1, very large.
- Test copy into a fixed buffer with the exact size and one byte more.
- Test index wrap and off-by-one with ring buffers and pools.
- Run with ASan so a silent overflow becomes a failure.

### 5. Limits and exhaustion

- Pool or queue **full**: the defined reaction.
- Allocation **failure** (`bad_alloc`, or null from a pool): inject with a fake allocator.
- File handle, thread, or socket limits (integration level).

### 6. Long-run checks (soak)

- Run many cycles. Check that memory use and handle counts stay **flat**.
- Run under `massif` or a memory monitor, or log RSS over time.

### 7. Footprint

- Check static memory (`size`, linker map), stack use (`-fstack-usage`, watermark), heap peak.
- Compare to the budget. Report the margin.

## Best practices

- **Prefer RAII and value types.** No owning raw pointers, no manual `new` and `delete`.
- **Test the Rule of 0/3/5:** copy, move, assign, destroy of owners.
- **Test exception safety** (basic and strong guarantee) by making collaborators throw.
- **Test lifetime:** no dangling reference after a move, a reallocation, or a destroy of the owner.
- **Initialize everything.** Use MSan or Valgrind to confirm.
- **Run sanitizers in CI** at each merge, on host.
- **Use a fake allocator** to inject failure and count calls.
- **Treat a leak report as a defect.** Do not suppress leaks without a reason and approval.
- **Do not claim** "no memory errors". Say which paths the sanitizers saw.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Sanitizers only on the developer's machine | Run in CI |
| `operator new` hook affects all tests | Use a separate test binary |
| Static or global objects cause leak noise | Use function-local statics. Clean up. Use a minimal suppression with approval. |
| Allocation inside `std::string` or `std::vector` not seen | Count `new` calls. Review those types in the cyclic path. |
| Test too short to show a leak | Add a long-run test |
| Host results used for target stack claims | Measure on target |

## Checklist

- [ ] ASan, UBSan, and (if used) MSan runs are clean.
- [ ] Owners have copy, move, exception, and destroy tests.
- [ ] Cyclic path: allocation count is zero after init.
- [ ] Limits and failure of pools, queues, and allocators are tested.
- [ ] Footprint (heap, stack, static) compared to the budget.
