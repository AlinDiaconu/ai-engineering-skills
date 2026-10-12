# 12 — Standard library: choice, use, pitfalls

This file owns: container selection, algorithms, strings, vocabulary types, standard-library
pitfalls, and library facilities to avoid. Iterator invalidation rules are in
`06-memory-ownership-lifetime.md` section 7.3. Removed and deprecated features are in
`11-templates-modern-cpp.md` section 8.

## 1. Policy by context

- GENERAL: use the standard library wherever possible (CG SL.1, SL.2). Prefer it to custom code
  and to third-party code.
- REAL-TIME: in real-time paths use parts that need no allocation and no exceptions, or
  fixed-capacity containers; allocate and `reserve` everything at initialization.
- SAFETY-CRITICAL: use only the parts that the project has approved. Some projects forbid
  the standard library completely; follow that rule and do not add standard-library code there.
- A new third-party library is a project decision, not a coding decision. Prefer the standard
  library; do not add a dependency only because it builds.

## 2. Container selection

| Need | Container |
|---|---|
| Default sequence | `std::vector` |
| Fixed size known at compile time | `std::array` |
| Fixed capacity, no heap | `std::inplace_vector` (C++26), `boost::container::static_vector` |
| Mostly small, sometimes large | small vector (`absl::InlinedVector`, `boost::container::small_vector`) |
| Push/pop at both ends, stable references | `std::deque` |
| FIFO/LIFO | `std::queue`/`std::stack` adapters (on `deque`/`vector`); ring buffer for bounded real-time queues |
| Sorted, mostly read | sorted `std::vector` + `std::lower_bound`, `std::flat_map`/`flat_set` (C++23) |
| Sorted, frequent insert/erase, stable iterators | `std::map`/`std::set` |
| Key lookup, large n | `std::unordered_map` (reserve, good hash) or an open-addressing map |
| Priority queue | `std::priority_queue` (reserve the underlying vector) |
| Bit set, fixed size | `std::bitset<N>` |
| Value or nothing | `std::optional<T>` |
| One of N types | `std::variant<Ts...>` |
| Non-owning views | `std::span`, `std::string_view`, `std::mdspan` (C++23) |
| Text | `std::string` (owning), `std::string_view` (view); fixed-capacity strings where allocation is not allowed |

Memory behavior of each container (overhead per element, growth, SSO): see
`06-memory-ownership-lifetime.md` section 9.

## 3. Algorithms

- Prefer algorithms to raw loops when they name the intent (`std::any_of`, `std::count_if`,
  `std::transform`, `std::accumulate`/`std::reduce`, `std::find_if`, `std::partition`).
- Ranges versions (C++20) accept whole containers and projections:
  `std::ranges::sort(orders, {}, &Order::createdAt);`. Rules for range views:
  `11-templates-modern-cpp.md` section 5.
- `std::sort` is not stable; use `std::stable_sort` when equal elements must keep their order.
- Comparators must be strict weak orderings (`<`, never `<=`); a bad comparator is UB in `std::sort`.
- `std::accumulate` uses the type of the initial value: `std::accumulate(v.begin(), v.end(), 0)`
  on `double` data truncates — use `0.0`.
- `std::reduce` may reorder operations (floating-point results can differ from `accumulate`).
- `std::remove`/`remove_if` do not erase; use `std::erase_if` (C++20) or erase-remove.
- `std::lower_bound`/`binary_search` need sorted ranges.
- `std::min`/`std::max`/`std::clamp` return references; `const auto& m = std::min(a(), b());` dangles.
  Store the result by value.

## 4. Strings and text

- `std::string` owns; `std::string_view` observes (CG SL.str.1, SL.str.2). `string_view` is not
  null-terminated — do not pass `sv.data()` to C APIs that expect a C string.
- Build strings with `std::format`/{fmt} or `reserve` + `append`, not repeated `+` in loops.
- Number conversion: `std::to_chars`/`std::from_chars` (fast, no locale, no exceptions). Avoid
  `atoi`/`atof` (no error checking), `std::stoi` (throws), `sprintf` (overflow risk).
- `std::snprintf` only with a buffer size and a checked return value, if formatting libraries are not allowed.
- Character classification (`std::isdigit` etc.) needs `unsigned char` values:
  `std::isdigit(static_cast<unsigned char>(c))`.
- Encoding: use UTF-8 everywhere internally. `char8_t`/`std::u8string` (C++20) has poor library
  support; most projects keep `std::string` with UTF-8 content.

## 5. Vocabulary types

- `std::optional`: check before access (`has_value()`, `if (opt)`); `*opt` on an empty optional is
  UB; `value()` throws. `value_or(default)` for defaults. Avoid `std::optional<bool>` (confusing).
- `std::variant`: use `std::visit` with an overload set; `std::get` throws on a wrong type,
  `std::get_if` returns null. Use `std::monostate` as the first alternative if no type is
  default-constructible.
- `Overloaded` helper: this is the single definition used by every file of this skill.
  ```cpp
  template <class... Ts>
  struct Overloaded : Ts...
  {
    using Ts::operator()...;
  };

  struct Pending {};
  struct Shipped { std::uint32_t trackingId; };
  struct Cancelled { std::string reason; };
  using OrderState = std::variant<Pending, Shipped, Cancelled>;

  [[nodiscard]] std::string describe(const OrderState& state)
  {
    return std::visit(Overloaded{
                        [](const Pending&) { return std::string{"pending"}; },
                        [](const Shipped& s) { return "shipped #" + std::to_string(s.trackingId); },
                        [](const Cancelled& c) { return "cancelled: " + c.reason; }},
                      state);
  }
  ```
- `std::span<T, N>` with a static extent when the size is fixed; dynamic extent otherwise.
  `operator[]` is not checked (checked in C++26 hardened implementations).
- `std::chrono`: use typed durations everywhere; `steady_clock` for intervals and timeouts (never
  `system_clock`, it can jump); `duration_cast` only when you accept truncation;
  `std::chrono::round`/`floor`/`ceil` for explicit rounding.
- `std::tuple`: only for generic code; use named structs in interfaces.
- `std::pair`: acceptable for very local use; prefer named structs (`.first`/`.second` hide meaning).

## 6. Utilities

Smart pointers: see `06-memory-ownership-lifetime.md` section 5. Also:
- `std::exchange` for move operations and "take the value and reset".
- `std::as_const` to call const overloads.
- `std::clamp(v, lo, hi)` requires `lo <= hi` (UB otherwise).
- `std::midpoint` (C++20) avoids overflow in `(a + b) / 2`.
- `std::numeric_limits<T>` for ranges; never hard-code `2147483647`.
- `<random>`: `std::mt19937` (seeded properly) or a better engine. Distributions differ between
  library implementations; for results reproducible across platforms, write your own mapping.
  Never use `<random>` engines for keys or tokens (not cryptographically secure).

## 7. I/O

- `std::format`/`std::print`/{fmt} for formatting. iostreams only when the project uses them.
- Check every I/O result. Files through RAII (`std::ifstream` closes automatically).
- `<filesystem>`: use the `std::error_code` overloads in code without exceptions.

## 8. Library facilities to avoid

| Facility | Problem | Use instead |
|---|---|---|
| `std::vector<bool>` | Bit-packed proxy, not a real container (no `bool&`, no `data()`) | `std::vector<std::uint8_t>`, `std::bitset<N>`, a dynamic bitset |
| `std::endl` | Flushes the stream every time (slow) | `'\n'` (CG SL.io.50) |
| `std::regex` | Very slow; large code; implementation differences | Hand-written parsers, RE2, CTRE |
| `std::async` | Launch policy surprises; the future's destructor blocks | A thread pool or executor |
| `std::any` | Weak typing; heap use; type errors only at run time | `std::variant`, templates, interfaces |
| `<iostream>` in size-critical or low-latency code | Code size, static initialization, slow formatting | `std::format` + `std::fwrite`/`std::print` |
| `std::unordered_map` in hot paths | One allocation per node, poor locality | Sorted `std::vector`, `std::flat_map`, open-addressing maps |
| `std::rand`, `RAND_MAX` | Poor quality, global state | `<random>` engines |
| `atoi`, `atof`, `strtol` without error checks | No or weak error reporting | `std::from_chars` |
| `strcpy`, `strcat`, `sprintf`, `gets`, `strtok`, `scanf("%s")` | Buffer overflow, hidden state | `std::string`, `std::format`, `std::span` |
| `std::list` as a default | Poor locality, one allocation per node | `std::vector` (use `std::list` only for stable iterators + splicing) |
| `std::shared_ptr` as a default | Atomic reference counts, unclear ownership | `std::unique_ptr` or values |
| `std::function` in hot paths | Possible heap allocation, indirect call | Templates, `function_ref` |

## 9. Known pitfalls

- `operator[]` on `std::map` inserts a default value when the key is absent; use `find`,
  `contains`, or `at` for lookup.
- `std::vector<T>::operator[]` is unchecked; `.at()` throws. Check external indexes explicitly.
- `std::unordered_map` iteration order is unspecified; never depend on it for output or hashing.
- `size()` returns an unsigned value: `for (int i = 0; i < v.size() - 1; ++i)` underflows when
  `v` is empty. Use `std::ssize()` or compare `i + 1 < v.size()`.
- `std::string::find` returns `npos` (largest `size_t`) on failure — compare with `npos`.
- `std::abs` on `INT_MIN` is UB; `std::abs` on an unsigned value is ambiguous.
- `std::move` of a `const` object copies.
- `std::initializer_list` elements are `const` (you cannot move from them).
- `std::shared_ptr<T>(new T[n])` before C++17 deletes with `delete` (wrong); use
  `std::shared_ptr<T[]>` (C++17) or `std::vector`.
- `std::function` copies its target; use `std::move_only_function` (C++23) for move-only callables.
- Brace initialization and `initializer_list` constructors: see `02-types-expressions-ub.md` section 1.
