# Benchmark and performance tests (C++)

## What

Tests that measure **speed, timing, and resource use**: execution time, throughput, latency, jitter, memory, CPU load.

## Why

- Check that timing requirements hold (cycle time, reaction time).
- Find performance regressions early.
- Compare design choices with data.

## When

- A requirement states a time or resource limit.
- A change touches a hot path or the cyclic path.
- You optimize. (Measure **before** and **after**.)
- Before release, on the target hardware.

## Types

| Type | Question | Tool |
|---|---|---|
| **Microbenchmark** | How fast is this function? | Google Benchmark |
| **Cycle-time test** | Does one cycle finish within the budget? | Target timer, tracing |
| **WCET / worst-case measurement** | What is the longest run time seen? | Instrumented run on target, with a WCET analysis tool when a proof is needed |
| **Jitter test** | How much does the period vary? | Timestamp each cycle start |
| **Load / stress test** | What happens at high input rate or under CPU load? | Load generator, `stress-ng` |
| **Soak test** | Is it stable for hours? Do memory and latency drift? | Long run with monitoring |
| **Memory footprint** | How much RAM, stack, and heap? | Sanitizers, `valgrind --tool=massif`, linker map, stack usage tools |
| **Startup time** | How long to reach ready? | Timer from process start |

## How (Google Benchmark)

```cpp
#include <benchmark/benchmark.h>

static void BM_ParseFrame(benchmark::State& state)
{
    const Buffer input = MakeFrameBuffer(static_cast<std::size_t>(state.range(0)));
    for (auto _ : state) {
        benchmark::DoNotOptimize(ParseFrame(input));
    }
    state.SetComplexityN(state.range(0));
}
BENCHMARK(BM_ParseFrame)->RangeMultiplier(2)->Range(8, 1024)->Complexity();

BENCHMARK_MAIN();
```

## Best practices

- **Define a target** first. "Fast" is not a target. Example: "ParseFrame takes under 200 µs at p99 on the target for a 256-byte frame."
- **Measure on a machine like production.** State the machine, compiler, flags, and load.
- **Use release builds** (`-O2` or the production flags). Never benchmark `-O0`.
- **Stop the optimizer** from removing work: `benchmark::DoNotOptimize`, `benchmark::ClobberMemory`.
- **Warm up** caches and branch predictors. Use enough iterations.
- **Report the spread**: min, median, p95, p99, max. For real-time, **the maximum matters most**, not the mean.
- **Reduce noise:** pin the CPU affinity, fix the CPU frequency, close other work, repeat runs (`--benchmark_repetitions=10`).
- **Change one thing at a time.** Compare to a **baseline** from the same machine.
- **Keep benchmarks separate from unit tests.** Do not put wall-time checks in unit tests. Machines differ.
- **Use logical time for logical requirements** (reaction in N cycles) in unit tests with a fake clock. Use real timing tests only for real timing.
- **Test input sizes** at small, typical, and worst case. Check how the time grows (complexity).
- **Check allocation** in the cyclic path (`../practice/determinism-and-realtime.md`).
- **Automate a small set in CI** and warn on large regressions (for example above 10%). Use a stable runner.
- **Store results** over time. Look at the trend.

## Real-time notes

- A mean time proves nothing for a deadline. Use the **observed maximum** and a margin. When a timing proof is needed, use a **WCET analysis** method that the project accepts.
- Test **under load** (other tasks, interrupts, cache pressure), because worst cases appear there.
- Measure **end to end** for reaction chains: detection + decision + output.
- Report the **margin** to the deadline.
- Timing results from the host do **not** prove timing on the target.

## Profiling (to find the cause)

| Tool | Use |
|---|---|
| `perf` | CPU sampling, cache misses |
| `valgrind --tool=callgrind` | Instruction counts, call graph (deterministic) |
| `valgrind --tool=massif` | Heap use over time |
| Flame graphs | See hot paths |
| Tracing (LTTng, Tracy, ETW, ARM trace) | Timeline of tasks and events |

Profile **after** a test shows a problem. Do not optimize without measurement.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Optimizer removes the code under test | `DoNotOptimize`, `ClobberMemory` |
| Debug build | Use the release build |
| Noisy machine | Pin CPU, repeat, fix frequency |
| Only the average is reported | Report percentiles and the maximum |
| Wall-time assert in a unit test | Use a benchmark suite. Use fake time for logic. |
| Benchmark with unrealistic data | Use realistic and worst-case data |
| Host result used for target claim | Measure on the target |

## Checklist

- [ ] A numeric target exists.
- [ ] Release build, realistic data, stated machine.
- [ ] Spread and maximum are reported, not only the mean.
- [ ] Compared to a baseline.
- [ ] Kept out of the unit test suite.
