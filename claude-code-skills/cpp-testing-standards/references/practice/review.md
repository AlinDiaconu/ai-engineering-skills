# Reviewing tests

Use before you report, and when you review tests that another person wrote. Rules for how a test looks are in `writing-tests.md`.

## Checklist

- [ ] The test traces to a requirement, or states why it has none.
- [ ] It has at least one assertion that can fail.
- [ ] It fails when the behavior is broken (proved, see below).
- [ ] It checks one behavior, through the public interface.
- [ ] Positive, negative, and boundary cases are present.
- [ ] Time, random, thread, file, and network dependencies are controlled.
- [ ] It passes shuffled and repeated. It does not depend on another test.
- [ ] Doubles check only what the requirement states.
- [ ] Floating-point checks use a tolerance with a reason.
- [ ] Failure messages tell a person what went wrong. The name shows unit, condition, result.
- [ ] Code that can fail has fault tests (`../types/fault-injection-tests.md`).
- [ ] Coverage meets the target, with no empty tests added for the number (`coverage-and-metrics.md`).
- [ ] Sanitizer runs are clean, or the gap is reported.

## Prove that a test can fail

Choose one and report which:

1. **Manual mutation:** in a scratch copy or with `git stash`, change one thing in production code (flip a comparison, remove a line). The test must fail. Undo the change. Never leave it in the code.
2. **Reasoning:** show which line change would fail the assertion, and why.
3. **Mutation tool:** run Mull or dextool on the module (`../types/mutation-tests.md`).

## Weak tests (reject these)

- Calls the function and checks nothing.
- Repeats the production code in the assertion.
- Checks that a mock was called, but not the effect.
- Always passes because of a wrong macro use (`EXPECT_TRUE(x == y || true)`).
- Uses `sleep` to wait for a thread.
- Passes on one machine only.
- Has `EXPECT_NO_THROW` as its only check.
- Reads the expected value from the code under test.
- Parameterized with one parameter set.
- `DISABLED_` with no issue link.

## Reviewing tests written by someone else

1. Read the requirement first. List what you would test.
2. Compare with the tests. Note the gaps.
3. Check each test against the checklist.
4. Give findings by severity: missing error-path case, test cannot fail, flaky risk, readability.
5. Do not rewrite the tests without asking. Give clear findings and a suggestion.
