# Simulation and target tests: MIL, SIL, PIL, HIL (C++, embedded)

## What

| Name | Meaning | Where the code runs | Environment (peripherals, plant) |
|---|---|---|---|
| **MIL** (model-in-the-loop) | The model of the function | Modeling tool | Simulated |
| **SIL** (software-in-the-loop) | The real C++ source, built for the **host** | Host PC or CI | Simulated |
| **PIL** (processor-in-the-loop) | The real code, built for the **target CPU** | Target processor or emulator | Simulated |
| **HIL** (hardware-in-the-loop) | The real code on the **real device** | Target hardware | Simulated, through real I/O |

## Why

- Test behavior that is hard, slow, or risky to test with the real environment: faults, rare situations, limits.
- Test on the **target**: real timing, real I/O, compiler and platform effects.
- Repeatable runs.

## When

| Level | Use for |
|---|---|
| **SIL** | Scenarios and the full chain, at scale, in CI (nightly). Fault scenarios. Regression of field issues. |
| **PIL** | Compiler, CPU, and number-format differences (endianness, alignment, floating point). Execution time estimate. |
| **HIL** | Real timing, real buses and peripherals, boot and start-up, watchdog, power cycles, hardware faults. |

Use the **cheapest** level that can show the defect. Do the **many** tests in SIL. Do the **few, targeted** tests in HIL.

## How

### SIL

1. Build the production source for the host with **the same code and configuration**. Replace only the platform layer (OS, I/O, clock).
2. Connect it to the simulator through an **adapter** with the same interface as the real I/O layer.
3. Run **scenarios** (`system-tests.md`). Use **logical time** (simulator-controlled) for repeatability.
4. Evaluate pass criteria automatically. Log all signals.
5. Run in CI with parallel jobs.

### HIL

1. Flash a **release-like** build. Record the exact version and hash.
2. Connect the device to the bench (real buses and I/O, simulated environment).
3. Run **automated test scripts**: start-up, scenarios, fault injection at the interface, power cycle, timing checks.
4. Evaluate automatically. Save logs and traces.
5. Reset the bench to a **known state** between tests.

## Best practices

- **Same source, same tests.** Reuse test cases and scenario files across SIL and HIL. Only the adapter changes.
- **Keep a platform abstraction layer**, so the application code does not know if it runs in SIL or on the device.
- **Control time.** SIL: simulator time. HIL: real time with timestamps from the bench.
- **Automate everything.** No manual steps in a normal run.
- **Make runs repeatable.** Log seeds, scenario version, program version, bench configuration, and tool versions.
- **Reset to a known state** before each test (power cycle, clean memory, default configuration).
- **Inject faults at the real interface:** message loss, wrong CRC, delay, bus-off, sensor value freeze, voltage drop.
- **Check timing on HIL:** cycle time, reaction time, jitter, boot time. Report the maximum.
- **Use a smoke subset** of a few quick tests after each flash.
- **Compare SIL and HIL results** for the same scenarios. Differences show platform issues or simulator limits.
- **Keep the bench healthy.** A broken bench gives false failures. Run a bench self-check first.
- **Get approval** before you run tests on shared or real hardware.
- **State the limits** of the simulation.

## What HIL finds that SIL does not

- Timing and scheduling problems, overruns, jitter.
- Real I/O behavior: bus load, message timing, signal faults.
- Start-up, boot, reset, watchdog, and power behavior.
- Compiler, optimization, endianness, alignment, and floating-point differences.
- Stack and memory limits on the real device.
- Hardware diagnostics and error handling.

## Test script shape (idea)

```cpp
// HIL test, run by a bench framework through an adapter. Names are examples.
TEST_F(HilBenchTest, PeerLost_ReactsWithinLimit)   // Verifies: REQ-XXX
{
    bench_.PowerCycleDevice();
    ASSERT_TRUE(bench_.WaitForReady(kBootLimit));

    bench_.StartScenario("steady_load");
    bench_.WaitForSimTime(kSettleTime);

    bench_.InjectFault(Fault::kPeerHeartbeatLost);
    const auto t = bench_.MeasureTimeTo(Signal::kFallbackActive, kReactionLimit);

    ASSERT_TRUE(t.has_value());
    EXPECT_LE(*t, kMaxReactionTime);
}
```

## Pitfalls

| Pitfall | Fix |
|---|---|
| SIL code differs from the product code | Same source and configuration. Only the platform layer differs. |
| HIL tests are manual and slow | Automate. Use a smoke subset. |
| Non-repeatable runs | Log versions and seeds. Control time. |
| Bench left in a bad state | Reset before each test |
| Simulator limits ignored | State them in the report |
| Timing claims from SIL | Measure on HIL or target |
| Tests duplicated per level | Share test cases and scenario files |

## Checklist

- [ ] Same application source in SIL and HIL. Only the platform layer differs.
- [ ] Scenario files and test cases are shared.
- [ ] Automated pass criteria and full logs.
- [ ] Version, configuration, bench state, and seeds recorded.
- [ ] Timing measured on the target, with the maximum reported.
- [ ] Approval obtained for hardware use.
