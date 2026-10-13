# Fault injection and error-path tests (C++)

## What

Tests that **cause a fault on purpose** and check how the code reacts. A fault is bad input, a failing dependency, a late or missing message, an exhausted resource, or a failed call.

## Why

- Error paths are the least tested code. They run rarely, but they decide what happens when things go wrong.
- A fault test shows three things: the fault is **detected**, the code **reacts** in the required time, and it ends in the **required state**.

## When

- Every function that can fail or that depends on something that can fail.
- Every timeout, retry, fallback, and recovery rule.
- Every parser and message handler.

## How

1. List the faults for the unit (checklists below).
2. Inject each fault through the **same interface** that the real fault uses: a fake that returns an error, a recorder that drops a message, an allocator that fails. Do not change production code to add the fault.
3. Check **detected**, **reacted to in time** (count cycles with a fake clock), and **end state**.
4. Check that the fault does not cause a **second, hidden** failure (stuck flag, lost message, leak, invalid output).
5. Check that a fault **below the detection limit** does not trigger a reaction (no false positive).
6. Check the **recovery**: a cleared fault gives the defined behavior (recover or stay latched until reset).

```cpp
// Verifies: REQ-XXX
TEST_F(ConnectionMonitorTest, HeartbeatMissingForTimeoutPlusOneTick_ReportsLost)
{
    feeder_.SendHeartbeat();

    feeder_.StopSending();                       // inject the fault
    clock_.Advance(kTimeout + kOneTick);
    monitor_.Step();

    EXPECT_TRUE(monitor_.IsPeerLost());          // detected
    EXPECT_EQ(sink_.LastEvent(), Event::kPeerLost);   // reacted
}
```

## Fault checklists

### Dependency and call faults

- [ ] A call returns an error code, throws, or returns an empty value.
- [ ] A call is slow (timeout) or never returns.
- [ ] A resource is exhausted: queue full, pool empty, allocation fails, handle limit.
- [ ] A callback throws.
- [ ] A dependency fails at the first call, in the middle, at the last call.

### Message and data faults (for each input message)

- [ ] Missing for 1 cycle, for N cycles, forever
- [ ] Delayed (age rises above the limit)
- [ ] Timestamp in the past, in the future, going backward
- [ ] Counter: skip, repeat, roll-over, reset
- [ ] Checksum or CRC wrong
- [ ] Length wrong, field out of range, enum value not defined
- [ ] Floating-point field is NaN or infinity
- [ ] Two sources disagree
- [ ] Valid flag false while the data looks valid
- [ ] Burst: many messages in one cycle

### Start-up, shutdown, and recovery

- [ ] Start with a fault present. Fault during start.
- [ ] Restart after a fault.
- [ ] Fault during shutdown.
- [ ] Fault that clears. Fault that must stay latched until reset.

## Rules

- Test **one fault at a time**. Then test chosen fault pairs that can happen together.
- Use a **fake clock** to test reaction times in logical cycles (`../practice/determinism-and-realtime.md`).
- Test the whole reaction chain: detection time + decision time + output time ≤ the budget.
- Use **exact exception types** and **error codes** in the checks.
- Test **exception safety**: after a thrown exception, the object is valid (basic guarantee) or unchanged (strong guarantee).
- Keep each fault test small. One fault, one expected reaction.

## Best practices

- Write a **fault table**: fault, injection point, expected detection, expected reaction, expected end state, time limit. One test per row (`TEST_P` when the body repeats).
- For state machines, test every invalid event in every state (`../foundations/design-techniques.md`).
- Run fault tests under **sanitizers**. Error paths often hide leaks and use-after-free.
- Add a fault test for **every defect** found in the field (`regression-and-characterization-tests.md`).

## Pitfalls

| Pitfall | Fix |
|---|---|
| Only the happy path is tested | Add the fault table |
| Fault injected by editing production code | Inject through the interface with a fake |
| Reaction time checked in wall time | Use a fake clock and cycle counts |
| Only the reaction is checked, not the end state | Check all three: detected, reacted, end state |
| Leak on the error path | Run under ASan. Test RAII on exception paths. |
| Fault pairs ignored | Test the pairs that can happen together |

## Checklist

- [ ] Fault table exists for the unit.
- [ ] Each fault: detected, reacted to in time, end state correct.
- [ ] No false positive below the detection limit.
- [ ] Recovery and latching behavior tested.
- [ ] Sanitizer runs are clean.
