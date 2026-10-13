# Unit tests (C++)

## What

A test of **one class or function** in isolation. For the principles of a unit test and a complete, built example, see `../tooling/gtest-unit-test-guide.md`. All collaborators are replaced (fakes, stubs) or are cheap, pure objects.

## Why

- Fast feedback (milliseconds).
- A failure points to one unit.
- Safe refactoring.
- Base of the test pyramid.

## When

- Logic, calculations, limits, error handling, state machines (per unit).
- Every new class and every changed function.
- Every bug fix (as a regression test).

Do **not** use for: wiring, real timing, real I/O, or behavior that needs many real parts (use integration or component tests).

## How

1. Create the unit with fakes for its dependencies (constructor injection).
2. Put it in the start state.
3. Do **one** action.
4. Check the result and observable effect.

```cpp
// Verifies: REQ-XXX
TEST(SpeedLimiter, SpeedAboveMax_ReturnsMax)
{
    const SpeedLimiter limiter{kMaxSpeed};

    EXPECT_NEAR(limiter.Limit(kMaxSpeed + 1.0), kMaxSpeed, kSpeedTol);
}
```

## Best practices

- **One behavior per test.** Name: `Unit_Condition_Expected`.
- Test the **public interface**. Do not test private members.
- Use **fakes** for time, I/O, and hardware. No `sleep`, no files, no network.
- Cover: valid input, invalid input, boundaries, error paths, and the failure behavior that the interface documents.
- Use **parameterized tests** for many values (`TEST_P`).
- Keep each test **under a few milliseconds**. A slow unit test is a design signal.
- Test **special member functions** when the class owns a resource: copy, move, assign, destroy, self-assign, moved-from state.
- Test **`noexcept`** with `static_assert(noexcept(f()))`. Test the **strong guarantee** by making a collaborator throw, then checking that the state did not change.
- Test **const-correctness** and **output parameters** (on failure, is the output unchanged?).
- Use **death tests** only for a defined abort or `assert` behavior. Name the suite `*DeathTest`. Keep them few. They are slow.
- Check **resource release** (RAII) with a counting fake: acquired count equals released count.
- Do not test the standard library or the compiler.

## C++ details

| Topic | Rule |
|---|---|
| **Floating point** | `EXPECT_NEAR` with a stated tolerance |
| **Integers** | Test limits (`std::numeric_limits`), overflow, sign mix, narrowing |
| **Strings** | Empty, one char, long, embedded NUL, non-ASCII, invalid UTF-8 |
| **Containers** | Empty, one, many, duplicates, order, capacity limits |
| **Optional / expected / variant** | Test every alternative |
| **Templates** | Instantiate with several types (`TYPED_TEST`): small, large, move-only, non-default-constructible |
| **Exceptions** | `EXPECT_THROW(stmt, ExactType)`. Check what stays valid after the throw. |
| **Time** | Inject a clock |
| **Static / global** | Avoid. Reset in `SetUp` and `TearDown` if you cannot avoid. |

## Pitfalls

| Pitfall | Fix |
|---|---|
| Testing private members with `friend` or `#define private public` | Test through the public interface. Extract a class if needed. |
| One test with many Acts | Split |
| The test repeats the production formula | Use a hand-checked fixed value |
| Hidden dependence on test order | Reset state. Run shuffled. |
| Too many mocks | Use fakes or real cheap objects |
| `EXPECT_TRUE(a == b)` | `EXPECT_EQ(a, b)` |

## Checklist

- [ ] Requirement or risk is named.
- [ ] Valid, invalid, boundary, and error cases exist.
- [ ] No sleep, no real I/O, no hidden state.
- [ ] Fast and deterministic.
- [ ] Can fail (proved).
