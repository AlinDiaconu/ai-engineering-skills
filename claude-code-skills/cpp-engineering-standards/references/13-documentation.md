# 13 — Comments, Doxygen, ASD-STE100

## 1. What to comment

Good comments explain what the code cannot say:
- **Why** this design or algorithm (and why not the obvious alternative).
- Constraints: units, ranges, timing, hardware behavior, thread-safety, ordering requirements.
- Invariants and assumptions.
- Safety and security reasons; requirement references.
- Workarounds: what bug or limitation they handle, with a ticket link and a removal condition.
- Non-obvious performance choices, with the measurement.

Do not:
- Repeat the code (`++i; // increment i`).
- Keep commented-out code (version control keeps history).
- Write comments that are not true any more — update or delete comments with every change.
- Add comments only to reach a comment-density metric.
- Write journal comments (author/date/change history) in the code; use version control.
- Leave `TODO` without an owner or ticket.

Better than a comment: a good name, a named constant, a small function, a strong type, a
`static_assert`.

## 2. Style

- Language: English (most standards require it).
- `//` comments in C++ code; many standards forbid `/* */` (nesting and commented-out code risks).
  Doxygen: `///` and `///<`, or `/** */` if the project allows.
- One space after `//`. No line continuation (`\`) at the end of `//` comments.
- Comment above the code it explains; short trailing comments only for members and constants.
- Write short, active, direct sentences. Use ASD-STE100 when the project requires it.

## 3. Doxygen for public APIs

Document every public type, function, constant, and macro in headers. Use Doxygen commands, not HTML.

```cpp
/// @brief Computes the shipping cost of an order.
///
/// The cost depends on the total weight and on the destination zone. The function does not
/// change any shared state.
///
/// @param[in]  items      Items of the order. Must not be empty.
/// @param[in]  zone       Destination zone. Range [1, 8].
/// @param[out] breakdown  Cost parts (base, weight, surcharge). Written only on success.
/// @return Cost, or Error::EmptyOrder / Error::InvalidZone.
///
/// @pre    !items.empty()
/// @post   result >= kMinShippingCost
/// @note   Thread-safe: yes (no shared state).
/// @warning Not async-signal-safe: do not call from a signal handler.
/// @requirement REQ-SHIP-123
[[nodiscard]] std::expected<Money, Error>
computeShippingCost(std::span<const OrderItem> items, std::uint8_t zone, CostBreakdown& breakdown) noexcept;
```

Recommended tags: `@brief`, `@details`, `@param[in|out|in,out]`, `@tparam`, `@return`/`@retval`,
`@throws`, `@pre`, `@post`, `@invariant`, `@note`, `@warning`, `@see`, `@deprecated`,
custom aliases such as `@requirement` and `@safety` (define them in the Doxyfile `ALIASES`).

For each parameter state: direction, meaning, unit, valid range, null rule, ownership/lifetime.
For each class state: responsibility, invariants, thread-safety, lifecycle, ownership of members.
For each member: what it stores, unit, which mutex protects it (`///< Protected by m_mutex`).

## 4. ASD-STE100 (Simplified Technical English) for comments

Key writing rules (summary):
- Use approved words with their approved meaning; one word = one meaning (for example "start",
  not "initiate"/"commence"). Use technical names and verbs from the domain as needed.
- Sentences: max 20 words for procedures (instructions), max 25 words for descriptions.
- One instruction per sentence. Write instructions in the imperative ("Validate the input before ...").
- Use the active voice. Avoid the passive voice in procedures.
- Use simple verb tenses (present, simple past, simple future).
- Do not use "-ing" forms as nouns or modifiers when another construction is possible.
- Use articles ("the", "a") and do not omit words to make sentences short.
- Paragraphs: one topic each, max 6 sentences.
- Give warnings and cautions before the instruction they apply to; start with the command.
- Be specific: "Wait 10 ms" instead of "Wait a short time".

Examples:
| Avoid | STE |
|---|---|
| `// Initiating the filtering of the signal is performed here` | `// Filter the signal.` |
| `// This might potentially overflow` | `// This sum can overflow when count > 1000. Check count first.` |
| `// Checking ptr` | `// Make sure that the pointer is not null.` |

## 5. File and module documentation

- Each header: one `@file` block with a short description of the module purpose (if the project wants it). No license text duplication beyond the required header; no author/date history.
- Each component: a README with responsibilities, interfaces, threading model, error strategy,
  memory policy, configuration, and how to build and use it.
- Architecture decisions: short ADRs (context, decision, consequences) in the repository.
