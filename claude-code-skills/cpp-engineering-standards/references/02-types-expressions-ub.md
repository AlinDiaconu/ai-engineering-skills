# 02 — Types, initialization, expressions, undefined behavior

## 1. Initialization

- Initialize every variable at its declaration (CG ES.20). Declare it in the
  smallest scope, as late as possible, when you know its value (CG ES.5, ES.21).
- Prefer brace initialization: it rejects narrowing (CG ES.23).
  Exception: `std::vector<int> v(10, 0)` vs `std::vector<int> v{10, 0}` — braces select the
  `initializer_list` constructor. Use parentheses when you need the size constructor.
- Use default member initializers so that every constructor gets a valid object:
  `std::int32_t m_count{0};`.
- Use an immediately invoked lambda to initialize a complex `const` value (CG ES.28):
  ```cpp
  const auto policy = [&]
  {
    RetryPolicy p{};
    p.maxAttempts = cfg.maxAttempts;
    p.backoff = isInteractive ? kShortBackoff : kLongBackoff;
    return p;
  }();
  ```
- `static` local variables: initialization is thread-safe (C++11), but every call pays a guard
  check. Use `constinit` (C++20) for namespace-scope objects to forbid dynamic initialization and
  avoid the static initialization order problem.
- Static initialization order: never let one namespace-scope object depend on another in a
  different translation unit. Use `constexpr`/`constinit` or a function-local static.
- C++26 makes reads of uninitialized automatic variables "erroneous behavior" instead of UB. Do
  not depend on this; initialize anyway.

## 2. `const` and compile-time

- `const` by default for locals, parameters passed by reference, and member functions (CG Con.1-4).
- `constexpr` for values and functions that can be computed at compile time (CG Con.5, F.4).
- `consteval` (C++20) for functions that **must** run at compile time (for example table builders, checked literals).
- `constinit` (C++20) for mutable globals that must have static initialization.
- `static_assert` to check assumptions at compile time: sizes, alignment, enum counts, traits.
  ```cpp
  static_assert(sizeof(MessageHeader) == 16U, "MessageHeader layout changed");
  static_assert(std::is_trivially_copyable_v<MessageHeader>);
  ```
- Do not make data members `const` or references in types that must be copyable or movable
  (CG C.12). Make them private and offer no setter instead.

## 3. Fundamental types

- Use fixed-width types from `<cstdint>` when size, range, or layout matters (interfaces,
  serialization, hardware, safety code): `std::int8_t` ... `std::uint64_t`. Many safety projects
  require project typedefs (for example `uint32`, `float32`); follow them.
- `std::size_t` for sizes and indexes of containers; `std::ptrdiff_t` for pointer differences.
  Use `std::ssize()` (C++20) when you need a signed size.
- `bool` only for true/false. No arithmetic on `bool`.
- `char` only for characters. Use `std::byte` (C++17) or `std::uint8_t` for raw bytes.
  The signedness of plain `char` is implementation-defined.
- `float` vs `double`: use the type the hardware supports fast. Some processors have only
  single-precision hardware; avoid silent promotion to `double` (write `1.0F`, use `std::sqrt(float)` overloads).
- Use `auto` when the type is obvious from the right side, or long/unknown (iterators, lambdas).
  Do not use `auto` when it hides an important type (units, signedness, size). Never
  `auto x = {1};` (it is `std::initializer_list<int>`).

## 4. Strong types

Primitive obsession causes unit and argument-order bugs. Wrap values that have a meaning:

```cpp
// Minimal strong type. Zero cost after optimization.
template <typename Tag, typename Rep>
class StrongType
{
public:
  constexpr explicit StrongType(Rep value) noexcept : m_value{value} {}
  [[nodiscard]] constexpr Rep value() const noexcept { return m_value; }
  friend constexpr bool operator==(StrongType a, StrongType b) noexcept { return a.m_value == b.m_value; }
private:
  Rep m_value;
};

using Meters     = StrongType<struct MetersTag, float>;
using Seconds    = StrongType<struct SecondsTag, float>;
using CustomerId = StrongType<struct CustomerIdTag, std::uint32_t>;

[[nodiscard]] float averageRate(Meters distance, Seconds duration);   // swapped arguments do not compile
```

- Use `std::chrono` durations and time points for all time values. Never pass time as a raw `int`.
- Consider a units library (for example mp-units, Au) for physics-heavy code, if approved.
- `enum class` with an explicit underlying type for states and modes:
  `enum class ConnectionState : std::uint8_t { Idle, Connecting, Connected, Closed };`.

## 5. Conversions and casts

- No C-style casts and no functional casts `T(x)` for non-class types (CG ES.48/ES.49, MISRA).
- `static_cast` for defined conversions. `const_cast` never to write to a `const` object (CG ES.50).
- `reinterpret_cast` only at hardware or serialization boundaries, with a justification. To read
  the bytes of an object as another type use `std::memcpy` or `std::bit_cast` (C++20), not a
  pointer cast (strict aliasing).
- `dynamic_cast`: avoid. Prefer virtual functions or `std::variant`. If used, check the result.
- No implicit narrowing (CG ES.46). Use a checked cast at boundaries:
  ```cpp
  template <typename To, typename From>
  [[nodiscard]] constexpr std::optional<To> checkedCast(From value) noexcept
  {
    if (!std::in_range<To>(value))   // C++20 <utility>
    {
      return std::nullopt;
    }
    return static_cast<To>(value);
  }
  ```
  Before C++20, compare with `std::numeric_limits<To>::min()/max()` carefully (signedness).
- Compare signed and unsigned values with `std::cmp_less`, `std::cmp_equal`, ... (C++20), or cast
  both to a common safe type.
- `explicit` on single-argument constructors and conversion operators (CG C.46, C.164).
- `std::to_underlying(e)` (C++23) or `static_cast<std::underlying_type_t<E>>(e)` for enum values.

## 6. Integer arithmetic

- Signed overflow is UB. Unsigned overflow wraps (defined, but often a logic bug).
- Check before the operation, not after:
  ```cpp
  [[nodiscard]] constexpr bool addOverflows(std::int32_t a, std::int32_t b) noexcept
  {
    return (b > 0) ? (a > std::numeric_limits<std::int32_t>::max() - b)
                   : (a < std::numeric_limits<std::int32_t>::min() - b);
  }
  ```
  GCC/Clang: `__builtin_add_overflow(a, b, &result)`. C++26: `std::add_sat`, `std::sub_sat`,
  `std::mul_sat`, `std::div_sat`, `std::saturate_cast` for saturation.
- Division and modulo: check divisor ≠ 0 and `INT_MIN / -1`.
- Shifts: shift count must be `< bit width` and `>= 0`. Do not shift negative values. Shift
  unsigned types only.
- Integer promotion: `std::uint8_t a, b; a + b` is `int`. Cast the result back explicitly and check range.
- Use `<bit>` (C++20) for bit work: `std::popcount`, `std::countl_zero`, `std::bit_width`,
  `std::has_single_bit`, `std::rotl`, `std::byteswap` (C++23), `std::endian`.
- Use unsigned types for bit masks and modular arithmetic only, not to say "this cannot be negative".

## 7. Floating point

- Never compare floats with `==` or `!=` (exception: comparison with exact sentinel values you assign yourself, or `0.0` after a check you control). Use a tolerance that fits the scale:
  ```cpp
  [[nodiscard]] inline bool nearlyEqual(float a, float b,
                                        float absTol = 1.0e-6F, float relTol = 1.0e-5F) noexcept
  {
    const float diff = std::fabs(a - b);
    return (diff <= absTol) || (diff <= relTol * std::fmax(std::fabs(a), std::fabs(b)));
  }
  ```
- Check `std::isfinite()` for all external inputs (sensors, messages, files). NaN fails every comparison, so `if (x > limit)` lets NaN through. Write checks that reject NaN: `if (!(x <= limit))`.
- Write float literals with a suffix that matches the type: `0.5F` for `float`. Write constants as numbers, not as run-time calculations, or compute them with `constexpr`.
- Division by a constant: multiplication by the reciprocal is faster, but the result can differ in the last bit. Compilers do it automatically only for powers of two (or with `-ffast-math`). Apply it by hand only when the precision change is acceptable, and use a named `constexpr` reciprocal.
- Do not use `-ffast-math` in safety or numerically sensitive code: it breaks NaN/Inf checks and IEEE rules. Use specific flags (`-fno-math-errno`, `-fno-trapping-math`) only with analysis.
- Some safety standards require one floating operation per statement, to control intermediate precision and to make results reproducible. Follow the project rule. Know `FLT_EVAL_METHOD` and x87 extended precision on old x86 targets.
- Float to integer conversion is UB when the value is out of range (and for NaN). Check range first.
- Accumulation: sum many values with Kahan/pairwise summation or `double` accumulators when error matters.
- Consider fixed-point arithmetic where the hardware has no FPU or where determinism across platforms is required.

## 8. Expressions and statements

- One side effect per expression. Do not read and write the same variable in one expression.
- Use parentheses to show precedence for mixed operators (`&&`/`||`, bit operators, shifts).
- Avoid function calls inside complex conditions when the call has side effects or can fail; assign the result to a named variable first (several safety standards require this). It also makes debugging easier.
- `switch`: handle every `enum class` value; for enums, prefer no `default` so that the compiler warns about new values; add a `default` that goes to the error path in SAFETY-CRITICAL code if the project requires it. Mark intentional fall-through with `[[fallthrough]]`. Every non-empty `case` ends with `break`, `return`, `[[fallthrough]]`, or `throw`.
- `if`/`else if` chains end with an `else` (MISRA style), also if it only reports an error.
- No assignment in conditions (except the C++17 `if (init; cond)` form).
- Use `if (auto it = m.find(k); it != m.end())` and structured bindings to limit scope.
- Loops: one loop counter; do not change it in the body; use range-for or algorithms when possible. Every loop in real-time code needs a known maximum iteration count.
- No `goto`. No `setjmp`/`longjmp`.
- No comma operator except in `for` headers. No unused results; mark intentional discards with `static_cast<void>(f())` or `std::ignore = f();`.
- Dead code and unreachable code: remove them. No commented-out code.
- `std::unreachable()` (C++23) only after a real proof; in SAFETY-CRITICAL code prefer an explicit error path.

## 9. Undefined behavior catalog (blockers)

Check every change against this list.

| Category | Examples | Prevention |
|---|---|---|
| Uninitialized read | local scalar without initializer, member not set in a constructor | initialize at declaration, default member initializers |
| Out of bounds | `a[i]` with bad `i`, `end()` dereference, `memcpy` too long | `std::array`/`std::span`, `.at()` in non-hot code, explicit checks |
| Null dereference | unchecked pointer from API | references for "never null", `gsl::not_null`, checks at boundaries |
| Dangling | reference/pointer/`string_view`/`span`/iterator to a dead object | see `06-memory-ownership-lifetime.md` section 7 |
| Use after move | reading a moved-from object (valid but unspecified for std types) | do not use after `std::move`, except assign/destroy |
| Signed overflow | `INT_MAX + 1`, `abs(INT_MIN)` | checks, wider types, saturating math |
| Bad shifts | shift ≥ width, negative shift, left shift of negative | check count, unsigned operands |
| Division | by zero, `INT_MIN / -1` | check before |
| Strict aliasing | `*reinterpret_cast<float*>(&u32)` | `std::bit_cast`, `std::memcpy` |
| Misalignment | casting a `char*` buffer to `std::uint32_t*` | `alignas`, `std::memcpy`, `std::assume_aligned` only when true |
| Object lifetime | using storage without constructing an object | placement new + `std::launder` where needed, `std::start_lifetime_as` (C++23), `std::construct_at` (C++20) |
| Data race | unsynchronized write + read from two threads | mutex, atomics, ownership transfer |
| Invalid enum value | `static_cast<E>(n)` outside the range of an enum without fixed underlying type | fixed underlying type, validate before cast |
| Float to int out of range | `static_cast<int>(1e20f)`, NaN to int | range check |
| Modifying a `const` object | `const_cast` then write | never |
| Infinite loop without side effects | `while (true) {}` without I/O, volatile, atomic | add a side effect or a wait |
| ODR violation | different definitions of one inline function/class in two TUs | headers only, no macros that change layouts between TUs |
| Bad `printf` format | wrong specifier | `std::format`/`fmt` (C++20) |
| Virtual call in ctor/dtor | calls the base version (pure virtual → UB) | do not call virtual functions there |
| Missing return | non-void function without `return` on a path | return a value on every path |
| Sequence | `i = i++ + 1` (pre-C++17 rules), unspecified evaluation order of arguments | one side effect per expression |
