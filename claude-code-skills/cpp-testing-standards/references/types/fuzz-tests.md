# Fuzz tests (C++)

## What

A tool sends **random or mutated input** to the code to find **crashes, hangs, and memory errors**. A coverage-guided fuzzer learns which inputs reach new code.

## Why

- Finds defects that humans do not think of: buffer overflows, integer overflows, infinite loops, bad state handling.
- Strong for **security** and **robustness** of code that reads external data.

## When

- Parsers and decoders (messages, files, configuration, maps, CAN or Ethernet frames).
- Deserialization code.
- Code with complex input handling or state machines driven by input.
- Any interface that receives data from outside the trust boundary.

Typical targets: message decoders, file and configuration loaders, network and diagnostic interfaces.

## How

1. **Write a fuzz target.** A small function that takes bytes and calls the code.
2. **Build** with a fuzzer and **sanitizers** (ASan + UBSan).
3. **Seed a corpus** with valid inputs (real recorded messages).
4. **Run** for a fixed time. Collect crashes.
5. **Minimize** each crash. Fix the bug. Add a **regression test** with the crash input.
6. Keep the **corpus** in version control. Re-run it in CI as a regression set.

### libFuzzer target

```cpp
// fuzz_frame_decoder.cpp
#include <cstddef>
#include <cstdint>

extern "C" int LLVMFuzzerTestOneInput(const std::uint8_t* data, std::size_t size)
{
    (void)DecodeFrame(data, size);   // must not crash, hang, or trigger UB
    return 0;
}
```

```cmake
add_executable(fuzz_frame_decoder fuzz_frame_decoder.cpp)
target_compile_options(fuzz_frame_decoder PRIVATE -fsanitize=fuzzer,address,undefined -g -O1)
target_link_options(fuzz_frame_decoder PRIVATE -fsanitize=fuzzer,address,undefined)
```

Run: `./fuzz_frame_decoder corpus/ -max_total_time=300`
Replay one input: `./fuzz_frame_decoder crash-<hash>`

### Structure-aware fuzzing

- Raw bytes are often rejected early (bad checksum, bad header). Use `FuzzedDataProvider` to build **valid-looking** structures.
- Or use a **custom mutator** that fixes the checksum after mutation.
- Or fuzz **below** the check (the inner parser) and test the check separately.

```cpp
#include <fuzzer/FuzzedDataProvider.h>

extern "C" int LLVMFuzzerTestOneInput(const std::uint8_t* data, std::size_t size)
{
    FuzzedDataProvider fdp{data, size};
    Message msg;
    msg.id      = fdp.ConsumeIntegral<std::uint16_t>();
    msg.counter = fdp.ConsumeIntegral<std::uint8_t>();
    msg.payload = fdp.ConsumeRemainingBytes<std::uint8_t>();
    FinalizeChecksum(msg);                 // keep the message "valid enough"
    (void)HandleMessage(msg);
    return 0;
}
```

## Tools

| Tool | Notes |
|---|---|
| **libFuzzer** (Clang) | In-process, coverage-guided. Default choice. |
| **AFL++** | Strong, many modes. Good for whole programs. |
| **Honggfuzz** | Alternative. |
| **FuzzTest (Google)** | Fuzz and unit-test in one. |
| **OSS-Fuzz style CI** | Long-running fuzzing for projects |

## Best practices

- **Always use sanitizers** (ASan, UBSan). Without them, many bugs stay hidden.
- **The target must be fast** and **deterministic**. No sleeps, no real I/O, no global state that grows.
- **Reset state** between calls, or create the object per call.
- **Limit resources** in the target: input size, loops, memory. Avoid "bugs" that are only huge inputs that are never legal (but do report real missing limits).
- **A good corpus** matters: use real valid messages. Keep it small (minimize with `-merge=1`).
- **Use a dictionary** (`-dict=`) of keywords or magic values for text formats.
- **Run short in CI** (for example 1 to 5 minutes, or only replay the corpus). **Run long** on a schedule.
- **Triage** each crash: reproduce, minimize, fix, add a regression test.
- **Track coverage** of the fuzzer to see which code it does not reach.
- **Do not fuzz** code that has side effects on real systems (files, network, hardware). Fake them.
- **Only fuzz systems you are allowed to test.**

## Fuzzing vs other tests

| | Unit / black-box | Property-based | Fuzz |
|---|---|---|---|
| Input | Chosen by a person | Generated from valid domain | Random and mutated, often invalid |
| Oracle | Expected value | Property | "No crash, no UB, no hang" (and optional properties) |
| Goal | Correct behavior | Broad correctness | Robustness and security |

Combine them. A fuzz target can also check properties (for example round trip) with `assert`.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Fuzzer stuck at a checksum | Structure-aware target or fix the checksum in a mutator |
| Slow target, low executions per second | Remove I/O and logging. Reduce setup. |
| Non-deterministic target | Remove time, random, global state |
| No sanitizers | Add ASan and UBSan |
| Crash found, no regression test | Add the crash file to a unit test |
| Out-of-memory from huge allocation inputs | Add a size limit in the code (a real fix) or in the harness |

## Checklist

- [ ] Fuzz target is small, fast, deterministic.
- [ ] Built with ASan and UBSan.
- [ ] Seed corpus has real valid inputs.
- [ ] Short run in CI, long run on a schedule.
- [ ] Every crash has a regression test.
