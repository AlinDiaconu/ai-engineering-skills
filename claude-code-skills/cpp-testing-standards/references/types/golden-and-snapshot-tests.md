# Golden and snapshot tests (C++)

## What

You save a **known-good output** (the golden file or snapshot). The test runs the code and **compares** its output to the saved copy.

Also called: golden master, approval test, snapshot test, back-to-back with recorded data.

## Why

- Cheap check of **large or complex output** (serialized messages, reports, trajectories, logs, text, binary frames).
- Fast safety net for **legacy code** and for **refactoring**.
- Catches **unintended changes** in the output.

## When

- Output is large and hard to write as asserts: sampled data series, serialized frames, generated code, formatted text, config dumps.
- Characterization of legacy code.
- A wire format must stay stable (compatibility).
- Replacement of an algorithm (compare with a recorded reference).

Do **not** use when a few focused asserts can say the same thing. Do not use for output that changes often for good reasons.

## How

1. Produce the output in a **deterministic** way (fixed seed, fake clock, stable order).
2. **Review the output by hand** once. It is the oracle. Say where it came from.
3. Save it in the repository (small text, or binary with a readable companion).
4. In the test, produce the output again and **compare**.
5. When behavior changes on purpose: **update** the golden file, and **review the diff** like code.

```cpp
// Verifies: REQ-XXX (wire format stays stable)
TEST(FrameCodecGolden, KnownMessage_MatchesRecordedFrame)
{
    const auto frame = Encode(MakeKnownFrame());

    const auto golden = ReadBinaryFile(TestDataPath("frame_known_message.bin"));
    EXPECT_EQ(frame, golden);
}
```

For text: compare as text and print a diff on failure. For floats: parse and compare with a tolerance. Do not compare raw float text.

## Best practices

- **Deterministic output only.** Remove timestamps, addresses, thread ids, and unordered container order. Or normalize them.
- **Keep golden files small and readable.** Prefer text (JSON, CSV) to binary. If binary, add a text dump.
- **One golden file per behavior.** Name it by behavior.
- **Document the origin:** tool, version, date, input, and who reviewed it.
- **Review every update** like code. Never regenerate all files without reading the diff.
- **Make updates explicit:** a flag or environment variable (`UPDATE_GOLDEN=1`). Never update silently in CI.
- **Use tolerances for floating point.** Compare numbers, not strings.
- **Show a clear diff on failure.** Print the first difference, with the position.
- **Keep the path independent from the working directory.** Pass the test-data path through the build system.
- **Combine with real asserts** for the important fields. A golden file alone can hide which part matters.
- **Do not store sensitive data** in golden files.
- **Version the format.** If the format has a version field, keep one golden file per supported version.

## Approval-test libraries

| Tool | Note |
|---|---|
| **ApprovalTests.cpp** | Approval tests for C++ with GoogleTest, Catch2, doctest |
| **Hand-made helpers** | A small `ExpectMatchesGolden(actual, "name")` function |

Use what the project has.

## Golden data from recorded real input

- Use recorded real messages as **replay input** to test decoders and the full function chain.
- Record: source, date, software version, and what each file shows.
- Check that recorded data contains **no private data**.
- Re-record when interfaces change. Keep the old files as compatibility tests when old formats are still supported.

## Pitfalls

| Pitfall | Fix |
|---|---|
| Golden file locks in a bug | Review the first golden output against the requirement |
| Update without review | Review each diff. Update only on purpose. |
| Fails on another machine (paths, locale, float format, endianness) | Normalize. Compare numbers with tolerance. |
| File too big to review | Use a smaller input. Add a summary. |
| Flaky from unordered data or time | Sort. Fake the clock. |
| Used as a replacement for all asserts | Add explicit asserts for the key requirements |

## Checklist

- [ ] Output is deterministic and normalized.
- [ ] Golden file was reviewed against a requirement or a trusted source.
- [ ] Origin is documented.
- [ ] Update path is explicit and reviewed.
- [ ] Float values use a tolerance.
- [ ] Key requirements also have explicit asserts.
