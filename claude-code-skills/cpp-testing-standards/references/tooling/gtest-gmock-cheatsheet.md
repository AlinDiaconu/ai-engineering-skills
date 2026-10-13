# GoogleTest and GoogleMock cheat sheet

For a first unit test and a complete example, read `gtest-unit-test-guide.md`. Check the project's GoogleTest version. Names below are for recent versions (`TEST_SUITE`, `INSTANTIATE_TEST_SUITE_P`).

## Test macros

| Macro | Use |
|---|---|
| `TEST(Suite, Name)` | A test with no fixture |
| `TEST_F(Fixture, Name)` | A test with a fixture class |
| `TEST_P(Fixture, Name)` | A parameterized test. Fixture derives from `TestWithParam<T>`. |
| `INSTANTIATE_TEST_SUITE_P(Prefix, Fixture, generator, [namegen])` | Give values to a `TEST_P` |
| `TYPED_TEST(Fixture, Name)` | Same test on many types. Use `TYPED_TEST_SUITE(Fixture, Types)`. |
| `GTEST_SKIP() << "reason";` | Skip at run time with a reason |
| `SCOPED_TRACE("msg")` | Add context to failures in helper functions |

## Assertion macros

`ASSERT_*` stops the test on failure. `EXPECT_*` continues.

| Macro | Meaning |
|---|---|
| `EXPECT_EQ(a, b)` / `NE` / `LT` / `LE` / `GT` / `GE` | Compare. Expected value first is the usual style. |
| `EXPECT_TRUE(x)` / `EXPECT_FALSE(x)` | Boolean. Use only for real booleans. |
| `EXPECT_NEAR(a, b, tol)` | Floating point within a tolerance |
| `EXPECT_FLOAT_EQ` / `EXPECT_DOUBLE_EQ` | Within 4 ULP |
| `EXPECT_STREQ(a, b)` | C strings |
| `EXPECT_THROW(stmt, Type)` | Throws exactly that type (or derived) |
| `EXPECT_NO_THROW(stmt)` | Does not throw |
| `EXPECT_THAT(value, matcher)` | Use a matcher (gMock) |
| `EXPECT_DEATH(stmt, regex)` | Process dies. Name the suite `*DeathTest`. |
| `EXPECT_PRED_FORMAT1(f, v)` | Custom check with a good message |
| `EXPECT_FATAL_FAILURE` | Test a helper that asserts (rare) |

Add a message: `EXPECT_EQ(a, b) << "context " << x;`

Notes:

- Use `ASSERT_*` inside a `void` helper function. A fatal failure only leaves that helper. Check with `ASSERT_NO_FATAL_FAILURE(helper())`.
- Do not use `ASSERT_*` in a constructor or destructor.
- `EXPECT_EQ(ptr, nullptr)` may not compile on old versions. Use `EXPECT_EQ(ptr, nullptr)` with `nullptr` (not `NULL`), or `EXPECT_TRUE(ptr == nullptr)`.

## Fixture

```cpp
class MyTest : public ::testing::Test {
protected:
    void SetUp() override {}      // before each test
    void TearDown() override {}   // after each test
    static void SetUpTestSuite() {}    // once per suite (immutable data only)
    static void TearDownTestSuite() {}
};
```

## Common matchers

| Matcher | Use |
|---|---|
| `Eq`, `Ne`, `Lt`, `Le`, `Gt`, `Ge` | Compare |
| `DoubleNear(x, tol)`, `FloatNear` | Floating point |
| `IsNull()`, `NotNull()`, `Pointee(m)` | Pointers |
| `StartsWith`, `EndsWith`, `HasSubstr`, `MatchesRegex` | Strings |
| `ElementsAre(...)`, `UnorderedElementsAre(...)`, `Contains(m)`, `Each(m)`, `SizeIs(n)`, `IsEmpty()` | Containers |
| `Field(&T::member, m)`, `Property(&T::method, m)` | Members of a struct or class |
| `AllOf(...)`, `AnyOf(...)`, `Not(m)` | Combine |
| `Optional(m)`, `VariantWith<T>(m)` | `std::optional`, `std::variant` |
| `_` | Any value |

## Mocks

```cpp
class MockSender : public ISender {
public:
    MOCK_METHOD(bool, Send, (const Message& msg), (override));
    MOCK_METHOD(Status, Status, (), (const, override));
};
```

| Item | Use |
|---|---|
| `EXPECT_CALL(mock, Method(args))` | Required interaction |
| `ON_CALL(mock, Method(args)).WillByDefault(Return(x))` | Default behavior (stub) |
| `.Times(n)` / `AtLeast(n)` / `Between(a, b)` | Call count |
| `.WillOnce(Return(x))` / `.WillRepeatedly(Return(x))` | Return values |
| `Invoke(f)` / `DoAll(...)` / `SetArgPointee<N>(v)` | Side effects |
| `InSequence seq;` / `.After(other)` | Order (use only when the order matters) |
| `::testing::StrictMock<T>` | Any call without an expectation fails |
| `::testing::NiceMock<T>` | Calls without an expectation are ignored |
| `Mock::VerifyAndClearExpectations(&mock)` | Check now, then reset |

Rules:

- Set expectations **before** the Act.
- Mock **interfaces** with a virtual destructor.
- Use matchers, not full-object equality, when only some fields matter.
- `MOCK_METHOD` with `noexcept` or `const` needs the right specifiers in the 4th argument.

## Run flags

See the command table in `build-and-run.md`. Also useful: `--gtest_filter=A.*:-A.Slow*`, `--gtest_catch_exceptions=0`, `--gtest_also_run_disabled_tests`.

## Print helpers

Define `PrintTo(const T&, std::ostream*)` or `operator<<` for your types. Failure messages then show readable values.

## Common mistakes

| Mistake | Fix |
|---|---|
| `EXPECT_TRUE(a == b)` | `EXPECT_EQ(a, b)` (better message) |
| `EXPECT_EQ(floatA, floatB)` | `EXPECT_NEAR` or `EXPECT_FLOAT_EQ` |
| Expectation set after the call | Set it before the Act |
| Mock with a non-virtual method | Make it virtual, or use a template seam |
| Test depends on `SetUpTestSuite` state that changes | Use `SetUp` |
| `DISABLED_` prefix left in code | Fix the test or remove it. Link an issue. |
| Static mock not reset | Reset in `TearDown` |
