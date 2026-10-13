# System and scenario tests (C++)

## What

- **System test:** the whole program runs as built, in a test environment, and is checked against the system requirements.
- **Scenario test:** a system test where the input is a **defined situation** (start state, inputs over time, injected faults) and the pass criteria are about the **outcome**.

## Why

- Shows that the whole chain of parts meets the requirements.
- Finds problems that appear only when all parts and the environment work together: start-up order, configuration, timing, resource limits.

## When

- Before release and on a schedule (nightly).
- After a change that touches the processing chain, timing, or interfaces.
- For each defect found in the field (add a scenario).
- To check the limits of the supported operating range.

## How

1. **Define the scenario** (a file in version control):
   - Start state and configuration.
   - Inputs over time (messages, requests, events, data files).
   - Faults to inject and when (see `fault-injection-tests.md`).
   - Environment limits (load, memory, slow peers).
2. **Define pass criteria**, measurable:
   - Final state or output is correct.
   - Time to reach it is within the limit.
   - Resource use (memory, CPU, handles) is within the limit.
   - Invariants hold during the run (for example no invalid output, no data loss, no error log above a level).
3. **Run** the real program (or the real library with a thin driver) in the test environment.
4. **Record** all inputs, outputs, the seed, the scenario version, and the program version.
5. **Evaluate** automatically against the criteria. Output pass or fail with the reason.

## Scenario catalog (guide)

| Dimension | Values to vary |
|---|---|
| Input rate and size | Low, typical, maximum, burst |
| Input quality | Valid, invalid, missing, late, duplicated, reordered |
| Configuration | Default, minimum, maximum, invalid |
| Start state | Clean start, restart after crash, upgrade from an old version |
| Fault type | Peer lost, slow, bad data, resource exhausted |
| Fault timing | At start-up, in the middle of an operation, during shutdown |
| Fault duration | Short glitch, long, permanent |
| Recovery | Fault clears, does not clear |
| Environment | Normal, low memory, slow disk, loaded CPU |

Use **pairwise** selection to limit the count (`../foundations/design-techniques.md`). Add high-risk full combinations by hand.

## Best practices

- **Repeatable.** Fixed seeds, fixed scenario files, fixed tool versions.
- **Parameterized scenarios.** One template, many parameter sets. Do not copy files.
- **Automated pass criteria.** No "look at the log" as the only check.
- **Justified tolerances** (timing, floating point).
- **Keep a regression set** of scenarios that found defects.
- **Keep a smoke subset** of a few scenarios that run fast after each build (`smoke-tests.md`).
- **Trace to requirements.** Each scenario names the requirement.
- **Cover the edges** of the supported range, not only the middle.
- **Vary one thing at a time**, then combine.
- **Log enough** to find the cause of a failure: inputs, internal state changes, decisions, outputs, timing.
- **Use a fake clock or a simulated clock** where the system allows it. Real time makes runs slow and flaky.
- **Keep the suite short enough to run often.** Run in parallel. Use a nightly full set.
- **State the limits** of the test environment. Do not claim that a simulated run replaces real-world validation.

## Test harness shape

```cpp
struct ScenarioResult {
    bool finalStateCorrect;
    bool invariantsHeld;
    double secondsToFinish;
    std::size_t peakMemoryBytes;
};

class ScenarioRunner {
public:
    ScenarioResult Run(const ScenarioSpec& spec, std::uint64_t seed);
};

class ScenarioTest : public ::testing::TestWithParam<ScenarioSpec> {
protected:
    ScenarioRunner runner_;
};

TEST_P(ScenarioTest, MeetsCriteria)   // Verifies: REQ-XXX
{
    const ScenarioResult r = runner_.Run(GetParam(), kSeed);

    EXPECT_TRUE(r.finalStateCorrect);
    EXPECT_TRUE(r.invariantsHeld);
    EXPECT_LE(r.secondsToFinish, kMaxSeconds);
    EXPECT_LE(r.peakMemoryBytes, kMaxMemory);
}
```

## Pitfalls

| Pitfall | Fix |
|---|---|
| Non-repeatable runs (seed lost) | Log and fix the seed |
| Pass criteria not measurable | Define numbers and tolerances |
| Only easy scenarios | Add edge, rare, and stress cases |
| Scenario files copied and edited | Use parameters |
| Test environment bug seen as a product defect (or the opposite) | Check the environment against known cases |
| Runs take too long, so they are skipped | Smoke subset, parallel runs, nightly full set |

## Checklist

- [ ] Scenario in version control, with parameters.
- [ ] Measurable pass criteria.
- [ ] Seed, scenario version, and program version logged.
- [ ] Edge cases covered.
- [ ] Regression and smoke subsets exist.
