# Integration and component tests (C++)

## Definitions

| Type | What |
|---|---|
| **Component test** | A group of classes behind **one interface** is tested together. Only the **outside** (I/O, other components, hardware) is replaced. |
| **Integration test** | **Real** components are connected and tested together. It checks the **interfaces** between them. |

## Why

- Unit tests do not find wrong assumptions between parts: units, ranges, timing, message layout, ownership, call order.
- Find configuration, start-up, and wiring defects.
- Check data flow from input to output.

## When

- Two or more parts exchange data or share a contract.
- A new interface is added or changed.
- Start-up, shutdown, and mode changes involve several parts.
- Message flow and timing across parts matter.

## How

### Component test

1. Build the component with its **real internals**.
2. Replace external dependencies with fakes (input source, output sink, clock).
3. Drive it through its public interface.
4. Check outputs and visible state.

```cpp
// Component: Parser + Validator + Dispatcher, with fake input and output.
TEST_F(PipelineComponentTest, InputLost_OutputsFallbackWithinBudget)   // Verifies: REQ-XXX
{
    input_.Push(MakeValidMessage());
    for (int i = 0; i < kWarmupCycles; ++i) { Cycle(); }

    input_.StopSending();
    for (int i = 0; i < kBudgetCycles; ++i) { Cycle(); }

    EXPECT_TRUE(sink_.Last().fallback.valid);
    EXPECT_EQ(sink_.Last().mode, Mode::kFallback);
}
```

### Integration test

1. List the **interfaces** to check (which part calls which).
2. Connect the **real** parts. Keep outside systems as fakes or simulators.
3. Test: **happy path**, **interface errors**, **timing**, **start-up and shutdown**.
4. Check both sides of each interface.

## Strategies

| Strategy | Meaning | Use when |
|---|---|---|
| **Bottom-up** | Integrate lower parts first, with drivers | Low-level parts are ready first |
| **Top-down** | Integrate top parts first, with stubs for lower parts | The top logic is ready first |
| **Sandwich** | Both | Large systems |
| **Big bang** | All at once | Avoid. Hard to find the cause. |
| **Incremental** | Add one part at a time | Default choice |

## What to check at an interface

- **Data layout:** field order, size, endianness, padding, alignment, version.
- **Units and frames:** m/s vs km/h, degrees vs radians, coordinate frames.
- **Ranges and validity:** what values and flags are legal.
- **Timing:** period, latency, age limit, timestamp base.
- **Ownership and lifetime:** who creates, who frees, how long a pointer or reference stays valid.
- **Error handling:** what the receiver does when the sender sends bad or no data.
- **Ordering:** what happens when messages arrive in a different order.
- **Start and stop:** order of initialization and shutdown. Start with a part missing.

## Best practices

- **Test the interface, not the internals** of each part.
- **Use real code** for the parts under test. Fake only the outside.
- **Keep tests deterministic.** Use a fake clock and a single-threaded step mode where possible.
- **Clean state per test.** Build the whole setup in the fixture. Do not share it between tests.
- **Keep few tests, each focused.** Do not copy unit tests to this level.
- **Use the same message types** as production. Do not invent test-only formats.
- **Check serialization round trips** and wire-format compatibility with recorded real messages.
- **Test configuration:** valid, invalid, missing, and default values.
- **Test start-up and shutdown order** and a restart after a fault.
- **Add timeouts** on all waits. A hung test must fail, not block the build.
- **Label** the tests (`integration`) so you can run them separately.
- **Use contract tests** to keep fakes aligned with real parts.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Tests depend on thread timing | Step mode, fake clock, latches |
| Shared global state between tests | Build fresh per test. Avoid singletons. |
| Too many parts, unclear failure cause | Smaller component, add logging at the interface |
| Test duplicates unit tests | Test only the interface behavior |
| Environment-specific (paths, ports) | Use temp dirs and port 0 |
| Slow tests slow down the commit stage | Move to the merge stage. Label them. |

## Checklist

- [ ] The interface and its contract are named.
- [ ] Real parts inside, fakes outside.
- [ ] Error, timing, and start-up cases included.
- [ ] Deterministic. Has timeouts. Labeled.
