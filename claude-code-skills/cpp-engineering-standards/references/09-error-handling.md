# 09 — Error handling

## 1. Decide the strategy early (CG E.1)

Choose one primary mechanism per component and document it in the component README or design.
Mixing mechanisms without rules causes lost errors.

| Mechanism | Use when | Avoid when |
|---|---|---|
| Exceptions | GENERAL context; errors are rare; failure crosses many layers; constructors that can fail | `-fno-exceptions`; hard real-time paths; SAFETY-CRITICAL without a qualified runtime; across C/ABI boundaries |
| `std::expected<T, E>` (C++23) / `tl::expected` / project `Result<T, E>` | Expected, recoverable failures; no exceptions; caller must decide | Errors that no caller can handle (use terminate/safe state) |
| `std::optional<T>` | "Not found" / "no value" with no reason needed | When the caller needs to know why |
| Error code / status enum + out parameter | C interfaces, legacy, SAFETY-CRITICAL code with strict rules | New C++ APIs when `expected` is possible |
| `std::error_code` / `std::error_condition` | OS/system-style errors, extensible categories | Simple local enums |
| Assertion / contract violation | Programming errors (bugs), broken invariants | Run-time conditions that can really happen (input, I/O) |
| Terminate / safe state | Unrecoverable faults, corrupted state | Recoverable conditions |

Classify every failure:
- **Programming error** (precondition violation, broken invariant): assert/contract; in SAFETY-CRITICAL → safe state + fault report.
- **Recoverable run-time error** (bad input, timeout, missing file): return/throw a typed error; the caller handles it.
- **Unrecoverable environment error** (memory corruption, hardware fault): go to the safe state; do not continue.

## 2. Universal rules

- Never ignore an error. Every result of a function that can fail is checked (`[[nodiscard]]`
  on the result type and functions).
- Handle an error at the level that has enough context to decide. Do not catch or check only to
  log and continue with bad data.
- Error messages and codes say what failed, with which input, and where (`std::source_location`, C++20).
- Keep the program state valid after an error (at least the basic exception/error guarantee).
- Release resources on every path — RAII does it automatically (CG E.6).
- Do not use `errno`-style global error state (CG E.28). Wrap such APIs at the boundary.
- Do not use error handling for normal control flow (CG E.3).

## 3. Exceptions (when enabled)

- Throw to signal that a function cannot do its task (CG E.2).
- Throw by value, catch by `const&` (CG E.15). Never throw pointers or built-in types (CG E.14).
- Use purpose-designed exception types derived from `std::exception` (often `std::runtime_error`/`std::logic_error`).
- Catch only what you can handle (CG E.17). No empty `catch`. `catch (...)` only at thread/task boundaries, `main`, callbacks from C code, and destructors' internals — log and terminate or translate.
- Never let an exception escape: destructors, `noexcept` functions, C callbacks, thread entry functions (uncaught → `std::terminate`), `swap`, move operations (CG E.16).
- Exception safety guarantees — document per function:
  - **No-throw**: `noexcept`; destructors, swap, moves, cleanup.
  - **Strong**: success or no change (commit-or-rollback; copy-and-swap).
  - **Basic**: invariants hold, no leaks, state may change.
- **Commit after success.** Do every step that can fail (allocation, copy, conversion, user code)
  **before** any step that destroys or changes existing data or observable state. Only then commit
  with operations that cannot fail (moves of `noexcept` types, swaps, pointer updates, counters).
  - Mutators of containers and caches give the strong guarantee when they would otherwise lose
    user data (for example: insert the new entry first, then evict the old one — never evict first).
  - The commit steps themselves must not throw. Check it: `static_assert` that the hash, equality,
    and move operations they use are `noexcept`, erase by iterator instead of by key where possible,
    and keep any step that may throw in the "before" part.
  - Observable side effects (statistics, recency, logging of "success") happen after the fallible
    step (for example after the copy of a returned value).
  - If only the basic guarantee is possible, document it on the function.
- Rethrow with `throw;` (not `throw e;`, which slices). Use `std::throw_with_nested` to add context.
- Pass exceptions between threads with `std::exception_ptr` (`std::promise::set_exception`).
- Constructors: throw on failure (no zombie objects). With exceptions off: private constructor + factory returning `expected`.

## 4. `std::expected` / Result types

```cpp
enum class [[nodiscard]] ParseError : std::uint8_t { Empty, InvalidChar, OutOfRange };

[[nodiscard]] std::expected<std::uint16_t, ParseError> parsePort(std::string_view text) noexcept
{
  if (text.empty())
  {
    return std::unexpected{ParseError::Empty};
  }
  std::uint16_t value{0U};
  const auto [ptr, ec] = std::from_chars(text.data(), text.data() + text.size(), value);
  if (ec == std::errc::result_out_of_range)
  {
    return std::unexpected{ParseError::OutOfRange};
  }
  if ((ec != std::errc{}) || (ptr != text.data() + text.size()))
  {
    return std::unexpected{ParseError::InvalidChar};
  }
  return value;
}

// C++23 monadic chaining
const auto endpoint = parsePort(portText)
                        .and_then(validatePortRange)
                        .transform([](std::uint16_t p) { return Endpoint{kHost, p}; });
```

- Error type: small `enum class`, or a struct with code + context. Keep it cheap to copy.
- Never call `.value()` without a check in code that runs with exceptions off (`value()` throws `bad_expected_access`). Use `has_value()` / `operator bool` first, or `value_or()`.
- Prefer `std::expected`, `std::optional`, or `std::variant` to a custom result type. If you write one,
  a wrong access (`value()` on an error, `error()` on a success) must **never return a plausible
  fabricated value** (a default object, a "first" error code). It must be impossible (only
  `visit`/pattern access), or assert/terminate, or throw. A silent fallback hides a programming error.
- Monadic operations (`and_then`, `or_else`, `transform`, `transform_error`) exist for `std::optional` and `std::expected` in C++23.
- Propagation helper macros (like `TRY(expr)`) are acceptable when the project defines them; keep one helper per project.

## 5. Error codes (legacy / C interfaces / strict SAFETY-CRITICAL)

- Use a strongly typed `enum class` status, not `int`. Make it `[[nodiscard]]`.
- One success value (`Ok`), specific failure values. No "magic" negative numbers.
- Outputs through `T&` parameters are written only on success; document their value on failure.
- At C boundaries convert to/from C codes in one adapter layer.
- `errno` and similar global error state:
  - Read it only when the C API documents that it sets it, and only for a failed call.
  - Set `errno = 0` before the call only for APIs that report errors **only** through `errno`
    (for example `strtol`).
  - Read it immediately after the call (any other library call can overwrite it) and convert it at
    once to a typed error (`std::error_code{errno, std::generic_category()}` or a project enum).

## 6. Assertions and contracts

- Use assertions for invariants and preconditions that are bugs if violated. Use error returns
  or exceptions for conditions that can really happen (input, I/O, resources).
- No side effects inside assertions: `assert()` disappears when `NDEBUG` is defined.
- `static_assert` for everything checkable at compile time.
- Decide the release-build policy per project:
  - GENERAL: `assert()` in debug builds is acceptable for internal invariants.
  - SAFETY-CRITICAL: never rely on `assert()`. Use a project assertion macro that stays active in
    release builds, reports the fault, and goes to the safe state (section 7) — or explicit checks
    with error returns.
- C++26 contracts give standard syntax: `pre(cond)` and `post(r: cond)` on declarations,
  `contract_assert(cond)` in bodies. The evaluation semantic (ignore, observe, enforce,
  quick-enforce) is chosen at build time. Use `enforce` for checks that protect safety.
- Until the compiler supports contracts, use `gsl::Expects`/`gsl::Ensures` (GSL) or one project
  macro. Use one mechanism per project.

```cpp
// C++26 (check compiler support before use)
[[nodiscard]] double squareRoot(double x)
  pre(x >= 0.0)
  post(r: r >= 0.0);
```

## 7. Defensive programming and safe state (safety-critical code)

- **Validate at the boundary:** pointers, indexes, ranges, enum values received as integers, and
  floating-point finiteness. Define what the function returns for invalid input. Never continue
  silently with invalid data. (General input-validation rules: `03-functions-interfaces.md` section 9.)
- **Define a safe state** for each component (for example: stop output, hold the last valid value
  for N cycles then degrade, request a fallback, controlled stop). Document it in the design and code.
- **Detect faults:** range checks, plausibility checks (cross-check redundant values), timeouts,
  heartbeat counters, checksums and sequence counters on communication, stuck-value detection.
- **React in bounded time.** Report the fault (event/log) with enough context.
- **Degrade gracefully:** limited functions instead of total loss, when the safety concept allows.
- **No silent recovery from corrupted state.** Restart/reinitialize only through a defined path.
- **Use the last valid value only for a limited time,** then go to the safe state.

```cpp
enum class [[nodiscard]] Status : std::uint8_t { Ok, OutOfRange, Stale, Implausible };

// SAFETY: Validates the tank level before the pump controller uses it.
// FAILURE MODE: A wrong level value lets the tank overflow.
// MITIGATION: Range, freshness, and rate-of-change checks; safe state after kMaxInvalidCycles.
Status LevelMonitor::update(float levelPercent, Timestamp sampleTime, Timestamp now) noexcept
{
  Status status{Status::Ok};
  if (!std::isfinite(levelPercent) || (levelPercent < 0.0F) || (levelPercent > 100.0F))
  {
    status = Status::OutOfRange;
  }
  else if ((now - sampleTime) > kMaxSampleAge)
  {
    status = Status::Stale;
  }
  else if (std::fabs(levelPercent - m_lastValid) > kMaxChangePerCycle)
  {
    status = Status::Implausible;
  }
  else
  {
    m_lastValid = levelPercent;
    m_invalidCycles = 0U;
  }

  if (status != Status::Ok)
  {
    ++m_invalidCycles;
    if (m_invalidCycles >= kMaxInvalidCycles)
    {
      enterSafeState();                                      // for example: stop the pump
    }
  }
  return status;                                             // single exit point
}
```

Mark safety-relevant code with searchable comments: `SAFETY:` (what the code protects),
`FAILURE MODE:` (what goes wrong if it fails), `MITIGATION:` (how the design controls it).

## 8. Error handling checklist

- [ ] One documented strategy per component; no unplanned mix.
- [ ] No ignored return values; error types are `[[nodiscard]]`.
- [ ] No empty `catch`; `catch (...)` only at boundaries, with a reason.
- [ ] Destructors, moves, swap, and `noexcept` functions cannot throw.
- [ ] State stays valid after every error; no leaks (RAII).
- [ ] Programming errors are asserted; run-time errors are returned/thrown; fatal errors go to the safe state.
