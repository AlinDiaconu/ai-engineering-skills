# Test design techniques (C++)

Use these to choose a **small set of good cases** from an unlimited number of inputs.
Write the case table before the code.

| ID | Requirement | Technique | Input / state | Expected |
|---|---|---|---|---|

## Black-box techniques (from the specification)

### Equivalence partitioning (EP)

- **What:** split inputs into classes where the code behaves the same. Test one value per class.
- **How:** list valid classes and invalid classes for each input.
- **Example:** `age` 0–17 child, 18–64 adult, 65+ senior, negative invalid → at least 4 cases.

### Boundary value analysis (BVA)

- **What:** defects gather at the edges of classes.
- **How:** test min−1, min, min+1, a normal value, max−1, max, max+1.
- **C++ extras:** zero, empty, one, many, `std::numeric_limits<T>::min()/max()/lowest()/epsilon()`, exactly at a threshold, one tick before and after a timeout.
- **Example:** limit is "up to 100". Test 99, 100, 101.

### Decision table

- **What:** a table of condition combinations and the action for each.
- **When:** rules with several conditions.
- **How:** conditions as rows, rules as columns, actions at the bottom. Test each column.

| | R1 | R2 | R3 | R4 |
|---|---|---|---|---|
| isMember | Y | Y | N | N |
| total > 100 | Y | N | Y | N |
| **Discount** | 15% | 5% | 5% | 0% |

Use `TEST_P` with one row per column.

### State transition testing

- **What:** test a state machine.
- **How:** draw states, events, guards, actions. Test:
  1. every state (reach it),
  2. every valid transition,
  3. every **invalid** event in every state (rejected or ignored in a defined way),
  4. important sequences (length 2 or more).
- Check the new state **and** the actions and outputs.

### Pairwise (combinatorial) testing

- **What:** test every pair of parameter values at least once, instead of every full combination.
- **Why:** most defects come from one or two parameters.
- **How:** use a tool (PICT, AllPairs). Add high-risk full combinations by hand. Feed the rows into `TEST_P`.

### Use-case / scenario testing

- One test per path: main flow, alternative flows, error flows. Check the end result and side effects.

### Error guessing: C++ checklist

- Null pointer, empty container, zero size, one element, maximum size.
- Negative values, zero, `NaN`, `+inf`, `-inf`, denormals, `-0.0`.
- Integer overflow, underflow, sign mix, narrowing, shift by too much.
- Empty string, embedded NUL, non-ASCII, invalid UTF-8, very long string.
- Wrong order of calls, double init, use before init, use after shutdown, double free by design.
- Stale data, missing data, duplicate data, reordered data.
- Wrap-around: counters, timestamps, angles.
- Timeout exactly at the limit, clock goes backward.
- Moved-from object, self-assignment, copy of an object that owns a resource.
- Exception thrown from a callback, a destructor, or a constructor.

### Exploratory testing

- Learn, design, and run tests at the same time. Write a **charter** ("Explore X with Y to find Z"). Time-box 60–90 min. Turn the findings into automated tests.

## White-box techniques (from the code)

Statement, branch, condition, MC/DC, loop, and path coverage: `../practice/coverage-and-metrics.md` and `../types/white-box-tests.md`.

## Property-based design

- State a rule that must hold for **all valid inputs** and let a tool generate inputs.
- Common properties: round trip, idempotence, invariant, model comparison, monotonic, bounds, symmetry, metamorphic, no crash.
- See `../types/property-based-tests.md`.

## Fuzzing and mutation

- **Fuzzing:** random or mutated input to find crashes. See `../types/fuzz-tests.md`.
- **Mutation:** change the code on purpose and check that tests fail. See `../types/mutation-tests.md`.

## Choosing techniques

| Situation | Use |
|---|---|
| Numeric input with ranges | EP + BVA |
| Rules with several conditions | Decision table |
| Modes, protocol, workflow | State transition |
| Many configuration options | Pairwise |
| Parser or untrusted input | Property-based + fuzz |
| High-risk code | Add MC/DC and mutation |
| New feature, unknown risk | Exploratory, then automate |
| Large structured output | Golden |
| Template or type API | Compile-time tests |

## From cases to `TEST_P`

```cpp
struct Row { const char* name; double in; double expected; };

class LimitTest : public ::testing::TestWithParam<Row> {};

TEST_P(LimitTest, MatchesCaseTable)
{
    const Row& r = GetParam();
    EXPECT_NEAR(Limit(r.in), r.expected, kTol) << r.name;
}

INSTANTIATE_TEST_SUITE_P(
    CaseTable, LimitTest,
    ::testing::Values(Row{"below",  99.0,  99.0},
                      Row{"at",    100.0, 100.0},
                      Row{"above", 101.0, 100.0}),
    [](const ::testing::TestParamInfo<Row>& p) { return std::string{p.param.name}; });
```

## Numeric, data, and interface rules

### Numeric

- Never compare floating-point values with `==`. Use `EXPECT_NEAR` and state the tolerance and why.
- Test NaN and infinity for every floating-point input.
- Test integer overflow and underflow at the type limits. Test signed and unsigned mixing, narrowing, and rounding direction.
- Test unit conversion at both ends (m/s and km/h).
- Test angle wrap-around (−π and +π), time wrap-around, and counter roll-over.
- Test fixed-point scaling at the minimum and maximum of the range.

### Data

- Use named constants and builders with safe defaults.
- Do not copy the production formula to compute the expected value.
- For recorded data (golden files), state the source, version, and date.

### Interface

- Test each public function for valid input, invalid input, and the documented failure behavior (error code, exception, return value).
- Test each documented precondition. A violated precondition gives the defined reaction.
- Test copy, move, and destruction when the class owns a resource.
- Check `noexcept` with `static_assert(noexcept(f()))`.
