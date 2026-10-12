# 03 — Functions and interfaces

## 1. Design

- One function, one logical operation (CG F.2). If you need "and" to describe it, split it.
- Short and simple (CG F.3). Defaults: ≤ 50 lines, cyclomatic complexity ≤ 10, nesting depth ≤ 3.
  SAFETY-CRITICAL: the project metric limits are hard limits. Reduce complexity with early returns
  (if allowed), helper functions, lookup tables, and polymorphism — never by hiding logic in macros.
- Prefer pure functions: output depends only on inputs, no side effects (CG F.8). They are easy to
  reason about and safe to call from several threads.
- Make interfaces explicit (CG I.1): no hidden dependence on globals, singletons, or call order.
- Make interfaces strongly typed (CG I.4): `setTimeout(std::chrono::milliseconds)`, not `setTimeout(int)`.
- Make an interface easy to use correctly and hard to use wrongly.
- Some safety standards require one exit point per function. Follow the project. Otherwise,
  early returns for preconditions are clearer.

## 2. Parameter passing (CG F.15-F.21)

| Purpose | Cheap to copy (≤ 2-3 words, trivially copyable: `int`, `float`, `std::string_view`, `std::span`, small PODs) | Expensive to copy / move-only |
|---|---|---|
| In | `T` (by value) | `const T&` |
| In, and the function keeps a copy (sink) | `T` | `T` by value + `std::move` inside, or `T&&` overload for hot paths (see the failure rule below) |
| In/out | `T&` | `T&` |
| Out | return `T` | return `T` (RVO / move) |
| Several outputs | return a `struct` | return a `struct` |
| Optional input | `std::optional<T>` or `const T*` (null = absent) | `const T*` |
| Optional output | `std::optional<T>` / `std::expected<T,E>` | same |
| Read-only sequence | `std::span<const T>` (C++20) or `gsl::span` | same |
| Read-only string | `std::string_view` | same |
| Callable | template parameter or `std::function_ref` (C++26) / `FunctionRef`; `std::function` only when you store it | |
| Ownership transfer | `std::unique_ptr<T>` by value | same |
| Shared ownership | `std::shared_ptr<T>` by value only if the function keeps a copy; else `T&` / `const T&` | |

Rules:
- **A sink that can fail must not destroy a move-only argument on failure.** A by-value parameter
  is consumed before the function body runs; if the function then returns an error (queue full,
  wrong pool, invalid state), the resource is lost. For move-only resources (handles, buffers,
  slots) in fallible operations: take `T&&` and move from it only on success, or return the
  argument with the error (for example `std::expected<void, std::pair<Error, T>>`).
- Never transfer ownership with a raw pointer or reference (CG I.11).
- A reference parameter means "not null". A pointer parameter means "may be null" — check it.
- Forwarding references `T&&` in templates: always `std::forward<T>(x)`, use each forwarded argument once.
- Do not pass `std::shared_ptr` by `const&` only to read the object; pass `const T&`.
- Keep the parameter count low (CG I.23). Default ≤ 4. Group related parameters in a struct;
  use designated initializers (C++20) for clear call sites:
  `connect({.host = "example.org", .port = 443, .timeout = 5s})`.
- Avoid two adjacent parameters of the same type (CG I.24). Use strong types.
- Avoid `bool` parameters that change behavior: `render(true)` is unclear. Use an `enum class` or two functions.
- Default arguments: avoid in virtual functions (CG C.140); keep them few and obvious.

### Safety-critical alternative
Some safety standards forbid returning class objects by value or forbid large stack objects. In
that case use an out-parameter (`T&`) or a caller-provided buffer and return an error/status code.
Since C++17, returning a prvalue has guaranteed copy elision, so returning by value has no copy
cost; mention this if the rule is project-optional.

## 3. Return values

- Return by value (CG F.20). C++17 guarantees elision for prvalues; NRVO is very common.
- Never `return std::move(local);` — it blocks NRVO.
- Never return a reference or pointer to a local or a temporary (CG F.43).
- Never return `T&&` (CG F.45). Never return `const T` by value (CG F.49; blocks move).
- Return a `struct` (or `std::pair`/`std::tuple` for very local use) for several values (CG F.21).
  Prefer a named struct for clarity. Use structured bindings at the call site.
- Use `std::optional<T>` for "value or nothing", `std::expected<T, E>` (C++23; `tl::expected` before)
  for "value or error". See `09-error-handling.md`.
- Returning a reference to a member is acceptable for getters on large objects; document that it
  is valid only while the object lives. Do not return non-const references to internal data
  (breaks encapsulation).
- `[[nodiscard]]` on functions whose result must be used: error/status results, factory functions,
  pure computations, `empty()`-style queries. Use `[[nodiscard("reason")]]` (C++20). Mark error
  types themselves `[[nodiscard]]` (class attribute) so every function that returns them is covered.

## 4. Specifiers

- `constexpr` if the function can run at compile time (CG F.4). `consteval` if it must.
- `noexcept` if the function cannot throw or a throw must terminate (CG F.6). Required for move
  constructors, move assignment, `swap`, destructors (implicit), and hash functions, so that
  containers can use them. Do not add `noexcept` to a function that may throw: a throw then calls
  `std::terminate`.
- `inline`: for ODR in headers, not as an optimization hint. The compiler decides inlining.
- `static` member functions when they do not use `this`. Free functions when they do not need
  private access (they improve encapsulation).
- `[[maybe_unused]]` for parameters used only in some configurations; do not leave unused
  parameters without a name or this attribute.
- `= delete` to forbid dangerous overloads: `void setRatio(double) = delete;` when only `float` is
  allowed; `T(const T&&) = delete` for rvalue-only traps.

## 5. Overloading

- Overload only for functions that do the same thing for different types (CG C.162/C.163).
- Do not overload on pointer vs integer (`f(int)` and `f(char*)` with `f(0)`).
- Do not overload `&&`, `||`, `,`, or unary `&` (CG C.167/C.170 spirit, MISRA).
- Operators: keep natural meaning. Symmetric binary operators as non-member (hidden friend or free
  function). C++20: `= default` the `operator<=>` and `operator==` when member-wise comparison is right.
- Templates: overload function templates, do not specialize them (`11-templates-modern-cpp.md` section 1).

## 6. Lambdas

- Use lambdas for local, short behavior: predicates for algorithms, local initialization, callbacks.
- Capture explicitly. Avoid `[=]` and `[&]` in lambdas that outlive the current scope (CG F.52/F.53).
- Never capture `this` implicitly by `[=]` (deprecated in C++20). Write `[this]` or `[*this]`.
- A lambda stored or passed to another thread must not capture locals by reference.
- Keep lambdas short (≤ 10 lines). Give a long lambda a name or make it a function.
- Generic lambdas (`auto` parameters) and template lambdas (C++20 `[]<typename T>(T x)`) are fine.
- `mutable` lambdas: avoid; they hide state.

## 7. Callbacks and type erasure

| Need | Use |
|---|---|
| Call now, do not store | template parameter `F&&` or `function_ref` (C++26 `std::function_ref`, or a small custom `FunctionRef`) |
| Store, copyable | `std::function<Sig>` (may allocate; check SBO size) |
| Store, move-only | `std::move_only_function<Sig>` (C++23) |
| Store, no heap | fixed-capacity, non-allocating delegate (for example an `inplace_function`), or interface + static object |
| Many listeners | observer with clear unregister on destruction (RAII subscription handle) |

Never call a callback while you hold a lock (CG CP.22; `10-concurrency.md` section 3).

## 8. Contracts: preconditions and postconditions

- Document the contract of every public function: valid ranges, units, null rules,
  thread safety, and what happens on violation.
- Check preconditions at module boundaries (inputs from other components, files, network,
  users). Inside a module, use assertions for "cannot happen" states.
- How to check (assertions, contract syntax, behavior in release builds): see
  `09-error-handling.md` section 6. Safe-state reaction in safety-critical code: section 7.

```cpp
/// @brief Computes the total price of an order line.
/// @param[in] unitPrice  Price of one item. Must be >= 0.
/// @param[in] quantity   Number of items. Range [1, 10000].
/// @return Total price, or Error::InvalidInput when an input is out of range.
[[nodiscard]] std::expected<Money, Error> computeLineTotal(Money unitPrice, std::int32_t quantity) noexcept;
```

## 9. Trust boundaries and input validation

A trust boundary is every place where data enters from outside the component: network, files,
IPC, shared memory, command line, environment variables, plugins, devices, user input.

- **Validate once, at the boundary.** Check length, range, format, and allowed values. Then
  convert to strong internal types (section 1, `02-types-expressions-ub.md` section 4). Inner
  code trusts these types and does not check again.
- **Reject, do not repair.** Return an error for unknown fields or out-of-range values. Silent
  correction hides attacks and bugs.
- **Check sizes before every copy.** Compare a length field with the real buffer size before
  `std::copy`/`std::memcpy`. Never use an external value as an index without a check.
- **Check size calculations for overflow** (`count * sizeof(T)`, `offset + length`) before you
  allocate or copy.
- **No unsafe C string and format functions:** no `strcpy`, `strcat`, `sprintf`, `gets`,
  `scanf("%s")`, `strtok`, `atoi`. Use `std::string`, `std::format`, `std::from_chars`, and
  `std::span`. Format strings are always literals.
- **Bound resource use.** Limit sizes, element counts, nesting depth, recursion depth, and loop
  counts that external data controls. Unbounded input allows denial of service.
- **No information leaks.** Initialize structs (including padding) before you send their bytes.
  Do not log secrets or personal data.
- **Clear secrets after use** with a function the compiler cannot remove (`explicit_bzero`,
  `SecureZeroMemory`, `memset_s`). A plain `memset` before the end of a lifetime is often removed
  by the optimizer. Do not keep secrets in `std::string` copies.
- **Time of check vs time of use.** Check the object you use (an open file handle), not a name
  that can change between the check and the use (a path).

```cpp
struct Header
{
  std::uint32_t payloadLength;
};

/// Reads a payload whose length comes from untrusted input.
[[nodiscard]] std::expected<std::span<const std::byte>, Error>
readPayload(std::span<const std::byte> packet) noexcept
{
  constexpr std::size_t kMaxPayload{64U * 1024U};          // bound resource use
  if (packet.size() < sizeof(Header))
  {
    return std::unexpected{Error::Truncated};
  }
  Header header{};
  std::memcpy(&header, packet.data(), sizeof(Header));     // no misaligned pointer cast
  const std::size_t length{header.payloadLength};
  if ((length > kMaxPayload) || (length > (packet.size() - sizeof(Header))))   // no overflow: subtract, not add
  {
    return std::unexpected{Error::InvalidLength};
  }
  return packet.subspan(sizeof(Header), length);
}
```

## 10. Function calls and side effects

- Do not depend on the evaluation order of function arguments (unspecified).
- Do not call functions with side effects inside `&&`/`||` right operands (they may not run) —
  MISRA forbids side effects there.
- `std::invoke` for generic callables. `std::bind` is obsolete; use lambdas.
- Avoid deep call chains in real-time paths; keep the worst-case call depth known.
