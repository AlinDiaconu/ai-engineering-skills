# Regression and characterization tests (C++)

## Regression tests

### What

Tests that make sure a **fixed bug stays fixed** and that **old behavior still works** after a change.

### Why

- A bug that came back costs more than the first time.
- Each fixed bug shows a case that the old tests missed.

### When

- **Every bug fix.** No exception.
- Before each release (full suite).
- After refactoring and after compiler or library upgrades.

### How (bug-fix workflow)

1. **Reproduce** the bug.
2. Write the **smallest test** that fails because of the bug. Make sure it fails for the **right reason**.
3. Fix the code.
4. See the test **pass**. Run the whole suite.
5. Keep the test. Put the **issue ID** in a comment.
6. Ask: can the same type of bug happen somewhere else? Add tests for similar places.

```cpp
// Regression: ISSUE-1234. Heartbeat counter did not roll over at 255.
TEST(HeartbeatCounter, At255_NextValueIsZero)
{
    HeartbeatCounter counter{255};
    EXPECT_EQ(counter.Next(), 0U);
}
```

### Best practices

- Test at the **lowest level** that shows the bug.
- Keep the test **small and named by behavior**, with the issue ID as a comment.
- Do not delete regression tests, unless the feature is removed.
- Keep **regression scenarios** for system-level field issues.
- If a **crash input** comes from fuzzing, save it in the corpus and add a unit test with it.
- Run the regression suite **often**. A regression suite that runs once a quarter does not catch regressions.

## Characterization tests

### What

Tests that record what **existing code does now**, right or wrong. They are a safety net, not a statement that the behavior is correct.

### Why

- You must change old code that has no tests.
- You need to know that your change did not alter behavior.

### When

- Legacy code with no tests, before a change or a refactoring.
- Before replacing an algorithm or porting code.

### How

1. **Find a seam**: a place to separate the code under test (constructor parameter, interface, link-time swap).
2. **Break dependencies** with the smallest, safest change. Ask the user before you change production code.
3. **Call the code** with chosen inputs. Record the actual outputs.
4. **Assert the recorded outputs.** Name the tests by behavior, and mark them as characterization (`Characterization_` prefix or a comment).
5. Make your change. Keep the tests green.
6. Replace characterization tests with **real tests** (from requirements) as you learn the intended behavior.

```cpp
// Characterization: records current behavior. Not verified against a requirement.
TEST(LegacyFilterCharacterization, StepInput_ProducesRecordedResponse)
{
    LegacyFilter filter{kLegacyConfig};
    const std::array<double, 5> expected{0.0, 0.2, 0.36, 0.488, 0.5904};  // recorded

    for (std::size_t i = 0; i < expected.size(); ++i) {
        EXPECT_NEAR(filter.Step(1.0), expected[i], 1e-9) << "step " << i;
    }
}
```

### Best practices

- **Record, do not judge.** If the behavior looks wrong, **mark it** and tell the user. Do not "fix" it inside a characterization test.
- Choose inputs that **cover many paths**: use coverage to guide you.
- For large output, use **golden files** (`golden-and-snapshot-tests.md`).
- Record **before** you change the code.
- Keep the tests **until** real tests replace them.
- Make the recorded values **traceable**: tool, version, date, input.

## Back-to-back (differential) tests

- Compare **old** and **new** implementations with the same inputs.
- Use for a rewrite, an optimization, or a port.
- Combine with **property-based** generation for broad input coverage.
- State the accepted difference (for example floating-point tolerance).

## Pitfalls

| Pitfall | Fix |
|---|---|
| Fix with no test | Test first. Always. |
| Test passes before the fix | The test does not show the bug. Fix the test. |
| Characterization values treated as correct | Mark them. Review them against requirements. |
| Large golden file nobody can read | Keep small. Add a readable summary. |
| Regression tests deleted "to speed up" | Make them faster, not fewer |

## Checklist

- [ ] The test failed before the fix, for the right reason.
- [ ] Issue ID in a comment.
- [ ] Lowest possible level.
- [ ] Characterization tests are marked as such.
- [ ] Suspected bugs are reported to the user.
