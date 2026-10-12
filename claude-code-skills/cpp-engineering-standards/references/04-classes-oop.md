# 04 — Classes: mechanics, special members, inheritance, dependency injection, design patterns

This file covers how to write classes in C++. Design principles (the four pillars, SOLID,
GRASP, composition over inheritance, coupling, code smells) are in `05-oop-design-principles.md`.

## 1. `struct` vs `class`

- `struct`: passive data, all members public, members can vary independently, no invariant (CG C.2).
- `class`: has an invariant; data is private; public functions keep the invariant (CG C.2, C.9).
- Some safety standards forbid non-const member functions in a `struct`. Simple rule that fits all:
  if a type needs mutating member functions, make it a `class` with private data.
- Do not mix public and private data in one class (CG C.134).
- Minimize exposure: least visibility possible for members and functions (CG C.9).

## 2. Value types and entity types

Decide the kind of type before you write it. The kind decides copy, comparison, and ownership.

| Kind | Meaning | Copy | Compare | Examples |
|---|---|---|---|---|
| **Value type** ("regular") | Defined only by its value; two equal objects are interchangeable | Copyable, cheap or move-optimized | `==` (and `<=>` if ordered) | `Money`, `Point`, `Timestamp`, `std::string`, `Config` |
| **Entity type** | Has an identity and a lifetime; two objects with the same data are still different | Non-copyable (copy is a bug); move or reference | By identity (address/ID), not by value | `TcpConnection`, `FileWriter`, `Thread`, `Logger` |
| **Polymorphic base** | Interface used through pointer/reference | Non-copyable, non-movable (slicing) | Not comparable | `IStorage`, `IRenderer` |
| **RAII handle** | Owns one resource | Move-only | Rare | `UniqueFd`, `std::unique_ptr`, `std::jthread` |

Rules:
- Value types: make them regular (copy, move, `==`, default construction where it makes sense),
  immutable where practical, small, and `noexcept` movable. Prefer value types — they are easy
  to reason about, to test, and to use across threads.
- Entity types: `= delete` copy operations; give them a clear owner; do not put them in
  containers by value if their address must stay stable (use `std::unique_ptr` or stable storage).

## 3. Constructors

- A constructor creates a fully initialized, valid object (CG C.41). No `init()` two-phase
  initialization unless the context forbids failing constructors; then use a factory:
  ```cpp
  class Connection
  {
  public:
    [[nodiscard]] static std::expected<Connection, Error> create(const Endpoint& endpoint) noexcept;
  private:
    explicit Connection(const Endpoint& endpoint) noexcept;   // cannot fail; validation is in create()
  };
  ```
- Use member initializer lists. Prefer default member initializers for constant defaults
  (CG C.45, C.48).
- `explicit` on single-argument constructors (CG C.46) and conversion operators (CG C.164).
  C++20 `explicit(condition)` for wrapper templates that should be implicit only for safe
  conversions:
  ```cpp
  template <typename T>
  class Wrapper
  {
  public:
    template <typename U>
    explicit(!std::is_convertible_v<U, T>) Wrapper(U&& u) : m_value(std::forward<U>(u)) {}
  private:
    T m_value;
  };
  ```
- Delegating constructors to avoid duplication (CG C.51). Inheriting constructors
  (`using Base::Base;`) when the derived class adds no data (CG C.52).
- Do not call virtual functions in constructors or destructors (CG C.82): the dynamic type is
  the class being constructed, not the most derived class. A pure virtual call there is UB.
- A default constructor should be cheap and `noexcept` when possible.

### 3.1 Order of construction and destruction

1. Virtual base classes (depth-first, left to right).
2. Direct base classes, in the order of the **base list** (not the initializer list).
3. Data members, in the order of **declaration** (not the initializer list).
4. The constructor body.

Destruction runs in exactly the reverse order. Consequences:
- Write the initializer list in declaration order (`-Wreorder` makes a mismatch an error).
- Never initialize a member from another member declared after it.
- Declare a mutex before the data and the threads that use it; declare `std::jthread` members
  **last**, so they stop and join before the data they use is destroyed.
- If a constructor throws, the already-constructed members and bases are destroyed; the
  destructor of the class itself does not run. RAII members make this safe.

## 4. Special members: Rule of Zero / Five

- **Rule of Zero** (CG C.20): use members that manage themselves (RAII members, containers,
  smart pointers). Do not declare any special member.
- **Rule of Five** (CG C.21): if you declare or `=delete` any of destructor, copy constructor,
  copy assignment, move constructor, move assignment — declare all five (use `= default` /
  `= delete` where right).
- Move operations and `swap` are `noexcept` (CG C.66, C.85). Otherwise `std::vector` copies
  instead of moves when it grows.
- Moved-from objects must be valid (destructible and assignable). Document if any other
  operation is allowed.
- Non-copyable types (handles, mutex owners): `= delete` the copy operations; allow move if useful.
- Use `= default` instead of empty bodies, so the type stays trivial when it can.

### 4.1 Move-only RAII handle

```cpp
class UniqueFd
{
public:
  explicit UniqueFd(int fd) noexcept : m_fd{fd} {}
  ~UniqueFd() { reset(); }

  UniqueFd(const UniqueFd&) = delete;
  UniqueFd& operator=(const UniqueFd&) = delete;

  UniqueFd(UniqueFd&& other) noexcept : m_fd{std::exchange(other.m_fd, kInvalid)} {}
  UniqueFd& operator=(UniqueFd&& other) noexcept
  {
    if (this != &other)
    {
      reset();
      m_fd = std::exchange(other.m_fd, kInvalid);
    }
    return *this;
  }

  [[nodiscard]] int get() const noexcept { return m_fd; }
  void reset() noexcept
  {
    if (m_fd != kInvalid)
    {
      static_cast<void>(::close(m_fd));   // close cannot be retried safely; error is ignored by design
      m_fd = kInvalid;
    }
  }

private:
  static constexpr int kInvalid{-1};
  int m_fd{kInvalid};
};
```

### 4.2 Copy-and-swap (strong exception guarantee)

Use it when a class manages a resource by hand and copy assignment must give the strong
guarantee. One assignment operator taking the argument **by value** serves both copy and move.

```cpp
class Buffer
{
public:
  explicit Buffer(std::size_t size) : m_data{std::make_unique<std::byte[]>(size)}, m_size{size} {}

  Buffer(const Buffer& other) : m_data{std::make_unique<std::byte[]>(other.m_size)}, m_size{other.m_size}
  {
    std::copy_n(other.m_data.get(), m_size, m_data.get());
  }
  Buffer(Buffer&& other) noexcept = default;

  Buffer& operator=(Buffer other) noexcept   // copy (or move) happens in the parameter
  {
    swap(*this, other);                        // cannot fail
    return *this;                              // old state is destroyed with `other`
  }
  ~Buffer() = default;

  friend void swap(Buffer& a, Buffer& b) noexcept
  {
    using std::swap;
    swap(a.m_data, b.m_data);
    swap(a.m_size, b.m_size);
  }

  [[nodiscard]] std::size_t size() const noexcept { return m_size; }

private:
  std::unique_ptr<std::byte[]> m_data;
  std::size_t m_size{0U};
};
```
Cost: one extra move per move-assignment. In hot code, write separate copy and move assignment.
Prefer Rule of Zero (here: `std::vector<std::byte>`) when possible — it gives all of this for free.

## 5. Destructors

- Never throw from a destructor (implicitly `noexcept`) (CG C.36, E.16).
- Base class destructor: `public virtual` if you delete through a base pointer, else
  `protected` non-virtual (CG C.35).
- A destructor releases only what the class owns. Keep it simple and bounded in time.
- A class with a virtual function and a public non-virtual destructor is a bug
  (`-Wnon-virtual-dtor`, `-Wdelete-non-virtual-dtor`).

## 6. Member functions and static members

- `const` member functions by default (CG Con.2). Use `mutable` only for caches/mutexes that
  do not change the observable state, and protect them for thread safety.
- Ref-qualifiers (`&`, `&&`) when a getter must not return a reference to a temporary's data:
  ```cpp
  const std::string& name() const& { return m_name; }
  std::string name() && { return std::move(m_name); }
  ```
- C++23 "deducing this" (`this auto&& self`) removes const/non-const duplication.
- Getters and setters: provide them only when they add value (validation, invariant, ABI).
  A trivial getter + setter pair for every member means the type should be a `struct`.
- Prefer non-member non-friend functions for operations that need only the public interface:
  they improve encapsulation and keep the class small.
- `static` data members: `inline static` (C++17) or `static constexpr` in the class; no separate
  definition in a `.cpp` file needed. Mutable `static` data is global state — avoid it (section 10.7).
- `static` member functions when the function needs private access but no object. A free
  function in the same namespace is often better.
- `friend`: avoid. Acceptable for hidden-friend operators and `swap`, iterators, and tightly
  coupled helper types. Some safety standards forbid it; follow the project.
- **Hidden friend idiom**: define symmetric operators and `swap` as `friend` inside the class.
  They are found only by ADL, so they do not pollute overload sets and compile faster:
  ```cpp
  class Money
  {
  public:
    [[nodiscard]] static constexpr Money fromCents(std::int64_t cents) noexcept { return Money{cents}; }
    friend constexpr Money operator+(Money a, Money b) noexcept { return Money{a.m_cents + b.m_cents}; }
    friend constexpr bool operator==(Money a, Money b) noexcept = default;
  private:
    constexpr explicit Money(std::int64_t cents) noexcept : m_cents{cents} {}
    std::int64_t m_cents{0};
  };
  ```
- Nested types: use them for types that only make sense with the outer class
  (`Parser::Config`, `Parser::Error`). Keep public nested types small.

## 7. Design principles (pointers)

- Encapsulation, abstraction, getters/setters, Tell-Don't-Ask, Law of Demeter: `05-oop-design-principles.md` sections 1.1, 1.2, and 4.
- SOLID with examples (Liskov, interface segregation, dependency inversion): `05-oop-design-principles.md` section 2.
- Composition over inheritance, and when inheritance is right: `05-oop-design-principles.md` section 1.3.
- Which polymorphism mechanism to choose (virtual, `std::variant`, templates, type erasure) and its cost: `05-oop-design-principles.md` section 1.4.

## 8. Inheritance mechanics

- Interfaces: abstract classes with only pure virtual functions and a virtual destructor, no
  data (CG C.121, I.25).
- Exactly one of `virtual`, `override`, `final` on a virtual function (CG C.128). `override` on
  every override (`-Wsuggest-override`, `-Winconsistent-missing-override`).
- `final` on classes not designed for inheritance; it also helps devirtualization.
- Polymorphic classes: suppress public copy and move (CG C.67) to prevent slicing. Make the
  copy operations `protected` in the base; provide a virtual `clone()` if copies are needed
  (section 8.2).
- Do not change the meaning of a base function in an override.
- No multiple inheritance of implementation. Multiple inheritance of interfaces is fine.
- Virtual inheritance only for diamond interfaces, rarely.
- Do not hide base functions by accident; use `using Base::f;` when you overload in the derived class.
- Default arguments on virtual functions: avoid (CG C.140). The default comes from the static
  type, the body from the dynamic type.
- Keep hierarchies shallow and wide rather than deep.
- Prefer static polymorphism (templates, CRTP, concepts) or `std::variant` + `std::visit` when
  the set of types is closed and performance matters.

### 8.1 Non-Virtual Interface (NVI)

Public functions are non-virtual and keep the contract (checks, logging, locking, timing);
private virtual functions are the customization points. The base class controls the invariant.

```cpp
class Filter
{
public:
  virtual ~Filter() = default;

  [[nodiscard]] float apply(float input)          // non-virtual: contract in one place
  {
    if (!std::isfinite(input))
    {
      return m_lastOutput;                          // reject invalid input for every filter
    }
    m_lastOutput = doApply(input);
    return m_lastOutput;
  }

private:
  [[nodiscard]] virtual float doApply(float input) = 0;   // customization point
  float m_lastOutput{0.0F};
};

class MovingAverageFilter final : public Filter
{
public:
  explicit MovingAverageFilter(float weight) noexcept : m_weight{weight} {}
private:
  [[nodiscard]] float doApply(float input) override
  {
    m_state += m_weight * (input - m_state);
    return m_state;
  }
  float m_weight;
  float m_state{0.0F};
};
```

### 8.2 Polymorphic copy (`clone`) and covariant return types

```cpp
class Shape
{
public:
  virtual ~Shape() = default;
  [[nodiscard]] std::unique_ptr<Shape> clone() const { return doClone(); }   // NVI
  [[nodiscard]] virtual float area() const = 0;
protected:
  Shape() = default;
  Shape(const Shape&) = default;              // derived classes can copy; users cannot slice
  Shape& operator=(const Shape&) = default;
private:
  [[nodiscard]] virtual std::unique_ptr<Shape> doClone() const = 0;
};

class Circle final : public Shape
{
public:
  explicit Circle(float r) noexcept : m_r{r} {}
  [[nodiscard]] float area() const override { return std::numbers::pi_v<float> * m_r * m_r; }
private:
  [[nodiscard]] std::unique_ptr<Shape> doClone() const override { return std::make_unique<Circle>(*this); }
  float m_r;
};
```
Covariant return types work only with raw pointers and references (`Circle* clone() const
override` for `virtual Shape* clone() const`), not with smart pointers. With `std::unique_ptr`,
use the NVI pattern above, plus a non-virtual `cloneCircle()` in the derived class if callers need
the derived type.

## 9. Dependency injection (DI)

Give a class its dependencies from outside instead of creating them inside. Benefits:
testability (fakes/mocks), replaceability (simulation vs production), explicit coupling.

| Form | How | Cost | Use |
|---|---|---|---|
| Constructor injection (interface) | `explicit ReportGenerator(IStorage& storage)` | One virtual call per use | Default; runtime choice; easy fakes |
| Template injection (static) | `template <Storage S> class ReportGenerator { S& m_storage; };` | Zero | Hot paths; fakes are types |
| Function injection | Pass a callable: `RateLimiter(Clock clock)` with `using Clock = std::function<TimePoint()>` | Possible allocation | Small dependencies (clocks, random sources) |
| Link-time injection | Different `.cpp` linked for different builds | Zero | C-style platform layers |

Rules:
- Inject through the constructor; store references (non-owning, never null) or
  `std::unique_ptr` (owning). Avoid setter injection (the object is incomplete until the setter
  runs).
- Build the object graph in one place (the **composition root**: `main`, a factory, a
  component initializer). Business classes never call `new` on their dependencies.
- No service locator or global registry: it hides dependencies just like a global variable.
- Inject time: a clock interface or template parameter makes timeouts deterministic without
  sleeping.

```cpp
class IClock
{
public:
  virtual ~IClock() = default;
  [[nodiscard]] virtual std::chrono::steady_clock::time_point now() const = 0;
protected:
  IClock() = default;
  IClock(const IClock&) = default;
  IClock& operator=(const IClock&) = default;
};

class SessionTimeout                 // depends only on the abstraction
{
public:
  SessionTimeout(const IClock& clock, std::chrono::seconds limit) noexcept
    : m_clock{clock}, m_limit{limit}, m_start{clock.now()} {}
  [[nodiscard]] bool isExpired() const { return (m_clock.now() - m_start) >= m_limit; }
private:
  const IClock& m_clock;             // non-owning; the composition root owns the clock
  std::chrono::seconds m_limit;
  std::chrono::steady_clock::time_point m_start;
};

// Fake implementation: deterministic, no waiting. Same interface as the real clock.
class FakeClock final : public IClock
{
public:
  [[nodiscard]] std::chrono::steady_clock::time_point now() const override { return m_now; }
  void advance(std::chrono::seconds delta) noexcept { m_now += delta; }
private:
  std::chrono::steady_clock::time_point m_now{};
};
```
Declare `m_start` after `m_clock` and `m_limit`: members are initialized in declaration order
(section 3.1).

## 10. Design patterns in modern C++

Use a pattern only when it solves a real problem in the code. Prefer the simplest form.
Each pattern below lists a classic form and a form without heap for REAL-TIME/SAFETY-CRITICAL contexts.

### 10.1 Strategy — choose an algorithm

| Form | Code |
|---|---|
| Runtime, classic | interface `IStrategy` + `std::unique_ptr<IStrategy>` member |
| Runtime, no heap | interface + reference to a statically allocated strategy; or `std::variant<StrategyA, StrategyB>` |
| Compile time | template parameter (policy-based design) |
| Lightweight | `std::function` / `function_ref` member (callable) |

```cpp
template <typename CompressionPolicy>
class ArchiveWriter
{
public:
  [[nodiscard]] std::vector<std::byte> pack(std::span<const std::byte> data) const
  {
    return m_compression.compress(data);
  }
private:
  [[no_unique_address]] CompressionPolicy m_compression{};   // empty policy costs no space
};
```

### 10.2 Observer — notify listeners

Main risks: dangling observers, callbacks under lock, re-entrancy (an observer that unsubscribes
during notification).

```cpp
template <typename Event>
class Signal
{
public:
  using Slot = std::function<void(const Event&)>;

  class Connection                  // RAII: unsubscribes in its destructor
  {
  public:
    Connection(Signal& s, std::size_t id) noexcept : m_signal{&s}, m_id{id} {}
    ~Connection() { if (m_signal != nullptr) { m_signal->disconnect(m_id); } }
    Connection(Connection&& o) noexcept : m_signal{std::exchange(o.m_signal, nullptr)}, m_id{o.m_id} {}
    Connection& operator=(Connection&&) = delete;
    Connection(const Connection&) = delete;
    Connection& operator=(const Connection&) = delete;
  private:
    Signal* m_signal;
    std::size_t m_id;
  };

  [[nodiscard]] Connection connect(Slot slot)
  {
    m_slots.emplace_back(m_nextId, std::move(slot));
    return Connection{*this, m_nextId++};
  }

  void emit(const Event& e) const
  {
    const auto snapshot = m_slots;  // copy: observers may disconnect during the call
    for (const auto& [id, slot] : snapshot) { slot(e); }
  }

private:
  void disconnect(std::size_t id)
  {
    std::erase_if(m_slots, [id](const auto& entry) { return entry.first == id; });
  }
  std::vector<std::pair<std::size_t, Slot>> m_slots;
  std::size_t m_nextId{0U};
};
```
Single-threaded version shown. For several threads: protect `m_slots` with a mutex, copy the
snapshot under the lock, and call the slots **after** you release the lock. The signal must
outlive every `Connection`. No-heap form: fixed-capacity array of non-allocating delegates
registered at initialization. Libraries: Boost.Signals2, sigslot.

### 10.3 State — behavior depends on mode

Prefer `std::variant` states with data per state: impossible states cannot be represented, and
`std::visit` forces you to handle every state.

```cpp
struct Disconnected {};
struct Connecting { std::uint8_t attempt; };
struct Connected { int socketFd; };
using ConnectionState = std::variant<Disconnected, Connecting, Connected>;

constexpr std::uint8_t kMaxAttempts{3U};

// Overloaded: see 12-standard-library.md section 5
[[nodiscard]] ConnectionState onTimeout(const ConnectionState& state)
{
  return std::visit(Overloaded{
                      [](const Disconnected& s) -> ConnectionState { return s; },
                      [](const Connecting& c) -> ConnectionState
                      {
                        if (c.attempt >= kMaxAttempts)
                        {
                          return Disconnected{};                       // give up
                        }
                        return Connecting{static_cast<std::uint8_t>(c.attempt + 1U)};
                      },
                      [](const Connected&) -> ConnectionState { return Disconnected{}; }},
                    state);
}
```
Alternatives: `enum class` + `switch` (simple, MISRA-friendly), transition tables, libraries
such as Boost.SML (compile-time, no heap) or Boost.MSM. In SAFETY-CRITICAL code, document every
state and transition.

### 10.4 Visitor — operations over a closed set of types

Modern form: `std::variant` + `std::visit` with an `Overloaded` set (section 10.3). The classic
double-dispatch visitor (virtual `accept`) is needed only for open class hierarchies that cannot
use `std::variant`.

### 10.5 Factory — hide the concrete type

```cpp
[[nodiscard]] std::unique_ptr<IImageDecoder> makeDecoder(ImageFormat format)
{
  switch (format)
  {
    case ImageFormat::Png: return std::make_unique<PngDecoder>();
    case ImageFormat::Jpeg: return std::make_unique<JpegDecoder>();
  }
  return nullptr;   // unreachable for valid enum values; caller checks
}
```
No-heap form: return `std::variant<PngDecoder, JpegDecoder>`, or construct into
caller-provided storage. Named constructors (`static create(...)` returning `std::expected`,
section 3) are factories for one type — use them when construction can fail.

### 10.6 Builder — many optional parameters

C++20 designated initializers often replace a builder:
```cpp
struct HttpClientConfig
{
  std::chrono::milliseconds timeout{5000};
  std::uint8_t maxRetries{3U};
  std::uint16_t port{443U};
  bool followRedirects{false};
};
const HttpClientConfig cfg{.timeout = std::chrono::milliseconds{2000}, .followRedirects = true};   // others keep defaults
```
Use a real builder only when parameters depend on each other or need validation; let `build()`
return `std::expected<T, Error>`.

### 10.7 Singleton — avoid

A singleton is a global variable with extra steps: hidden dependency, hidden shared state,
initialization and destruction order problems, hard to test, data races.

Do instead:
1. Create one instance in the composition root and inject it (section 9). "Only one" becomes a
   decision of `main`, not of the class.
2. If a true process-wide object is required (logger backend, OS resource): use a function that
   returns a reference to a function-local static (see `06-memory-ownership-lifetime.md`
   section 10), and put an injectable interface in front of it.
3. Real-time code: one statically allocated object per resource, created at start-up, passed by
   reference.

### 10.8 Other patterns (short)

| Pattern | Modern C++ form | Note |
|---|---|---|
| Adapter | Wrapper class or free functions that map one interface to another | Isolate third-party and C APIs |
| Facade | One small class over a complex subsystem | Good boundary for a component |
| Decorator | Wrapper that implements the same interface and forwards | Adds logging, metrics, checks without changing the original |
| Command | Callable objects (`std::function`, lambdas) or `std::variant` of command structs | Queues, undo, scheduling |
| Template Method | NVI (section 8.1) | |
| CRTP / mixins | `template <typename Derived> class Comparable` | Static polymorphism; see `11-templates-modern-cpp.md` |
| Type erasure | `std::function`, `std::any`, custom "concept/model" class | Value semantics with runtime polymorphism |
| Object pool | Preallocated storage + free list | See `07-memory-allocation.md` section 3.2 |
| RAII | Everywhere | The most important C++ "pattern" |

## 11. Operators and comparisons

- Implement operators with their usual meaning (CG C.160).
- C++20: `auto operator<=>(const T&) const = default;` and `bool operator==(const T&) const = default;`
  for member-wise comparison.
- Binary arithmetic operators as non-members (hidden friends) based on compound assignment:
  `a + b` uses `a += b`.
- Equality of floating members: decide the semantics explicitly; default `==` compares exactly.
- Do not overload `&&`, `||`, `,`, or unary `&`.

## 12. pImpl and ABI

- pImpl (`std::unique_ptr<Impl>`) to hide implementation, cut compile times, and keep ABI stable.
  Declare the destructor in the header and define it in the `.cpp` (incomplete type rule).
- Cost: one heap allocation and one indirection. Do not use in hot paths or in no-heap code
  (use a fast pImpl with aligned in-place storage, if needed).

## 13. Class mechanics checklist

- [ ] Kind of type decided: value, entity, polymorphic base, or RAII handle.
- [ ] All members initialized by every constructor; initializer list in declaration order.
- [ ] Invariant documented and checked at construction; kept by every public mutating function, also on error paths.
- [ ] Special members follow Rule of Zero or Five; moves and `swap` are `noexcept`.
- [ ] No public data in a class with an invariant; no non-const references to internals returned.
- [ ] Polymorphic bases: virtual or protected destructor, no public copy (no slicing), `override`/`final` everywhere.
- [ ] Dependencies injected through constructors; no singletons, service locators, or hidden globals.
- [ ] Patterns used only where they solve a real problem; heap-free forms in REAL-TIME/SAFETY-CRITICAL code.
- [ ] Thread safety documented: which functions can run at the same time.
- [ ] Design principles checked with `05-oop-design-principles.md` section 11.
