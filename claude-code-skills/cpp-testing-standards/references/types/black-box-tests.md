# Black-box tests (C++)

## What

Tests that are designed from the **specification** and the **public interface**. The test author does **not** use the internal code to choose cases.

## Why

- Checks the **requirement**, not the implementation.
- Finds missing or wrong behavior that the code author did not think about.
- Survives refactoring. The tests do not depend on the internals.
- Can be written by a person who did not write the code (independence).

## When

- For every requirement.
- For public APIs, message interfaces, and critical requirements.
- When independence is needed (a second person writes the tests).
- Before you read the code, to avoid bias.

## How

1. Read the **requirement** and the **interface specification** (types, ranges, units, preconditions, timing, errors).
2. Choose cases with these techniques:
   - **Equivalence classes** per input.
   - **Boundary values** at each limit.
   - **Decision table** for rules with several conditions.
   - **State transition** for modes.
   - **Pairwise** for many configuration options.
   - **Error guessing** (NaN, empty, stale, reordered).
3. Write the **case table** (ID, requirement, technique, input, expected).
4. Write the tests through the **public interface** only.
5. Run them. Only then look at coverage to find gaps (see `white-box-tests.md`).

## Example

Requirement: "Output speed shall be limited to 130 km/h. Negative input shall be rejected."

| ID | Technique | Input (km/h) | Expected |
|---|---|---|---|
| TC-01 | BVA | 129.9 | 129.9 |
| TC-02 | BVA | 130.0 | 130.0 |
| TC-03 | BVA | 130.1 | 130.0 |
| TC-04 | EP invalid | −0.1 | rejected |
| TC-05 | BVA | 0.0 | 0.0 |
| TC-06 | Error guessing | NaN | rejected |
| TC-07 | Error guessing | +inf | rejected |

```cpp
struct SpeedCase { double in; Result expected; };

class SpeedLimitBlackBox : public ::testing::TestWithParam<SpeedCase> {};

TEST_P(SpeedLimitBlackBox, MatchesSpecification)   // Verifies: REQ-XXX
{
    const auto& c = GetParam();
    const Result r = Limit(c.in);
    EXPECT_EQ(r.accepted, c.expected.accepted) << "in = " << c.in;
    if (c.expected.accepted) {
        EXPECT_NEAR(r.value, c.expected.value, kTol);
    }
}
```

(In a real test, avoid the `if` by splitting into two parameterized tests.)

## Best practices

- **Use only the public API** and the documented types. No `friend`, no peeking at internals.
- **Derive the expected value from the requirement**, not from running the code.
- **Cover the negative space**: invalid input, wrong state, wrong order, missing data.
- **Test the interface contract**: units, ranges, timing, error codes, ownership, thread safety statement.
- **One requirement, at least one positive and one negative test.**
- **Write a case table** and keep it in the test file or the test report.
- **Keep tests stable** when the implementation changes.
- **Review the tests against the requirement**, not against the code.

## Black-box for C++ interfaces: what to check

| Interface element | Check |
|---|---|
| Parameters | Ranges, null, empty, ownership (who frees?) |
| Return value | Success and every error value |
| Exceptions | Which are thrown, and the state after the throw |
| Preconditions | Reaction when violated |
| Postconditions | Always true after a successful call |
| Thread safety | The documented statement holds (see `concurrency-tests.md`) |
| Timing | Documented limit holds (logical time with a fake clock) |
| Lifetime | Object valid after move, copy, and destroy as documented |

## Limits of black-box testing

- It cannot show that all code paths run. Add white-box analysis.
- It can miss defects in hidden special cases of the code.
- Without a clear specification the tests are guesses. Tell the user if the requirement is unclear.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Expected value copied from the current output | Take it from the requirement or a hand calculation |
| Test uses internal state to check | Check the public output |
| Only happy paths | Add the negative cases |
| Unclear requirement | Ask the user. Do not guess. |

## Checklist

- [ ] Cases come from the requirement, with named techniques.
- [ ] Only the public interface is used.
- [ ] Positive, negative, and boundary cases exist.
- [ ] Expected values are independent from the code.
