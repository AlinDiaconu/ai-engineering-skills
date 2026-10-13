# Smoke tests (C++)

## What

A **small, fast** set of checks that show that the build is alive and the **core path** works. It does not look for deep defects.

## Why

- Stop a broken build early. Do not waste time on slow tests.
- Give a quick yes or no after each build and each deployment.

## When

- After each build in CI (right after the unit tests, or before the slow stages).
- After each deployment to a test bench, a simulator, or a device.
- After a flash of new software.
- Before you start long HIL or scenario runs.

## How

1. Pick the **core path**: start, initialize, run one cycle, produce the main output, stop.
2. Keep the set to **5 to 20 checks**. Total time: seconds to a few minutes.
3. Run the **real binary** (or a thin wrapper), not a unit-test binary.
4. Check **liveness and basic correctness**, not details.

## Smoke checks (examples)

| Check | Pass condition |
|---|---|
| Process starts and initializes | Exit code 0 on init. Ready state is reached within the start-up limit. |
| Configuration loads | A valid file is accepted. An invalid file is rejected with a defined error. |
| First cycle runs | One cycle completes. No error is set in a healthy start. |
| Main output is produced | The main output is sent in every cycle for N cycles. |
| Output is valid | Message has valid counter, checksum, timestamp, and no NaN. |
| Input handling | A valid input message is accepted. |
| Error path is alive | One simple injected fault gives the expected reaction. |
| Clean shutdown | Process stops in the time limit. No leaks reported. |
| Version and build info | The version string matches the build. |

## Example (CTest + GoogleTest)

```cpp
// Smoke: the program starts, runs 10 cycles, and keeps the output alive.
TEST(Smoke, StartsAndKeepsOutputAlive)
{
    FakeClock clock;
    RecordingSink sink;
    App app{MakeDefaultConfig(), clock, sink};

    ASSERT_TRUE(app.Start());
    for (int i = 0; i < 10; ++i) {
        clock.Advance(kCyclePeriod);
        app.Step();
    }

    EXPECT_EQ(sink.OutputCount(), 10U);
    EXPECT_TRUE(sink.AllValid());
}
```

```cmake
add_test(NAME app_smoke COMMAND app_smoke_test)
set_tests_properties(app_smoke PROPERTIES LABELS "smoke" TIMEOUT 60)
```

## Best practices

- **Fast and stable.** A flaky smoke test is worse than none. Fix at once.
- **Fail fast.** If smoke fails, stop the pipeline.
- **Use a timeout** on every smoke test.
- **Test the real build** (release config, real startup path) at least once per build.
- **Keep it small.** Add a check only if its failure means "the build is unusable."
- **Keep it independent** of external services that you do not control.
- **Run it after each deployment** to each target (host, SIL, HIL, device).
- **Report clearly.** One line per check, pass or fail, with the reason.

## Smoke vs sanity vs regression

| | Smoke | Sanity | Regression |
|---|---|---|---|
| Scope | Whole product, shallow | One changed area, narrow | Whole product, deep |
| Goal | Is the build usable? | Is this change reasonable? | Did we break something old? |
| Size | 5–20 checks | A few | Full suite |

## Pitfalls

| Pitfall | Fix |
|---|---|
| Smoke set grows to a full suite | Move deep checks to other suites |
| Depends on a shared test bench that is often down | Add a host-only smoke too |
| Passes with a dead system (checks too weak) | Check real outputs, not only "process running" |
| No timeout, hangs the pipeline | Set `TIMEOUT` |

## Checklist

- [ ] Under a few minutes in total.
- [ ] Covers start, one cycle, main output, shutdown.
- [ ] Has timeouts.
- [ ] Runs on every build and every deployment.
- [ ] Fails the pipeline early when it fails.
