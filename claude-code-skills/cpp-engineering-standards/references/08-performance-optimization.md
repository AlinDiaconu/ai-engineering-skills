# 08 — Performance and optimization

## 1. Rules of engagement

1. Do not optimize without a reason (CG Per.1) or too early (CG Per.2). But do not write slow
   code by habit either: right algorithm, right container, no needless copies — from the start.
2. Measure. No performance claim without data from an optimized build on representative input
   and, if possible, the target hardware (CG Per.6).
3. Find the bottleneck first with a profiler on an optimized build. Optimize the top item. Measure again.
4. Keep the optimized code correct, tested, and readable. Comment why an unusual form is faster
   and what measurement shows it.
5. Real-time code: optimize the **worst case** (WCET) and jitter, not the average.

Order of impact (largest first): algorithm and complexity → data layout and memory access →
allocation → avoidable work (copies, conversions, recomputation) → branches and dispatch →
micro-optimizations and intrinsics.

## 2. Algorithms and complexity

- Know the complexity of each operation you call in a loop (`std::find` on a vector is O(n);
  `std::map::find` O(log n); `std::unordered_map::find` O(1) average, O(n) worst).
- Prefer standard algorithms (`std::sort`, `std::lower_bound`, `std::nth_element`,
  `std::partial_sort`, `std::partition`); they are tested and fast. C++17 parallel policies
  (`std::execution::par_unseq`) for large data, if the platform supports them.
- For small `n` (< ~50-100), linear search over contiguous data often beats trees and hash maps.
- Avoid quadratic patterns: `erase` in a loop over a vector (use `std::erase_if`), string `+=` in
  nested loops without `reserve`, repeated `find` inside a loop over the same container.
- Hash maps: good hash, `reserve()` ahead, consider open-addressing maps (abseil `flat_hash_map`,
  `boost::unordered_flat_map`, `ankerl::unordered_dense`) when approved — `std::unordered_map` is
  node-based and cache-unfriendly.
- Precompute and cache: lookup tables (`constexpr` generated), memoization for pure functions.

## 3. Data layout and memory access (CG Per.19)

Memory access is the limit of most modern code. Rough latencies on a desktop/server CPU: L1 ~1 ns,
L2 ~4 ns, L3 ~10-20 ns, DRAM ~60-100 ns. One cache miss costs as much as ~100 simple instructions.
Other CPUs have other numbers — measure on the real hardware.

| Technique | When | Effect |
|---|---|---|
| Contiguous arrays (`std::vector`, `std::array`, flat maps) | Default | Prefetch-friendly; fewer misses than pointer chasing (`std::list`, `std::vector<std::unique_ptr<T>>`, trees of small nodes) |
| Sequential access (linear; row-major for C arrays) | Loops over large data | The hardware prefetcher streams the data |
| Structure of Arrays (SoA) | A loop uses few fields of many objects | Less memory traffic; enables SIMD |
| Array of Structures (AoS) | Code uses all fields of one object together | Good locality per object |
| Hot/cold split | Some members are used rarely | Hot data fits in cache |
| Indexes (`std::uint32_t`) instead of pointers | Graphs, large object sets | Half the size on 64-bit; relocatable; serializable |
| Smallest safe type in large arrays (`std::uint16_t` IDs, `float` instead of `double`) | Memory-bound loops | More elements per cache line |
| Separate cache lines for data written by different threads | Per-thread counters, queues | No false sharing |
| `[[no_unique_address]]` (C++20) | Empty members (allocators, policies, deleters) | No size cost |

```cpp
struct ParticlesSoA
{
  std::vector<float> x, y, vx, vy;   // a loop over positions touches only x and y
};

struct alignas(64) PaddedCounter     // or std::hardware_destructive_interference_size
{
  std::atomic<std::uint64_t> value{0U};
};
std::array<PaddedCounter, 8U> perThreadCounters;   // each thread writes its own cache line
```

- Padding and member order: `07-memory-allocation.md` section 8.2. Bit-fields save memory but cost
  access time; their layout rules are in `07-memory-allocation.md` section 8.3.

## 4. Allocation

Allocation costs tens of nanoseconds on the fast path and much more on the slow path (locks,
system calls, fragmentation) — see `07-memory-allocation.md` section 1. Strategies (arena, pool,
ring buffer) are in section 3, the no-allocation policy for real-time code in section 6.

- Hoist allocations out of loops; reuse buffers (`clear()` keeps the capacity).
- `reserve()` vectors and strings when the size is known or bounded.
- Do not reserve a large maximum up front in GENERAL code when typical use is much smaller (a cache
  with a capacity of 10 million entries must not allocate 10 million nodes in its constructor).
  Reserve the expected size and let it grow, or document the up-front cost. Real-time code is the
  exception: there, allocate the maximum at initialization on purpose (`07-memory-allocation.md` section 6).
- Per-frame or per-request temporary data: an arena / `std::pmr::monotonic_buffer_resource`, released at once.
- In hot paths avoid types that allocate or synchronize behind your back: `std::function` (may
  allocate), `std::shared_ptr` copies (atomic operations), `std::string` temporaries. Use
  `std::string_view`, `std::span`, templates, `function_ref`.
- Small-buffer containers (`boost::container::small_vector`, `absl::InlinedVector`, LLVM
  `SmallVector`) when most cases fit in a small fixed size.

## 5. Copies and moves

Parameter passing and return values: `03-functions-interfaces.md` sections 2-3 (by value / `const&`
/ sink, RVO, no `return std::move(local)`). Move constructors must be `noexcept` so that containers
move on growth (`06-memory-ownership-lifetime.md` section 9.2). Performance-specific traps:

- `std::move` on a `const` object silently copies.
- Range-for: `for (const auto& x : v)` for read; `for (auto& x : v)` to modify; `for (auto x : v)` only for cheap types. Watch implicit conversions: `for (const std::pair<std::string, int>& p : map)` copies each element (the key is `const std::string`); use `const auto&` or structured bindings.
- `emplace`/`try_emplace` to build in place; `insert_or_assign` for maps.
- Avoid temporaries from implicit conversions (`const std::string&` parameter called with a literal allocates); use `std::string_view` parameters.
- Heterogeneous lookup: `std::map<std::string, V, std::less<>>` and `find(std::string_view)` avoid temporary strings. Same for unordered containers with transparent hash (C++20).

## 6. Compile-time computation (CG Per.11)

- `constexpr` functions and variables, `consteval` table builders, `if constexpr` to remove
  branches at compile time, templates to specialize code for fixed parameters.
- Generate lookup tables at compile time:
  ```cpp
  constexpr auto kSinTable = []
  {
    std::array<float, 256U> table{};
    for (std::size_t i = 0U; i < table.size(); ++i)
    {
      table[i] = constexprSin(static_cast<float>(i) * kStep);   // constexpr math (C++26 makes many <cmath> functions constexpr)
    }
    return table;
  }();
  ```
- Do not trade huge code size (template bloat) for small speed gains on cache-limited targets.

## 7. Branches and control flow

- Make the common path straight and the rare path out of line (error handling in a separate
  `[[gnu::cold]]` / `[[unlikely]]` function).
- `[[likely]]` / `[[unlikely]]` (C++20) only with profile data; wrong hints make code slower.
- Replace unpredictable branches in hot loops with arithmetic, `std::min`/`std::max`, conditional
  moves, or lookup tables — only when measured.
- Virtual calls cost an indirect branch and block inlining. In hot loops, group objects by type,
  use `final`, `std::variant`, or templates. Profile-guided optimization helps devirtualization.
- Exceptions: zero cost on the normal path in table-based ABIs, very expensive when thrown. Never use them for normal control flow.

## 8. Functions and inlining

- Small functions in headers (or with LTO) can be inlined. Do not force-inline without measurement (`[[gnu::always_inline]]`).
- Pass small trivially copyable objects by value (registers), not by reference.
- Avoid aliasing problems: the compiler must reload values through pointers that may alias. Use local copies of members in hot loops; `__restrict` (compiler extension) only with care.
- Pure functions can be marked `[[gnu::pure]]` / `[[gnu::const]]` on GCC/Clang when approved.

## 9. Floating point and SIMD

- Let the compiler auto-vectorize: simple loops, contiguous data, no aliasing, no function calls
  inside, known trip counts. Check with `-fopt-info-vec` (GCC) or `-Rpass=loop-vectorize` (Clang).
- Use `float` instead of `double` when precision allows and the FPU is single precision.
- Avoid denormals in hot DSP loops (flush-to-zero mode on platforms that allow it, with analysis).
- Explicit SIMD: `std::experimental::simd` / C++26 `std::simd`, Highway, xsimd, or intrinsics
  behind a portable wrapper. Keep a scalar reference implementation.
- `std::fma` for accuracy and speed where hardware FMA exists.
- Avoid `-ffast-math` in safety code (see `02-types-expressions-ub.md`).

## 10. Concurrency performance

- Minimize shared writable data. Batch work per thread; merge at the end.
- Keep critical sections short; never do I/O or allocation inside a lock.
- Avoid contended atomics in hot loops (each RMW on a shared line costs a cache-line transfer).
- Reader-heavy data: `std::shared_mutex`, RCU-like double buffers, or immutable snapshots via `std::atomic<std::shared_ptr<T>>` (C++20; may use a lock internally).
- Thread pools instead of creating threads per task. Pin threads and set priorities in real-time systems.

## 11. I/O and strings

- `'\n'` instead of `std::endl` (CG SL.io.50; `endl` flushes).
- `std::format` / `std::print` (C++20/23) or {fmt}: faster and type-safe compared with iostreams and printf.
- `std::to_chars` / `std::from_chars` (C++17): fastest locale-free number conversion.
- Logging in real-time paths: asynchronous, preallocated, binary loggers; compile-time log level filters.

## 12. Performance checklist

- [ ] Algorithm complexity fits the expected data size; no hidden quadratic loops.
- [ ] Contiguous data structures; hot data compact; no false sharing.
- [ ] No allocation, I/O, or locks in hot or real-time paths (or justified and measured).
- [ ] No needless copies (parameters, range-for, returns, conversions).
- [ ] `reserve()` where sizes are known.
- [ ] Work moved to compile time or out of loops where possible.
- [ ] Every optimization that reduces clarity has a comment with the measurement.
- [ ] Benchmarks/profiles exist for performance claims; worst case checked for real-time code.
