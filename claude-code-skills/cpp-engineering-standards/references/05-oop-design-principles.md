# 05 — Object-oriented design principles (complete reference)

This file covers design principles. C++ class mechanics (constructors, special members,
inheritance rules, dependency injection, design patterns) are in `04-classes-oop.md`.

Principles are tools, not laws. They often pull in different directions (for example DRY vs low
coupling, Open/Closed vs YAGNI). Use them to explain a design decision, not to win an argument.
When two principles conflict, choose the design that is simpler to understand, test, and change
for the **expected** changes — not for imagined ones.

## 1. The four pillars

Overview:

| Pillar | Question it answers | Main C++ tools | Main risk if misused |
|---|---|---|---|
| **Encapsulation** | Who may see and change this state? | `private`, `class`, `const`, non-member functions, unnamed namespaces, pImpl, modules | Public data with an invariant; getter/setter for every member |
| **Abstraction** | What does the user need to know? | Functions, classes, interfaces, concepts, strong types, components | Speculative or leaky abstractions |
| **Inheritance** | Is this type substitutable for that type? | Public inheritance, pure virtual interfaces, NVI, CRTP | Inheritance for code reuse; deep, fragile hierarchies |
| **Polymorphism** | How do I select behavior by type? | Virtual functions, `std::variant`, templates/concepts, overloading, type erasure | Type `switch` spread in the code; virtual calls in hot loops |

Encapsulation and abstraction are related but not the same:
- **Abstraction** is a design decision: *which* operations and concepts the user sees.
- **Encapsulation** is a mechanism: *how* the rest is hidden and protected.
A good type uses both: a small abstract interface (abstraction) over protected state (encapsulation).

### 1.1 Encapsulation

**What it is.** Keep data and the operations that keep it valid together, and hide the
representation. Only the class can change its state, so only the class can break its invariant.

**Why use it.**
- The invariant is guaranteed in one place. Every object is valid at every public call boundary.
- You can change the representation (types, storage, caching, algorithm) without changing callers.
- Validation, locking, logging, and fault detection happen in one place — important for safety
  and thread safety.
- Callers depend on behavior, not on data layout: lower coupling, smaller recompilation.

**When to use it.**
- The type has an **invariant**: a relation between members (`min <= max`, `size <= capacity`),
  a valid range, a state machine, or a cached value that must match the source data.
- The type **owns a resource** (memory, handle, lock, hardware).
- The **representation is likely to change** (for example a fixed array today, a ring buffer later).
- The type is **shared between threads** — the mutex and the data must stay together.
- The type is at a **module, library, or ABI boundary**.

**When not to use it (or use less of it).**
- Passive data with no invariant: messages, DTOs, configuration records, results, points.
  Use a `struct` with public members. Private members + trivial getters/setters only add code.
- Do not hide things that the user really needs to control; that creates workarounds.

**How to apply it in C++ (from small to large scope).**

| Level | Mechanism |
|---|---|
| Member | `private` data; `const` member functions; no non-const references to internal data returned |
| Class | Public functions that express operations (`deposit()`), not field access (`setBalance()`); validation in constructors/factories |
| Free functions | Non-member non-friend functions for operations that need only the public API (they do not see private data, so they cannot break the invariant) |
| File / translation unit | Unnamed namespace for helpers and internal data in the `.cpp` file |
| Library / header | `detail` namespace; private headers; forward declarations; pImpl or opaque handles for ABI |
| Module (C++20) | Export only the interface; non-exported names are invisible |
| Component | Narrow public interface; internal types never appear in public signatures |

**Example — invariant protected by the class.**

```cpp
// BAD: passive struct, but it has an invariant. Any caller can set minC > maxC.
struct TemperatureRangeData
{
  float minC;
  float maxC;
};

// GOOD: the invariant (finite, min <= max) holds for every object.
class TemperatureRange
{
public:
  [[nodiscard]] static std::optional<TemperatureRange> create(float minC, float maxC) noexcept
  {
    if (!std::isfinite(minC) || !std::isfinite(maxC) || (minC > maxC))
    {
      return std::nullopt;
    }
    return TemperatureRange{minC, maxC};
  }

  [[nodiscard]] float clamp(float valueC) const noexcept { return std::clamp(valueC, m_minC, m_maxC); }
  [[nodiscard]] float minC() const noexcept { return m_minC; }
  [[nodiscard]] float maxC() const noexcept { return m_maxC; }

private:
  TemperatureRange(float minC, float maxC) noexcept : m_minC{minC}, m_maxC{maxC} {}
  float m_minC;
  float m_maxC;
};
```
`clamp()` is safe without any check because the invariant guarantees `min <= max`
(`std::clamp` has UB otherwise).

**Example — representation can change without touching callers.**

```cpp
class SampleHistory
{
public:
  void add(float sample) noexcept
  {
    m_samples[m_next] = sample;
    m_next = (m_next + 1U) % kCapacity;
    m_count = std::min(m_count + 1U, kCapacity);
  }
  [[nodiscard]] float average() const noexcept
  {
    if (m_count == 0U)
    {
      return 0.0F;
    }
    float sum{0.0F};
    for (std::size_t i = 0U; i < m_count; ++i)
    {
      sum += m_samples[i];
    }
    return sum / static_cast<float>(m_count);
  }
private:
  static constexpr std::size_t kCapacity{16U};
  std::array<float, kCapacity> m_samples{};   // could become a running sum later: callers do not change
  std::size_t m_next{0U};
  std::size_t m_count{0U};
};
```

**Getters and setters — rules.**
- A getter is fine when callers need the value (a query). Prefer names from the domain.
- A setter is a smell when it only assigns a field. Replace it with an operation that has a
  meaning and keeps the invariant (`range.narrow(newMax)`, `account.freeze()`).
- Never return a non-const reference or pointer to internal data from a class with an invariant.

**Costs.** Almost none at run time (inlined accessors). pImpl costs one allocation and one
indirection. Too much encapsulation (hiding what users need) causes workarounds.

**Self-test.** Can you change every private member type and name without changing one caller?
Can any caller create an object that breaks the invariant? If the answers are "yes" and "no",
encapsulation is correct.

### 1.2 Abstraction

**What it is.** Show only the essential properties and operations of a concept, in the
language of the problem domain, and hide how they work. The user thinks in terms of "save the
document", not "open a temporary file, write bytes, flush, and rename it".

**Why use it.**
- Reduces cognitive load: the reader understands the code at one level at a time.
- Separates **what** from **how**: you can change or replace the "how" (algorithm, hardware,
  middleware) without changing the users.
- Creates test seams: an abstract interface can be replaced by a fake in tests.
- Isolates change and risk: hardware, third-party libraries, and operating systems stay behind
  a stable interface (GRASP Protected Variations).

**Forms of abstraction in C++.**

| Form | Example | Cost |
|---|---|---|
| Procedural (function) | `computeShippingCost(weight, distance)` | None |
| Data abstraction (class/ADT) | `TemperatureRange`, `Invoice` | None |
| Strong types | `Meters`, `std::chrono::milliseconds` | None |
| Interface (abstract class) | `IPaymentGateway` | Virtual call |
| Concept (compile-time interface) | `template <Storage S>` | Code size |
| Type erasure | `std::function`, custom wrappers | Indirect call, possible heap |
| Component / platform layer / module | `net::Socket`, `storage` library | Build structure |

**When to use it.**
- The concept exists in the domain or the requirements (order, invoice, payment, account).
- There are **already** two or more implementations, or a second one is **certain**
  (real device + simulator, two vendors, test fake).
- At boundaries: hardware, OS, middleware, third-party libraries, other teams' components.
- When a block of code has a meaning that a name can express (extract a function).
- To hide a complex algorithm behind a simple, stable operation.

**When not to use it.**
- Only one implementation, no test need, no boundary: a concrete class is simpler. Interfaces
  "for the future" are speculative generality (YAGNI).
- Wrappers that add no meaning (`MyVector` around `std::vector` with the same functions).
- When the abstraction would hide something the user must control (timing, memory, errors) —
  that creates a leaky abstraction.
- **Rule of three**: write it once; tolerate a second copy; abstract on the third occurrence,
  when you know what really varies.

**How to design a good abstraction.**
1. Start from the **user's** needs, not from the implementation. Write the calling code first.
2. Use domain names. One concept per type. One level of abstraction per interface.
3. Keep it **minimal**: the smallest set of operations that is complete for the users.
4. Make it **hard to misuse**: strong types, `[[nodiscard]]`, no ordering requirements between calls.
5. State the **contract**: preconditions, postconditions, errors, units, timing, thread safety.
6. Do not leak implementation types (no third-party or middleware types in the signature).
7. **Single Level of Abstraction Principle (SLAP)**: inside one function, all statements are at
   the same level of detail.

```cpp
// BAD: mixes high-level steps with byte-level details.
void ReportJob::run()
{
  const std::vector<std::uint8_t> raw = m_source.readAll();
  const auto recordCount = static_cast<std::uint32_t>((raw[0] << 8U) | raw[1]);
  // ... 80 more lines of mixed levels ...
}

// GOOD: one level per function; details move down into named functions.
void ReportJob::run()
{
  const Records records = readRecords();
  const Report report = buildReport(records);
  publish(report);
}
```

**Example — abstraction at a system boundary.**

```cpp
/// Abstraction of a payment provider. Amounts in cents. Thread safety: one caller at a time.
class IPaymentGateway
{
public:
  virtual ~IPaymentGateway() = default;
  /// @return Status::Ok, or Status::Rejected when the provider declines the payment.
  [[nodiscard]] virtual Status charge(std::string_view accountId, std::int64_t amountCents) = 0;
  [[nodiscard]] virtual bool isAvailable() const = 0;
protected:
  IPaymentGateway() = default;
  IPaymentGateway(const IPaymentGateway&) = default;
  IPaymentGateway& operator=(const IPaymentGateway&) = default;
};
// Implementations: CardPaymentGateway, SandboxPaymentGateway, FakePaymentGateway (tests).
```

**Signs of a leaky or wrong abstraction.**
- Users must know the implementation to use it correctly ("call `flush()` before `read()`
  because of internal buffering").
- Implementation types appear in the interface.
- Every new implementation needs changes to the interface.
- Most functions of the interface are not supported by some implementations (violates ISP/LSP).
- Users bypass it to get performance or control.

**Costs.** Virtual calls and indirection; more types to navigate; wrong abstractions are
expensive to remove. A wrong abstraction costs more than duplicated code.

### 1.3 Inheritance

**What it is.** A class derives from a base class and gets its interface (and maybe its
implementation). There are two very different uses:
- **Interface inheritance (subtyping)**: the derived class *is a* base and can replace it
  everywhere (Liskov). This is the main correct use.
- **Implementation inheritance**: the derived class reuses base code. This is the most common
  misuse — composition does it better.

**Why use it.**
- **Substitutability**: code written against the base works with every present and future
  derived class (polymorphism, Open/Closed).
- A common **contract** for a family of types (interface).
- A controlled **extension point** in a framework (NVI / template method).

**When to use it.**
- A real "is-a" relation where every derived class keeps the base contract (LSP holds).
- Implementing an interface (abstract class with only pure virtual functions).
- An extension point designed for inheritance (protected/private virtual hooks, NVI).
- Static reuse with CRTP / mixins where zero overhead is needed.
- Ask these questions; inheritance is right only if all answers are "yes":
  1. Is every `Derived` really a `Base` in every context where `Base` is used?
  2. Does `Derived` keep every promise of `Base` (preconditions, postconditions, invariants, errors)?
  3. Will callers use `Derived` through `Base&`/`Base*`?
  4. Was `Base` designed for inheritance (virtual or protected destructor, documented hooks)?

**When not to use it.**
- Only to reuse code → composition (member) or free functions.
- "Has-a" relations (a `Car` has an `Engine`, it is not one).
- To change the behavior of a concrete class not designed for it (fragile base class problem).
- From standard containers or other classes without a virtual destructor (`std::vector`,
  `std::string`, `std::map`).
- To share data between classes.
- When the hierarchy would be deeper than 2-3 levels (excluding interfaces).
- When the set of types is closed and small → `std::variant` is often simpler.

**How to apply it in C++.**

| Form | Meaning | Use |
|---|---|---|
| `public` inheritance | "is-a" (subtyping) | Interfaces and real subtypes only |
| `private` inheritance | "implemented in terms of" | Rare; prefer a member. Old use (empty base optimization) is replaced by `[[no_unique_address]]` (C++20) |
| `protected` inheritance | Rarely meaningful | Avoid |
| `virtual` inheritance | Shared base in a diamond | Only for interface diamonds |
| Multiple inheritance | Several bases | Fine for several interfaces; avoid for several implementations |

C++ rules for writing a hierarchy (destructors, `override`/`final`, slicing, NVI, `clone`):
see `04-classes-oop.md` section 8.

**Fragile base class problem.** A change inside a base class (for example a base function now
calls another virtual function) silently breaks derived classes. Protect against it: make
classes `final` by default; design explicit hooks (NVI); keep base classes small and stable;
prefer interfaces without implementation.

**Composition over inheritance.** Use inheritance only for substitutable "is-a" relations with a
stable interface. For code reuse and "has-a", use composition (CG C.120, C.129). Inheritance
creates the strongest coupling in C++.

Signs that inheritance is wrong: the derived class overrides a function to do nothing or to
throw; the derived class uses only part of the base; you inherit from a concrete class to reuse
its helpers; the hierarchy has more than 2-3 levels.

**Example — inheritance for reuse vs composition.**

```cpp
// BAD: inherits to reuse code; the Logger and MathUtils interfaces leak into every user.
class ReportGeneratorBad : public Logger, public MathUtils {};

// GOOD: composition; dependencies explicit, replaceable, testable.
class ReportGenerator
{
public:
  ReportGenerator(ILogger& logger, const ReportOptions& options) noexcept
    : m_logger{logger}, m_options{options} {}
private:
  ILogger& m_logger;               // used, not owned
  ReportOptions m_options;         // owned value
  TableFormatter m_formatter;      // owned part
};
```

**Example — inheriting a container vs wrapping it.**

```cpp
// BAD: inherits a container to reuse its functions.
// - std::vector has no virtual destructor (deleting through std::vector* is UB).
// - Every vector operation is public, so callers can break the invariant (sorted by time).
class EventLogBad : public std::vector<LogEntry> {};

// GOOD: composition; only meaningful operations are public; the invariant is protected.
class EventLog
{
public:
  [[nodiscard]] bool append(const LogEntry& entry)        // keeps the entries sorted by time
  {
    if (!m_entries.empty() && (entry.timestampMs < m_entries.back().timestampMs))
    {
      return false;
    }
    m_entries.push_back(entry);
    return true;
  }
  [[nodiscard]] std::span<const LogEntry> entries() const noexcept { return m_entries; }
private:
  std::vector<LogEntry> m_entries;
};
```

**Performance notes.** Each polymorphic class has a vtable and each object a vptr (one pointer
of extra size). Virtual calls block inlining and make the call graph harder to follow. Keep
hierarchies shallow and prefer composition and templates inside performance-critical code.

### 1.4 Polymorphism

**What it is.** One interface, several behaviors: the code that uses an operation does not need
to know the exact type. C++ supports several kinds:

| Kind | C++ mechanism | Binding time |
|---|---|---|
| Subtype (dynamic) | Virtual functions through `Base&`/`Base*` | Run time, open set of types |
| Sum type (closed) | `std::variant` + `std::visit` | Run time, closed set of types |
| Parametric (static) | Templates, concepts, CRTP | Compile time |
| Ad hoc | Function and operator overloading | Compile time |
| Type erasure | `std::function`, `std::any`, custom wrappers | Run time, value semantics |

**Why use it.**
- Removes repeated `switch (type)` / `if (dynamic_cast...)` chains: adding a type does not
  change every user (Open/Closed).
- Writes generic algorithms once for many types.
- Enables substitution in tests (fakes, mocks) and in deployments (production vs simulation).

**When to use which form.**

| Situation | Best choice | Reason |
|---|---|---|
| New types are added by other modules or teams; choice at run time | Virtual functions | Open set; stable interface; separate compilation |
| Fixed, small set of types known in one place; value semantics wanted | `std::variant` | No heap, exhaustive checking by the compiler, data per alternative |
| Type known at compile time; performance-critical | Templates + concepts | Zero overhead, full inlining |
| Same operation for different argument types | Overloading | Simple and clear |
| Store "anything callable/printable" by value | Type erasure | Value semantics without a common base class |
| Behavior depends on **data values**, not on types | Data/table (lookup, configuration) | No polymorphism needed |

**When not to use it.**
- Only one implementation exists and no test seam is needed.
- To select behavior by a value that is just data (use a table or a parameter).
- Virtual calls per element in hot loops over millions of objects: group objects by type and
  process each group in a non-virtual loop, or use `std::variant` / templates.
- Overloads with different meanings (`process(int)` does something different from `process(float)`).

**The same problem in three forms.**

```cpp
// 1. Subtype polymorphism: open set, run-time choice.
namespace dynamic_form
{
  class Shape
  {
  public:
    virtual ~Shape() = default;
    [[nodiscard]] virtual float area() const = 0;
  protected:
    Shape() = default;
    Shape(const Shape&) = default;
    Shape& operator=(const Shape&) = default;
  };
  class Circle final : public Shape
  {
  public:
    explicit Circle(float r) noexcept : m_r{r} {}
    [[nodiscard]] float area() const override { return std::numbers::pi_v<float> * m_r * m_r; }
  private:
    float m_r;
  };
  class Square final : public Shape
  {
  public:
    explicit Square(float side) noexcept : m_side{side} {}
    [[nodiscard]] float area() const override { return m_side * m_side; }
  private:
    float m_side;
  };
  [[nodiscard]] inline float totalArea(std::span<const Shape* const> shapes)
  {
    float sum{0.0F};
    for (const Shape* shape : shapes) { sum += shape->area(); }
    return sum;
  }
}

// 2. Sum type: closed set, value semantics, no heap, compiler checks every alternative.
namespace variant_form
{
  struct Circle { float r; };
  struct Square { float side; };
  using Shape = std::variant<Circle, Square>;

  [[nodiscard]] inline float area(const Shape& shape)
  {
    return std::visit(Overloaded{   // Overloaded: see 12-standard-library.md section 5
                        [](const Circle& c) { return std::numbers::pi_v<float> * c.r * c.r; },
                        [](const Square& s) { return s.side * s.side; }},
                      shape);
  }
  [[nodiscard]] inline float totalArea(std::span<const Shape> shapes)
  {
    float sum{0.0F};
    for (const Shape& shape : shapes) { sum += area(shape); }
    return sum;
  }
}

// 3. Parametric polymorphism: compile-time, zero overhead, one type per container.
namespace static_form
{
  template <typename T>
  concept HasArea = requires(const T& t) { { t.area() } -> std::convertible_to<float>; };

  template <HasArea T>
  [[nodiscard]] float totalArea(std::span<const T> shapes)
  {
    float sum{0.0F};
    for (const T& shape : shapes) { sum += shape.area(); }
    return sum;
  }
}
```

**Cost comparison.**

| Form | Run-time cost | Memory | Adding a new type | Adding a new operation |
|---|---|---|---|---|
| Virtual | Indirect call, no inlining | vptr per object, vtable per class; objects usually by pointer | Easy (new class) | Hard (change the interface and all classes) |
| `std::variant` | Index check + jump, inlinable | Size of largest alternative + tag; by value | Change the variant; the compiler finds every `visit` | Easy (new function with `visit`) |
| Templates | None | None; more code (one copy per type) | Easy | Easy |
| Type erasure | Indirect call | Small buffer or heap | Easy | Hard |

This is the "expression problem": virtual functions make new *types* easy; `std::variant`
makes new *operations* easy. Choose by the change you expect more often.

**Rules for virtual functions.** Exactly one of `virtual`/`override`/`final`; no default
arguments; no virtual calls in constructors/destructors; virtual destructor or protected
destructor in the base; prefer NVI for contracts; avoid `dynamic_cast` (if needed, the design
probably needs another virtual function or a `std::variant`).

### 1.5 How the pillars work together

1. **Abstraction** chooses the concept and its minimal interface (`IPaymentGateway`).
2. **Encapsulation** protects the state and invariant of each implementation (`CardPaymentGateway` keeps its protocol details private).
3. **Inheritance** (interface only) makes each implementation substitutable for the abstraction.
4. **Polymorphism** lets the user code (`CheckoutService`) call the operation without knowing the implementation.

Decision flow for a new type:
1. Is it passive data without an invariant? → `struct` with public members. Stop.
2. Does it have an invariant or own a resource? → `class` with private data (encapsulation).
3. Do several implementations exist, or is a test seam needed at a boundary? → define an abstraction.
4. Is the set of implementations open and chosen at run time? → interface + virtual functions.
   Closed set? → `std::variant`. Known at compile time? → template + concept.
5. Do you want to reuse code? → composition, not inheritance.

### 1.6 Pillars checklist

- [ ] Every type with an invariant hides its data; every passive data type is a plain `struct`.
- [ ] No setters that only assign fields; operations have domain meaning.
- [ ] Abstractions exist only for real variation, boundaries, or test seams; they are minimal and do not leak.
- [ ] Each function stays at one level of abstraction.
- [ ] Public inheritance is used only for substitutable subtypes; no inheritance from classes without a virtual destructor; hierarchies are shallow.
- [ ] Classes are `final` unless designed for inheritance.
- [ ] The polymorphism form fits the situation (virtual / variant / template / overload / type erasure).
- [ ] No type `switch` or `dynamic_cast` chains repeated across the code.
- [ ] No virtual dispatch per element in hot loops without measurement.

## 2. SOLID (detailed)

| Principle | Meaning | C++ practice |
|---|---|---|
| **S** — Single Responsibility | A class has one reason to change | Split I/O, parsing, logic, and presentation into separate classes; small headers |
| **O** — Open/Closed | Open for extension, closed for modification | Add new behavior with new types (interfaces, `std::variant` alternatives, template policies), not with new `if`/`switch` branches in old code |
| **L** — Liskov Substitution | A derived class works everywhere the base works | Do not strengthen preconditions or weaken postconditions in overrides; no new error kinds; no "not supported" overrides |
| **I** — Interface Segregation | Clients do not depend on functions they do not use | Several small interfaces (`IReadable`, `IWritable`) instead of one large one; concepts with minimal requirements |
| **D** — Dependency Inversion | High-level code depends on abstractions, not on details | Inject interfaces or template parameters; the concrete type is chosen at the composition root (`main`, factory) |

### S — Single Responsibility Principle (SRP)
"A class has one reason to change" — one actor or one concern drives its changes.
- Signs of violation: the class name contains "And", "Manager", "Helper", "Util", "Processor"
  without a clear domain meaning; members fall into groups that never interact; changes for
  unrelated features touch the same file.
- Typical splits: data acquisition / validation / computation / output; protocol parsing /
  business logic; configuration loading / configuration use.
- Do not over-split: a class with one function and no state is often a free function.

### O — Open/Closed Principle (OCP)
"Open for extension, closed for modification."
- Add behavior by adding code (a new type, a new `std::variant` alternative, a new policy), not
  by editing tested code.
- Apply OCP at the points where change is **likely** (new file formats, new payment providers). Do
  not make everything extensible (YAGNI).
- Note: with `std::variant`, adding a type modifies the variant, but the compiler shows every
  `std::visit` that must handle it — often safer than an open hierarchy.

### L — Liskov Substitution Principle (LSP)
"A subtype must be usable everywhere its base type is used, without surprises."

Rules for an override:
- Preconditions: the same or weaker (accept at least everything the base accepts).
- Postconditions: the same or stronger (deliver at least what the base promises).
- Invariants of the base: kept.
- Errors: no new error kinds that callers of the base cannot expect.
- History: no state changes that the base does not allow.

Violation signs: overrides that do nothing, throw "not supported", or check the dynamic type;
callers that use `dynamic_cast` to handle special subtypes; the classic Rectangle/Square case:

```cpp
class Rectangle
{
public:
  virtual ~Rectangle() = default;
  virtual void setWidth(float w) { m_w = w; }
  virtual void setHeight(float h) { m_h = h; }
  [[nodiscard]] float area() const { return m_w * m_h; }
protected:
  float m_w{0.0F};
  float m_h{0.0F};
};

// BAD: a Square that changes both sides in setWidth breaks code that works for Rectangle
// (after setWidth(2) and setHeight(3), callers expect area() == 6).
// Fix: no inheritance; make Rectangle and Square separate immutable value types,
// or give them a common read-only interface (IShape::area()).
```

### I — Interface Segregation Principle (ISP)
"Clients must not depend on functions they do not use."
- Several small role interfaces instead of one large one: `IReadable`, `IWritable`,
  `IConfigurable` instead of `IDevice` with 20 functions.
- One class can implement several role interfaces. Each client gets only the role it needs.
- With templates: write concepts with minimal requirements.
- Benefit: smaller recompilation, simpler fakes, clearer dependencies.

### D — Dependency Inversion Principle (DIP)
"High-level modules do not depend on low-level modules. Both depend on abstractions.
Abstractions do not depend on details."
- The interface belongs to the **client** (the high-level module), not to the implementation.
  Example: the billing module defines `IPriceSource`; the market-data adapter implements it.
- Concrete types are selected only in the composition root.
- How to inject dependencies in C++: `04-classes-oop.md` section 9.

**Example — interface segregation and dependency inversion together.**

```cpp
class IPriceSource               // small interface owned by the client: only what it needs
{
public:
  virtual ~IPriceSource() = default;
  [[nodiscard]] virtual std::optional<double> currentPrice(std::string_view symbol) const = 0;
protected:
  IPriceSource() = default;
  IPriceSource(const IPriceSource&) = default;
  IPriceSource& operator=(const IPriceSource&) = default;
};

class QuoteService               // high-level policy depends only on the abstraction
{
public:
  explicit QuoteService(const IPriceSource& prices) noexcept : m_prices{prices} {}
  [[nodiscard]] std::optional<double> quote(std::string_view symbol, double quantity) const
  {
    const auto price = m_prices.currentPrice(symbol);
    return price ? std::optional<double>{*price * quantity} : std::nullopt;
  }
private:
  const IPriceSource& m_prices;  // non-owning; the composition root owns the source
};
```
The protected copy operations in the interface prevent slicing through the base, while derived
classes stay copyable if they want.

## 3. GRASP (General Responsibility Assignment Software Patterns)

GRASP answers "which class should do this?".

| Principle | Rule | Example |
|---|---|---|
| **Information Expert** | Give a responsibility to the class that has the information for it | `Order::total()`, not `OrderUtils::computeTotal(order)` |
| **Creator** | Class B creates A if B contains, aggregates, closely uses, or has the data to initialize A | `Order` creates its `OrderLine`s |
| **Controller** | A non-UI object receives and coordinates a system event | `CheckoutController` handles the "place order" event, not the button callback |
| **Low Coupling** | Assign responsibilities so that dependencies stay few and weak | Depend on an interface or a value type, not on five concrete classes |
| **High Cohesion** | Keep each class focused; its members work together | Split a class whose functions use disjoint member groups |
| **Polymorphism** | When behavior varies by type, use polymorphism, not type checks | Virtual `Shape::area()` instead of `switch (shape.kind)` in ten places |
| **Pure Fabrication** | Create a class that is not a domain concept to keep cohesion and low coupling | `OrderRepository`, `MessageSerializer` |
| **Indirection** | Put an intermediate object between two parts to decouple them | Adapter between the domain and the message bus |
| **Protected Variations** | Wrap points of likely change or instability behind a stable interface | Platform interface over OS or hardware APIs; adapter over a third-party library |

## 4. General design principles

| Principle | Rule | C++ practice | Limit |
|---|---|---|---|
| **Encapsulate what varies** | Isolate the parts likely to change | Strategy/policy, configuration, adapters | Only for real expected variation |
| **Program to an interface, not an implementation** | Depend on abstractions | References to interfaces, concepts, `std::span` instead of `std::vector&` | Value types and stable types need no interface |
| **Composition over inheritance** | Build behavior from parts | Members, injected dependencies | Inheritance is still right for real "is-a" |
| **DRY** (Don't Repeat Yourself) | One authoritative place for each piece of **knowledge** | Shared functions, constants, generated code from one source | Similar-looking code with different reasons to change is not duplication; wrong abstractions cost more than duplication |
| **SSOT** (Single Source of Truth) | Each data item has one owner | One config struct, one state owner | — |
| **KISS** (Keep It Simple) | Prefer the simplest design that works | Plain functions and structs before hierarchies and templates | Simple ≠ easy-to-write; simple = easy to understand |
| **YAGNI** (You Aren't Gonna Need It) | Do not build features or extension points before they are needed | No speculative interfaces, parameters, or generality | Safety mechanisms and testability are needed now |
| **Separation of Concerns** | Separate different concerns into different modules | I/O vs logic vs presentation; policy vs mechanism | — |
| **Command-Query Separation (CQS)** | A function either changes state (command) or returns information (query), not both | `const` queries; commands return `void` or a status | Accepted exceptions: `pop()`-style operations, atomic `exchange`, `std::expected` status of a command |
| **Tell, Don't Ask** | Tell objects what to do; do not pull data out to decide outside | Behavior next to the data | Value types and DTOs are meant to be read |
| **Law of Demeter** | Talk only to direct collaborators | No `a.b().c().d()` chains through objects | Fluent builders and value accessors are fine |
| **Fail Fast** | Detect invalid states at the earliest point | Validate in constructors, assertions, `static_assert`, strong types | In SAFETY-CRITICAL: fail to a **safe state**, not to a crash |
| **Principle of Least Astonishment** | Code behaves as its name and type promise | Operators with natural meaning, `const` that is really const, no hidden side effects in getters | — |
| **Principle of Least Privilege** | Give each part the minimum access it needs | `private` by default, `const&` parameters, narrow interfaces, read-only views | — |
| **Inversion of Control / Hollywood Principle** | "Do not call us, we call you": the framework calls your code | Callbacks, event loops, template method/NVI | Keep control flow traceable in SAFETY-CRITICAL code |
| **Design by Contract** | Explicit preconditions, postconditions, invariants | See `03-functions-interfaces.md` section 8 | — |
| **Make illegal states unrepresentable** | Use types so that invalid combinations cannot be built | `std::variant` states, strong types, private constructors + factories, `enum class` | — |
| **Immutability by default** | Objects do not change after construction unless needed | `const` members via no setters, "with" functions, value types | Large objects updated often may need mutation |
| **Orthogonality** | Changing one thing does not affect unrelated things | Independent modules, no hidden shared state | — |
| **Boy Scout Rule** | Leave touched code a little cleaner | Small, safe improvements in the lines you change | Do not mix large refactors into feature or safety changes |

**Tell, Don't Ask — example.**
```cpp
// Ask: the decision logic leaks out of the class.
if (inventory.count(itemId) < inventory.reorderLevel(itemId)) { inventory.markForReorder(itemId); }

// Tell: the logic stays with the data.
inventory.checkReorder(itemId);
```

**Law of Demeter — example.** `order.customer().address().country().taxRate()` couples the caller
to four classes. Add `order.taxRate()`, or pass the tax rate in. Fluent builders and value-type
accessors such as `point.x` are not violations.

## 5. Coupling and cohesion

### Coupling types (from worst to best)
| Type | Description | Example |
|---|---|---|
| Content | A module changes or depends on the internals of another | `#define private public`, pointer into another class's private data |
| Common (global) | Modules share global mutable data | Global state structs, singletons with state |
| External | Modules share an external format or device protocol directly | Two modules parse the same raw file or message layout themselves |
| Control | One module passes a flag that tells another what to do | `process(data, /*mode=*/3)` |
| Stamp | A module receives a whole structure but uses only part of it | `computeShipping(const Order&)` uses only the weight |
| Data | Modules exchange only the data they need, as parameters | `computeShipping(Kilograms weight, Kilometers distance)` |
| Message | Modules communicate only via messages/interfaces | Queues, interfaces, events |

### Cohesion types (from worst to best)
Coincidental (unrelated things in one place: `Utils`) → Logical (grouped by category: all
"handlers") → Temporal (things run at the same time: `init()` of everything) → Procedural →
Communicational (operate on the same data) → Sequential (output of one is input of next) →
**Functional** (all parts contribute to one well-defined task — the goal).

## 6. Component and package principles (Robert C. Martin)

Apply them to libraries, CMake targets, and components.

**Cohesion of components**
- **REP** — Reuse/Release Equivalence: the unit of reuse is the unit of release; version components as a whole.
- **CCP** — Common Closure: classes that change together belong together (SRP for components).
- **CRP** — Common Reuse: classes used together belong together; do not force users to depend on things they do not use (ISP for components).

**Coupling of components**
- **ADP** — Acyclic Dependencies: no cycles in the component dependency graph. Break cycles with an interface (DIP) or a new shared component.
- **SDP** — Stable Dependencies: depend in the direction of stability. Volatile components depend on stable ones, never the reverse.
- **SAP** — Stable Abstractions: a stable component should be abstract (interfaces, value types) so that stability does not block extension.

Metrics: instability `I = Ce / (Ca + Ce)` (efferent vs afferent couplings), abstractness
`A = abstract types / all types`; components should lie near the line `A + I = 1`.

## 7. Architecture-level application

- **Layered architecture**: dependencies go one way (presentation → application → domain →
  infrastructure, or with DIP: infrastructure depends on domain interfaces).
- **Hexagonal / Ports and Adapters / Clean Architecture**: the domain logic is in the center
  and has no dependency on middleware, OS, hardware, or frameworks. "Ports" are interfaces owned
  by the domain; "adapters" implement them for databases, network protocols, message buses,
  files, user interfaces, or test fakes.
  - Benefit: the same domain logic runs in tests, in simulation, and in production.
- **Functional core, imperative shell**: pure functions for decisions; a thin outer layer does
  I/O and state. The core is easy to test exhaustively.
- **Bounded contexts** (Domain-Driven Design): one model per context; translate at the borders
  (for example the `Customer` of billing vs the `Customer` of support).
- **Ubiquitous language**: class and function names use the terms of the domain experts and
  the requirements.

## 8. Code smells and the refactorings that fix them

| Smell | Sign | Refactoring |
|---|---|---|
| God class / Blob | One class knows and does everything; > ~500 lines, > ~20 public functions | Extract Class by responsibility; move functions to the data they use |
| Long function | Many levels, many comments that separate "steps" | Extract Function; replace temp with query; decompose conditional |
| Long parameter list | > 4-5 parameters | Introduce Parameter Object; preserve whole object; strong types |
| Primitive obsession | `double` for money, distance, time; `int` for IDs | Replace Primitive with Object (strong types, `std::chrono`) |
| Data clumps | Same group of fields/parameters appears together often | Extract struct |
| Feature envy | A function uses another class's data more than its own | Move Function to that class |
| Inappropriate intimacy | Classes access each other's internals (`friend` everywhere) | Move members; introduce an interface; merge if really one concept |
| Message chains | `a.b().c().d()` | Hide Delegate; move behavior |
| Middle man | A class only forwards calls | Remove Middle Man; inline |
| Shotgun surgery | One change touches many classes | Move related code together (CCP) |
| Divergent change | One class changes for many different reasons | Split by reason (SRP) |
| Switch on type | Same `switch (type)` in several places | Replace Conditional with Polymorphism or `std::variant` + `std::visit` |
| Refused bequest | Derived class does not use or rejects base behavior | Replace Inheritance with Delegation |
| Parallel inheritance hierarchies | Adding a subclass in one hierarchy forces one in another | Merge hierarchies; use composition |
| Speculative generality | Unused parameters, hooks, abstract classes with one implementation | Collapse Hierarchy; inline; remove |
| Temporary field | Member valid only in some states | Extract class; `std::optional`; `std::variant` states |
| Lazy class | Class does too little to justify itself | Inline Class; free function |
| Data class | Only fields + getters/setters, logic lives elsewhere | Move behavior to the data (Information Expert), or accept it as a plain `struct` DTO |
| Comments as deodorant | Comments explain confusing code | Rename, extract, simplify; then delete the comment |
| Duplicated code | Same logic in several places | Extract Function/Class (but check DRY limits) |
| Global state | Mutable globals, singletons | Inject dependencies; pass state explicitly |

Refactor in small steps, each covered by tests, each one compiling and passing. Keep refactoring
commits separate from behavior changes (required for clean reviews and for safety impact analysis).

## 9. OO metrics (Chidamber & Kemerer and others)

| Metric | Meaning | Guidance |
|---|---|---|
| WMC — Weighted Methods per Class | Sum of complexities of the methods | High → class does too much |
| DIT — Depth of Inheritance Tree | Levels from the root | Keep ≤ 3 (excluding pure interfaces) |
| NOC — Number of Children | Direct subclasses | Very high → base may be too generic |
| CBO — Coupling Between Objects | Number of classes a class depends on | Keep low (common target ≤ 10-14) |
| RFC — Response For a Class | Methods that can run in response to a message | High → complex, hard to test |
| LCOM — Lack of Cohesion of Methods | How much methods share members | High → split the class |
| Public API size | Number of public functions | Large → check SRP/ISP |
| Fan-in / fan-out | Dependents / dependencies | High fan-out = fragile; high fan-in = keep stable and abstract |

Use metrics as signals, not as goals by themselves.

## 10. OOP in real-time and safety-critical code

- Prefer composition, value types, and static polymorphism; keep virtual dispatch for real
  runtime variation at component boundaries (platform layers, adapters, strategies).
- Allocate all polymorphic objects statically or at initialization; no `new` at run time.
- Keep hierarchies shallow and the call graph simple and bounded.
- Avoid RTTI and `dynamic_cast` (often disabled); use `std::variant` or virtual functions.
- Exceptions-free construction: private constructor + `create()` returning `std::expected`.
- Document each class's role in the architecture (which component, which safety level, which interfaces).
- Isolation: classes of different safety levels do not share mutable state.

## 11. OOP design review checklist

- [ ] Each class has one clear responsibility and a domain name (SRP, ubiquitous language).
- [ ] Responsibilities placed with the class that has the information (Information Expert).
- [ ] Invariants are encapsulated; no getter/setter pairs that expose all state.
- [ ] Inheritance is used only for substitutable subtypes (LSP holds); hierarchies are shallow.
- [ ] Interfaces are small and owned by their clients (ISP, DIP).
- [ ] Dependencies point toward stable abstractions; no cycles between components (ADP, SDP, SAP).
- [ ] Coupling is data or message coupling; no global state, no control flags.
- [ ] Commands and queries are separated; queries are `const` and have no side effects.
- [ ] Illegal states cannot be represented (types, variants, factories).
- [ ] No code smells from section 8 without a reason.
- [ ] No speculative generality (YAGNI); extension points exist only where change is expected.
- [ ] Domain logic is independent of middleware/hardware (ports and adapters) and testable in isolation.
