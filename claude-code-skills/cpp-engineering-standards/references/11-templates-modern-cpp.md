# 11 — Templates, generic code, modern C++ by version

## 1. Templates

- Use templates to raise abstraction and to write one algorithm for many types (CG T.1, T.2).
- Constrain every template parameter (CG T.10). C++20: concepts. Before C++20: `static_assert`
  with type traits, or `std::enable_if_t` in a defaulted template parameter.
- Use standard concepts first (CG T.11): `std::integral`, `std::floating_point`,
  `std::invocable`, `std::regular`, `std::copyable`, `std::ranges::range`, `std::same_as`, `std::convertible_to`.
- Shorthand syntax for simple constraints (CG T.13): `void f(std::integral auto x);`.
- Custom concepts express semantics, not only syntax:
  ```cpp
  template <typename T>
  concept Shape = requires(const T& shape)
  {
    { shape.area() } -> std::convertible_to<double>;
    { shape.name() } -> std::convertible_to<std::string_view>;
  };

  template <Shape S>
  void print(const S& shape);
  ```
- `using` aliases instead of `typedef` (CG T.43).
- Overload function templates; do not specialize them (CG T.144). Class template partial specialization is fine.
- `if constexpr` (C++17) instead of tag dispatch and SFINAE for branches inside one template.
- Fold expressions (C++17) instead of recursive variadic templates.
- Template metaprogramming only when `constexpr` code cannot do the job (CG T.120).
- Keep templates readable: name intermediate types, limit nesting.
- Control code bloat on small targets: move type-independent code into non-template base functions; use explicit instantiation (`extern template`) for common types.
- Put templates in a `detail` namespace when they are implementation parts. Avoid unconstrained templates in visible namespaces (ADL surprises; CG T.47).
- Error messages: concepts and `static_assert` with a message make errors readable.

## 2. CRTP and static polymorphism

```cpp
template <typename Derived>
class Filter
{
public:
  [[nodiscard]] float apply(float x) noexcept { return static_cast<Derived&>(*this).applyImpl(x); }
protected:
  Filter() = default;   // only derived classes construct it
};

class LowPass final : public Filter<LowPass>
{
  friend class Filter<LowPass>;
  [[nodiscard]] float applyImpl(float x) noexcept;
};
```
C++23 deducing `this` replaces many CRTP uses: `auto apply(this auto& self, float x)`.

## 3. `constexpr` everything that can be

- C++14: `constexpr` functions with loops and local variables.
- C++17: `constexpr` lambdas, `if constexpr`, `inline` variables.
- C++20: `constexpr` `std::vector`/`std::string` (transient allocation only), virtual functions,
  `try` blocks, `std::is_constant_evaluated()`, `consteval`, `constinit`.
- C++23: `if consteval`, more `constexpr` in `<cmath>`-related parts, `std::unique_ptr` constexpr.
- C++26: `constexpr` placement new, more `<cmath>`, `constexpr` exceptions (compiler support varies).

## 4. Feature guide by version

Use a feature only if the project standard version and the compiler support it. Check
`__cpp_*` feature-test macros (`<version>`, C++20) for portable code.

### C++11/14 (baseline everywhere)
`auto`, range-for, lambdas, move semantics, `nullptr`, `enum class`, `constexpr`,
`static_assert`, `override`/`final`, `= default`/`= delete`, smart pointers, `std::array`,
`<chrono>`, `<thread>`, `<atomic>`, uniform initialization, `noexcept`, variadic templates,
generic lambdas (14), `std::make_unique` (14), digit separators `1'000'000` (14), binary literals (14).

### C++17
| Feature | Use |
|---|---|
| Structured bindings | `auto [it, inserted] = m.try_emplace(k, v);` |
| `if`/`switch` with initializer | Limit variable scope |
| `if constexpr` | Compile-time branches |
| `std::optional`, `std::variant`, `std::any` | Value or nothing; closed sum types; avoid `any` (`12-standard-library.md` section 8) |
| `std::string_view` | Non-owning string parameters |
| Guaranteed copy elision | Return by value without cost |
| `[[nodiscard]]`, `[[maybe_unused]]`, `[[fallthrough]]` | Intent for compiler checks |
| Nested namespaces `a::b::c` | Shorter code |
| `inline` variables | Header constants with one instance |
| CTAD | `std::array a{1, 2, 3};`, `std::scoped_lock lock{m};` |
| Fold expressions | Variadic logic |
| `std::byte` | Raw memory |
| `std::from_chars`/`to_chars` | Fast, locale-free conversion |
| `<filesystem>` | Paths and files (GENERAL context) |
| `std::scoped_lock`, `std::shared_mutex` | Locking |
| `<memory_resource>` (pmr) | Custom allocation strategies |
| Parallel algorithms | `std::execution::par` |
| `std::clamp`, `std::gcd`, `std::lcm`, `std::size`, `std::data`, `std::invoke`, `std::apply` | Utilities |

### C++20
| Feature | Use |
|---|---|
| Concepts, `requires` | Constrained templates |
| Ranges (`std::ranges::sort`, views) | Composable algorithms; see section 5 |
| `std::span` | Non-owning contiguous view (replaces pointer + size) |
| Three-way comparison `<=>`, defaulted `==` | Less boilerplate |
| Designated initializers | `Config{.rate = 10, .retries = 3}` |
| `consteval`, `constinit` | Compile-time guarantees |
| `std::format` | Type-safe formatting |
| `std::jthread`, `std::stop_token`, `std::latch`, `std::barrier`, semaphores | Concurrency |
| `std::atomic_ref`, `atomic::wait/notify`, `atomic<shared_ptr>` | Atomics |
| `<bit>`: `std::bit_cast`, `popcount`, `endian`, `has_single_bit` | Bit work |
| `std::source_location` | Logging/assert location without macros |
| `[[likely]]`, `[[unlikely]]`, `[[no_unique_address]]` | Hints, layout |
| `std::cmp_*`, `std::in_range`, `std::ssize` | Safe integer comparisons |
| `std::erase`/`erase_if`, `contains()`, `starts_with()`/`ends_with()` | Container/string utilities |
| `std::make_unique_for_overwrite`, `std::to_array`, `std::midpoint`, `std::lerp`, `std::numbers::pi` | Utilities |
| `using enum` | Shorter switch statements |
| Lambda improvements: template lambdas, `[=, this]`, lambdas in unevaluated contexts | |
| Modules | Faster builds, no macro leakage; see section 7 |
| Coroutines | Asynchronous code and generators; see section 6 |

### C++23
| Feature | Use |
|---|---|
| `std::expected` | Value or error |
| Monadic `optional`/`expected` (`and_then`, `transform`, `or_else`) | Chaining |
| `std::print`, `std::println` | Formatted output |
| Deducing `this` | Removes const/non-const overload duplication, simpler CRTP |
| `std::flat_map`, `std::flat_set` | Cache-friendly sorted maps |
| `std::mdspan` | Multidimensional views (matrices, images) |
| `std::to_underlying`, `std::unreachable`, `std::byteswap` | Utilities |
| `std::move_only_function` | Store move-only callables |
| `if consteval`, `[[assume(expr)]]` | Compile-time branch; optimizer hint (UB if false — avoid in SAFETY-CRITICAL) |
| `std::start_lifetime_as` | Objects in I/O buffers |
| `std::stacktrace` | Diagnostics (GENERAL) |
| `std::ranges::to`, `views::zip`, `views::enumerate`, `views::chunk`, `views::slide` | Ranges |
| `std::generator` | Coroutine generator |
| `import std;` | Standard library module |
| Range-for temporary lifetime extension | Fixes `for (auto& x : f().items())` dangling |
| `std::string::contains` | |

### C++26 (feature-complete 2025, ISO publication 2026; check compiler support before use)
| Feature | Use |
|---|---|
| Contracts (`pre`, `post`, `contract_assert`) | Standard preconditions/postconditions |
| Static reflection (`^^T`, `std::meta`) | Generate serialization, enum-to-string, bindings |
| `std::inplace_vector` | Fixed-capacity vector without heap (no-allocation code) |
| Erroneous behavior for uninitialized reads | Safer defaults; still initialize |
| Hardened standard library | Bounds checks on `operator[]`, `front()`, etc. in hardened mode |
| `std::execution` (senders/receivers) | Structured async |
| `std::simd` | Portable SIMD |
| Saturating arithmetic (`std::add_sat`, ...) | Safe arithmetic |
| `std::function_ref` | Non-owning callable parameter |
| `std::hive`, `std::text_encoding`, `<linalg>`, `#embed`, pack indexing, `= delete("reason")`, placeholder `_` variables, structured binding as condition | Various |

## 5. Ranges (C++20)

- Range algorithms (`std::ranges::sort(v)`, `std::ranges::find(v, x)`) take the whole range and
  support projections. Prefer them to iterator pairs.
- **Views are lazy.** `v | std::views::filter(p) | std::views::transform(f)` does no work until
  you iterate. Each iteration runs the whole pipeline again — do not iterate an expensive view
  twice; materialize it (`std::ranges::to<std::vector>()` in C++23, or a loop into a vector).
- **Views refer to data; they do not own it** (except `owning_view`). The rules for dangling
  pointers apply (`06-memory-ownership-lifetime.md` section 7).
  - Range algorithms called on a temporary non-borrowed container return `std::ranges::dangling`
    instead of an iterator: the code does not compile. Do not work around it; keep the container
    in a named variable.
  - Do not store a view as a class member unless the class also owns or outlives the data.
- Some views cache state on first use (`filter`, `drop_while`): iterating them is not
  `const`-safe, and a view used from two threads at the same time is a data race. Pass views by
  forwarding reference (`auto&&`) or by value, not by `const&`.
- Do not change the underlying container while a view over it is in use.
- Use a plain loop when it is clearer: several outputs, early exit with complex state, or a
  pipeline longer than three or four steps. Readability first.
- Long view pipelines increase compile time and debug-build run time. Keep them short in headers
  and hot debug paths.

```cpp
[[nodiscard]] std::vector<std::string> openOrderIds(const std::vector<Order>& orders)
{
  auto ids = orders
           | std::views::filter([](const Order& o) { return o.isOpen; })
           | std::views::transform([](const Order& o) { return o.id; });
  return {ids.begin(), ids.end()};   // C++23: return ids | std::ranges::to<std::vector>();
}
```

## 6. Coroutines (C++20)

- Use a library coroutine type (`std::generator` in C++23, Asio awaitables, a project task
  type). Do not write promise types in application code; they are hard to get right.
- **Parameters by value.** A coroutine can run after the caller's full expression ends.
  Reference parameters (`const std::string&`) and `std::string_view`/`std::span` parameters
  dangle when the caller passes a temporary.
- **Lambda coroutines:** captures live in the lambda object, not in the coroutine frame. If the
  lambda dies first, captures dangle. Pass data as parameters by value instead of captures.
- **Never hold a lock across `co_await`.** The coroutine can resume on another thread
  (`std::mutex` must be unlocked by the thread that locked it) and other work can deadlock on the lock.
- **Know who resumes the coroutine and on which thread.** Document it for each awaitable.
- **Allocation:** the coroutine frame is normally heap-allocated. The compiler can elide this
  (HALO) only when the coroutine's lifetime is nested in the caller. In real-time code use a
  task type with a custom allocator, or do not use coroutines.
- **Every started coroutine is awaited or destroyed;** a detached coroutine that keeps
  references is a use-after-free source.
- Exceptions in coroutines go to the promise (`unhandled_exception`); make sure the library type
  rethrows or reports them.

```cpp
// C++23 std::generator: lazy sequence; parameter by value.
std::generator<std::uint64_t> fibonacci(std::uint32_t count)
{
  std::uint64_t a{0U};
  std::uint64_t b{1U};
  for (std::uint32_t i = 0U; i < count; ++i)
  {
    co_yield a;
    a = std::exchange(b, a + b);
  }
}
```

## 7. Modules (C++20)

- A module interface unit starts with `export module name;`. Only exported declarations are
  visible to importers. Macros never cross a module boundary.
- Export only the public interface. Keep implementation details unexported (they replace the
  `detail` namespace and pImpl for hiding, but not for ABI stability).
- Split large modules into partitions (`export module lib:part;`, imported as
  `import :part;` inside the module). Only the primary interface unit re-exports the public partitions.
- Put legacy `#include` directives in the **global module fragment** (`module;` before
  `export module name;`), not after the module declaration.
- `import std;` (C++23) replaces all standard headers in modular code.
- Do not mix `#include` and `import` of the same library in one translation unit without a reason.
- Adopt modules per project when the compiler and build system of every target support them;
  keep header interfaces for code that must build with older toolchains.

```cpp
// geometry.cppm — primary module interface unit
module;                         // global module fragment: legacy headers only
#include <cmath>

export module geometry;

export struct Point
{
  double x;
  double y;
};

export [[nodiscard]] double distance(Point a, Point b) noexcept
{
  return std::hypot(a.x - b.x, a.y - b.y);
}

double unexportedHelper(double v) { return v * v; }   // not visible to importers
```

## 8. Removed and deprecated features

Do not use these in new code. Replace them when you change the code around them. Library
facilities that are legal but have better alternatives are in `12-standard-library.md` section 8.

| Feature | Status | Replacement |
|---|---|---|
| `std::auto_ptr` | Removed in C++17 | `std::unique_ptr` |
| `std::bind1st`, `std::bind2nd`, `std::ptr_fun`, `std::mem_fun` | Removed in C++17 | Lambdas |
| `std::random_shuffle` | Removed in C++17 | `std::shuffle` with a `<random>` engine |
| `register` storage class | Removed in C++17 | Nothing (let the compiler decide) |
| Dynamic exception specifications `throw(X)`; `throw()` | Removed in C++17; `throw()` removed in C++20 | `noexcept` |
| `std::uncaught_exception()` | Removed in C++20 | `std::uncaught_exceptions()` |
| `std::result_of` | Removed in C++20 | `std::invoke_result_t` |
| `std::shared_ptr::unique()` | Removed in C++20 | `use_count() == 1` (only as a hint) |
| `std::get_temporary_buffer`, `std::raw_storage_iterator` | Removed in C++20 | Allocators, uninitialized algorithms |
| `std::iterator` base class | Deprecated in C++17 | Declare the five member types directly |
| `<codecvt>`, `std::wstring_convert` | Deprecated in C++17 | Platform APIs or a Unicode library |
| `std::is_pod` | Deprecated in C++20 | `std::is_trivial` + `std::is_standard_layout` |
| Implicit capture of `this` by `[=]` | Deprecated in C++20 | `[=, this]` or `[*this]` |
| `volatile` compound assignment and `++`/`--` | Deprecated in C++20 (bitwise compound forms allowed again in C++23) | Explicit read, modify, write |
| Comma in subscript `a[i, j]` | Deprecated in C++20, changed meaning in C++23 | `a[i][j]` or `mdspan` |
| `std::aligned_storage`, `std::aligned_union` | Deprecated in C++23 | `alignas(T) std::byte buf[sizeof(T)]` |
| `gets` | Removed from C and C++ | `std::getline`, `fgets` |
