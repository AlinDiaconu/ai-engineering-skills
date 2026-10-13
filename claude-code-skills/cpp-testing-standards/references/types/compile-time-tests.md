# Compile-time tests (C++)

## What

Tests that the **compiler** checks: `static_assert`, `constexpr` evaluation, type traits, concepts, and "this code must **not** compile" checks.

## Why

- Zero run-time cost. A defect fails the **build**, not a run.
- The best check for **types, templates, concepts, constexpr code, and API contracts**.
- Critical code benefits from checks that cannot be skipped at run time.

## When

- Template and concept constraints.
- `constexpr` and `consteval` functions (tables, conversions, math).
- Type properties: size, alignment, trivially copyable, `noexcept`, standard layout.
- Wire-format and struct layout (ABI and message layouts).
- Strong types (units): "meters plus seconds must not compile".
- Public API: some calls must be rejected at compile time.

## How

### 1. `static_assert` for properties

```cpp
static_assert(sizeof(FrameHeader) == 16, "wire layout: header is 16 bytes");
static_assert(alignof(FrameHeader) == 4);
static_assert(std::is_trivially_copyable_v<FrameHeader>);
static_assert(std::is_standard_layout_v<FrameHeader>);
static_assert(noexcept(std::declval<Speed>().ToMetersPerSecond()));
static_assert(std::is_nothrow_move_constructible_v<MessageBuffer>);
```

### 2. `constexpr` function tests

```cpp
static_assert(KmhToMs(36.0) == 10.0);
static_assert(Crc8(kKnownBytes) == 0xA7);   // run at compile time: no UB allowed
```

- `constexpr` evaluation also **detects undefined behavior** (overflow, out-of-bounds) at compile time.
- Add the **same** checks as normal run-time tests too, when you need coverage or sanitizers.

### 3. Concepts and constraints

```cpp
template <typename T>
concept SpeedLike = requires(T t) { { t.ToMetersPerSecond() } -> std::convertible_to<double>; };

static_assert(SpeedLike<Speed>);
static_assert(!SpeedLike<int>);
```

### 4. "Must not compile" tests

Use a **detection idiom** or a concept to test that an expression is **invalid**:

```cpp
template <typename A, typename B>
concept CanAdd = requires(A a, B b) { a + b; };

static_assert(CanAdd<Meters, Meters>);
static_assert(!CanAdd<Meters, Seconds>, "unit mix must not compile");
```

For failures that a concept cannot show (hard errors inside a body), use a **CMake compile-fail test**:

```cmake
add_executable(unit_mix_must_fail EXCLUDE_FROM_ALL unit_mix_must_fail.cpp)
add_test(NAME unit_mix_must_fail
         COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR} --target unit_mix_must_fail)
set_tests_properties(unit_mix_must_fail PROPERTIES WILL_FAIL TRUE)
```

Make sure it fails **for the right reason** (check the error text with `PASS_REGULAR_EXPRESSION` or a small script).

### 5. Check layout and ABI

- `static_assert` on `sizeof`, `alignof`, and `offsetof` for wire and shared-memory structs.
- Check the **endianness** assumption (C++20): `static_assert(std::endian::native == std::endian::little)`. Concepts also need C++20.
- Use a **golden test** on the byte stream too (`golden-and-snapshot-tests.md`).

### 6. Test the same code with several types

- Use `TYPED_TEST` (GoogleTest) to run one test on several types at run time.
- Use `static_assert` in a template to check the same property for a list of types.

## Best practices

- **Put compile-time checks near the type** or in a dedicated `*_static_test.cpp` file that is part of the normal build.
- **Give each `static_assert` a message** that says what the rule is and why.
- **Keep them cheap.** They run at every build.
- **Test the negative case** (what must not compile) as well as the positive one.
- **Do not replace run-time tests.** Compile-time tests show type and constant properties. They do not show run-time behavior.
- **Use `constexpr` tests** to detect UB in small pure functions.
- **Run on all supported compilers.** Some constructs differ.
- **Avoid macros** for these tests when a template or concept can do the job.

## Pitfalls

| Pitfall | Fix |
|---|---|
| "Must not compile" test fails for another reason | Check the error text |
| `static_assert` with no message | Add a message that explains the rule |
| Layout checked on one compiler only | Build on all supported compilers and targets |
| Compile-fail test slows the build | Keep it `EXCLUDE_FROM_ALL`, run as a CTest |
| Only compile-time checks, no run-time tests | Add both |

## Checklist

- [ ] Layout, size, and alignment of wire structs are asserted.
- [ ] Concepts and constraints have positive and negative checks.
- [ ] Important `constexpr` functions are checked at compile time.
- [ ] Each `static_assert` has a clear message.
- [ ] Checked on every supported compiler.
