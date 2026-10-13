# Property-based tests (C++)

## What

You state a **rule (property)** that must hold for **all valid inputs**. A tool generates many inputs, runs the test, and tries to break the rule. When it finds a failure, it **shrinks** the input to a small example.

## Why

- Finds cases that you did not think of.
- Gives broad input coverage with little code.
- Shrinking gives a small, clear failing case.
- Good for code with an invariant: parsers, encoders, math, containers, state machines.

## When

- Encode and decode pairs (serialization, message packing, CRC).
- Math and geometry with known laws (symmetry, monotonic, bounds).
- Containers and algorithms (sort, search, ring buffer, queue).
- State machines (random event sequences must keep the invariants).
- Replacing an implementation (compare to a simple model).

Do not use when you have no clear property, or when each test is very slow.

## How

1. **Find a property.** See the list below.
2. **Write a generator** for valid inputs (and a second one for edge inputs).
3. **Write the test:** for any generated input, the property holds.
4. Run **many cases** (100 to 1000 per run).
5. On failure: keep the **seed** and the **shrunk input**. Add the shrunk input as a normal unit test.

## Property patterns

| Pattern | Example |
|---|---|
| **Round trip** | `Decode(Encode(msg)) == msg` |
| **Idempotence** | `Normalize(Normalize(x)) == Normalize(x)` |
| **Invariant** | After any `push`/`pop` sequence: `size() <= capacity()` |
| **Model / oracle** | Ring buffer behaves like a `std::deque` with a limit |
| **Commutative** | `Merge(a, b) == Merge(b, a)` |
| **Monotonic** | A bigger input never gives a smaller output |
| **Bounds** | Output is always in `[min, max]` |
| **No crash / no UB** | Any valid input is handled. Run with sanitizers. |
| **Symmetry** | `Distance(a, b) == Distance(b, a)` |
| **Metamorphic** | Scale input by 2 → output scales by 2 |
| **Differential** | New code equals old code for all inputs |

## Tools for C++

| Tool | Notes |
|---|---|
| **RapidCheck** | QuickCheck style. Has a GoogleTest integration (`rapidcheck/gtest`). Generators, shrinking, state-machine testing. |
| **GoogleTest value-parameterized + `std::mt19937` with a fixed seed** | Simple home-made approach. No shrinking. |
| **FuzzTest (Google)** | Property tests that run as unit tests and as a fuzzer. Coverage guided. |
| **libFuzzer harness with structured input** | See `fuzz-tests.md` |

If the project already has one, use that one. Ask before you add a new library.

## Example (RapidCheck)

```cpp
#include <rapidcheck/gtest.h>

// Verifies: REQ-XXX. Round trip of a frame.
RC_GTEST_PROP(FrameCodec, RoundTrip, (const Frame& msg))
{
    const auto bytes = Encode(msg);
    const auto back  = Decode(bytes);

    RC_ASSERT(back.has_value());
    RC_ASSERT(*back == msg);
}
```

## Example (no library, fixed seed)

```cpp
TEST(RingBuffer, MatchesDequeModel)
{
    constexpr unsigned kSeed = 12345;
    std::mt19937 rng{kSeed};
    SCOPED_TRACE(::testing::Message() << "seed = " << kSeed);

    RingBuffer<int, 8> ring;
    std::deque<int> model;
    for (int step = 0; step < 10000; ++step) {
        if (rng() % 2 == 0) {
            const int v = static_cast<int>(rng());
            ring.Push(v);
            model.push_back(v);
            if (model.size() > 8) { model.pop_front(); }
        } else if (!model.empty()) {
            ASSERT_EQ(ring.Pop(), model.front()) << "step " << step;
            model.pop_front();
        }
    }
}
```

## Best practices

- **Print and fix the seed.** Make it settable. A failure must be reproducible.
- **Generators make valid data.** Constrain them to the interface contract. Add a separate test for invalid data.
- **Include edge values** in generators: zero, max, min, NaN (if valid), empty, one element.
- **Keep the property simple.** A complex property may have its own bugs. Prefer small properties.
- **Do not repeat the production code** in the property.
- **Limit run time.** Use a fixed number of cases in unit runs. Run more on a schedule.
- **Save shrunk failures** as normal regression tests.
- **Test state machines** with random event sequences. Check invariants after each event.
- **Use sanitizers** with property tests. They find memory errors in many paths.
- **Floating point:** compare with a tolerance. Constrain generators to avoid NaN and inf if the contract excludes them.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Flaky (seed not fixed or not printed) | Fix and print the seed |
| Generator makes invalid data, test fails for the wrong reason | Constrain it |
| Property too weak (always true) | Check with a deliberate bug |
| Test too slow | Fewer cases per run. Faster generators. |
| Property repeats the implementation | Use a simpler model or a round trip |

## Checklist

- [ ] The property is stated in one sentence.
- [ ] The seed is fixed and printed.
- [ ] Generators match the contract and include edge values.
- [ ] Failures become normal regression tests.
- [ ] Number of cases is bounded in CI.
