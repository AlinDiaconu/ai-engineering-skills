# Writing tests: structure, naming, assertions, anti-patterns

Owns the rules for **how a test looks**. For GoogleTest syntax, see `../tooling/gtest-gmock-cheatsheet.md`. To check finished tests, see `review.md`.

## Structure

- One behavior per test. One reason to fail.
- Arrange, Act, Assert. Separate the three parts with a blank line. One Act per test.
- No `if`, loops, or `try/catch` logic in a test body. Use `TEST_P`.
- `TEST` for no shared setup. `TEST_F` when two or more tests share setup. `TEST_P` or `TYPED_TEST` for the same check on many values or types. Do not copy and paste tests.
- Put the requirement ID in a comment above the test (`// Verifies: REQ-XXX`).

```cpp
// Verifies: REQ-XXX
TEST_F(SessionStateMachineTest, Connected_HeartbeatLost_StartsReconnectWithinLimit)
{
    // Arrange
    clock_.SetTime(kStart);
    machine_.Start();

    // Act
    clock_.Advance(kHeartbeatTimeout + kOneTick);
    machine_.Step();

    // Assert
    EXPECT_EQ(machine_.State(), SessionState::Reconnecting);
}
```

## Naming

- The name tells **unit**, **condition**, **result**. Format: `Unit_Condition_ExpectedResult`.
  - Example: `RouteSelector_PrimaryLinkLost_SwitchesToBackupRoute`
  - With `TEST(Suite, Name)`, the suite is the unit and the name is `Condition_ExpectedResult`.
- A failing name must tell what broke without opening the code.
- Do not use `Test1`, `Works`, `Basic`, `HappyPath`.
- Update the name when the behavior changes.

## Fixtures and data

- Use a fixture only for shared setup. Keep it small. Use composition and helpers, not fixture inheritance chains.
- Do not hide important values in the fixture. The reader must see them in the test.
- Reset all static and global state in `SetUp` and `TearDown`. `SetUpTestSuite` only for **immutable**, expensive setup.
- Use **builders** with safe defaults. Set only the values that matter to the test.
- Use named constants (`kHeartbeatTimeout`). No magic numbers.
- Do not copy the production formula to compute the expected value. Use a hand-checked number.
- Test data files: pass the path from the build system. Do not use a relative path from the working directory.

## Parameterized tests

```cpp
struct LimitCase { const char* name; double speed; bool limited; };

class SpeedLimitTest : public ::testing::TestWithParam<LimitCase> {};

TEST_P(SpeedLimitTest, LimitFlagMatchesSpeed)
{
    const auto& c = GetParam();
    EXPECT_EQ(IsLimited(c.speed), c.limited) << c.name;
}

INSTANTIATE_TEST_SUITE_P(
    Boundaries, SpeedLimitTest,
    ::testing::Values(LimitCase{"below", kMax - kEps, false},
                      LimitCase{"at",    kMax,        false},
                      LimitCase{"above", kMax + kEps, true}),
    [](const ::testing::TestParamInfo<LimitCase>& p) { return std::string{p.param.name}; });
```

Give instances readable names (the name generator argument).

## Assertions

- Check the **result and observable effect**, not internal steps.
- Use the most specific macro (`EXPECT_EQ`, `EXPECT_NEAR`, `EXPECT_THAT` + matcher). Do not use `EXPECT_TRUE(a == b)`: the failure message is poor.
- `ASSERT_*` when the test cannot continue (null pointer, empty container). `EXPECT_*` for all other checks.
- Add a message when the cause is not clear (`EXPECT_EQ(a, b) << "after step " << i;`).
- Floating point: `EXPECT_NEAR` with a stated tolerance. `EXPECT_DOUBLE_EQ` only for "within 4 ULP".
- Enums: give a `PrintTo` or `operator<<`, so the failure shows the name.
- Exceptions: `EXPECT_THROW(stmt, ExactType)` and check key content. Do not use `EXPECT_ANY_THROW`.
- Death tests only for a defined abort. Name the suite `*DeathTest`.
- Containers: `ElementsAre`, `UnorderedElementsAre`, `Contains`, `SizeIs`.
- Do not assert on log text or full error strings. Assert on codes and key content.

## Comments

- ASD-STE100. Comment **why**, not what. Name the requirement or the reason for a special value.
- No commented-out tests. No dead code.

## Test code is production code

Apply `cpp-engineering-standards` to test code.

- No undefined behavior, uninitialized data, or data races.
- Same warning flags as production code. Do not silence a warning to make a test build.
- No owning raw pointers and no manual `new` and `delete`. Use values and smart pointers.
- Shared fakes, builders, and matchers live in one test-utility library. Do not copy them between files.
- Do not use `#define private public` or `friend` hacks. Test through the public interface. If a path is not reachable, extract a class.
- Keep each test short. If setup is long, use a fixture or a builder.

## Anti-patterns and fixes

| Anti-pattern | Problem | Fix |
|---|---|---|
| No assertion | Cannot fail | Check the result |
| Tautology (test repeats the production formula) | Always passes | Fixed, hand-checked values |
| Giant test (many behaviors) | Unclear failure cause | Split |
| Logic in tests (`if`, loops) | The test can have its own bugs | `TEST_P` |
| Interdependent tests | Order-dependent failures | Fresh state per test. Run shuffled. |
| Shared mutable or static state | Flaky | Reset in setup. Avoid globals. |
| `sleep` to wait | Slow and flaky | Fake clock, latch, future |
| Over-mocking | Breaks on refactoring | Real objects or fakes (`../foundations/test-doubles.md`) |
| Testing implementation or private members | Brittle, locks the design | Test the public interface |
| Mystery guest (hidden file or global value) | Hard to understand | Put the data in the test or a clear builder |
| Assertion roulette (many asserts, no messages) | Unclear failure | Messages, or split |
| Copy-paste tests | Hard to maintain | `TEST_P`, helpers |
| Hard-coded dates and times | Fails later | Inject a clock |
| Float `==` | Flaky across platforms | `EXPECT_NEAR` |
| `DISABLED_` or skip with no issue | Silent coverage loss | Fix or remove. Link an issue. |
| Snapshot without review | Locks in bugs | Review each change |
| Coverage chasing | False confidence | Mutation testing, assertion review |
| Ice-cream cone (mostly system tests) | Slow, flaky | Move checks down (`../foundations/strategy.md`) |
