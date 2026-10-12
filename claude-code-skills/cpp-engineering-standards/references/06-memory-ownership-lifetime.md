# 06 — Memory model, ownership, RAII, smart pointers, lifetime

Allocation strategies, custom allocators, `pmr`, no-allocation policy, stack, data layout, and
shared memory are in `07-memory-allocation.md`.

## 1. Principles

1. Every resource (memory, file, socket, lock, handle, thread, GPU buffer) has exactly one
   clear owner, and that owner is an RAII object (CG R.1, P.8).
2. Ownership is visible in the type. A raw pointer or reference never owns (CG R.3, R.4).
3. Prefer automatic storage (stack) and members by value. Allocate on the heap only when size or
   lifetime requires it (CG R.5).
4. Know the lifetime of everything you refer to. Most C++ memory bugs are lifetime bugs.
5. In real-time and safety code, memory use is static and bounded, and known at design time.
6. Make memory errors impossible by design (types, RAII, views with clear scope) before you rely
   on tools to find them.

## 2. The C++ memory model (fundamentals)

### 2.1 Storage duration

| Storage duration | Created | Destroyed | Where (typical) | Notes |
|---|---|---|---|---|
| **Automatic** | When execution reaches the declaration | At the end of the scope (reverse order) | Stack / registers | Fast, deterministic; limited size |
| **Static** | Before `main` (constant or dynamic init) or at first use (function-local `static`) | After `main` returns (reverse order of construction) | `.data` (initialized), `.bss` (zero), `.rodata` (const) | Init order across TUs is unspecified |
| **Thread** (`thread_local`) | At thread start or first use | At thread exit | Thread-local storage block | Access cost on some platforms; dynamic init guard |
| **Dynamic** | `new`, allocator, `malloc` | `delete`, allocator, `free` | Heap / custom pool | Must be owned by RAII |

Memory sections in a typical executable:

| Section | Content | Properties |
|---|---|---|
| `.text` | Code | Read-only, shared between processes |
| `.rodata` | `const`/`constexpr` data, string literals, vtables | Read-only, shared between processes |
| `.data` | Initialized non-const statics | Writable, per process |
| `.bss` | Zero-initialized statics | Writable, zeroed at start, no space in the file |
| Heap | Dynamic allocation | Writable, managed by the allocator |
| Stack(s) | One per thread | Writable, fixed size per thread |

Consequence: `constexpr`/`const` tables live in read-only memory: they cannot be corrupted by a
stray write, need no start-up initialization, and are shared between processes. Make lookup
tables `constexpr` or `const`.

### 2.2 Object lifetime

- An object's lifetime **begins** when storage is obtained and initialization is complete. It
  **ends** when the destructor call starts (class types) or when the storage is released or reused.
- Using an object outside its lifetime is UB, even if the memory is still there.
- Members are constructed in declaration order and destroyed in reverse order. Bases before members.
- Temporaries are destroyed at the end of the full expression, unless their lifetime is extended (section 7.1).
- **Trivially copyable** types (no user-provided copy/move/destructor, no virtual) can be copied
  with `std::memcpy`. **Implicit-lifetime** types (C++20: scalars, arrays, aggregates with trivial
  destructor, trivially copyable classes) start living implicitly in memory from `malloc`,
  `std::memcpy`, or `std::start_lifetime_as` (C++23).
- **Object representation**: the bytes of an object, including **padding** with indeterminate
  values. Never compare objects with `memcmp` and never hash or send raw bytes without zeroing padding.

### 2.3 Value categories and memory

- **lvalue**: has identity (a named object). **prvalue**: a pure value (since C++17, materialized
  directly into its destination: guaranteed copy elision). **xvalue**: an object whose resources
  can be reused (`std::move(x)`).
- `std::move` does not move anything. It is a cast to an rvalue reference that **allows** a move.
- After a move, a standard library object is in a "valid but unspecified" state. Only assign to
  it, destroy it, or call functions without preconditions (`clear()`, `size()`).

## 3. Ownership vocabulary

| Type | Meaning | Size (64-bit, typical) |
|---|---|---|
| `T` (value member, local) | Owns, exclusive, fixed lifetime | `sizeof(T)` |
| `std::unique_ptr<T>` | Owns, exclusive, transferable, may be null | 8 bytes (stateless deleter) |
| `std::unique_ptr<T, void(*)(T*)>` | Same, with a function-pointer deleter | 16 bytes |
| `std::shared_ptr<T>` | Shared ownership; last owner destroys | 16 bytes + control block |
| `std::weak_ptr<T>` | Non-owning observer of a `shared_ptr`; breaks cycles | 16 bytes |
| `T&` / `const T&` | Non-owning, never null, must not outlive the object | — |
| `T*` / `const T*` | Non-owning, may be null, must not outlive the object | 8 bytes |
| `gsl::not_null<T*>` | Non-owning, never null (checked) | 8 bytes |
| `gsl::owner<T*>` | Legacy owning raw pointer, marked for tools; migrate to RAII | 8 bytes |
| `std::span<T>` / `std::string_view` | Non-owning view of contiguous data | 16 bytes |
| `std::optional<T>` | Owns a value that may be absent; no heap | `sizeof(T)` + flag + padding |
| Container (`std::vector<T>`) | Owns all elements | 24 bytes + heap buffer |

Rules:
- `std::unique_ptr` by default for heap objects. `std::shared_ptr` only when ownership is really
  shared and the last owner is unknown at design time (CG R.21).
- Create with `std::make_unique` / `std::make_shared` (CG R.22, R.23) — no `new` in user code.
- No `malloc`/`free`/`realloc` in C++ code (CG R.10). No naked `new`/`delete` (CG R.11). Never
  mix `new`/`free`, `malloc`/`delete`, or `new[]`/`delete`.
- One resource acquisition per statement (CG R.13).

## 4. RAII in depth

**Rule**: acquire a resource in a constructor; release it in the destructor; never let it exist
without an owner object.

Why it works: destructors run on every exit path — normal return, early return, `break`, and
exception unwinding. RAII gives leak freedom and exception safety without `try`/`finally`.

RAII class rules:
1. The constructor acquires the resource or fails (throws / factory returns an error). No "half-open" objects.
2. The destructor releases it, never throws, and handles the "moved-from/empty" state.
3. Copy is deleted unless copying the resource is meaningful (then deep copy).
4. Move transfers ownership and leaves the source empty (`std::exchange`).
5. Provide `get()` for C APIs, `release()` only if needed, `reset()` for early release.
6. Keep the class small: one resource per RAII class. Compose RAII classes for several resources.

### 4.1 `std::unique_ptr` with custom deleters (C resources)

```cpp
struct FileCloser
{
  void operator()(std::FILE* file) const noexcept
  {
    if (file != nullptr)
    {
      static_cast<void>(std::fclose(file));
    }
  }
};
using FilePtr = std::unique_ptr<std::FILE, FileCloser>;   // empty deleter: no size overhead

[[nodiscard]] FilePtr openFile(const char* path, const char* mode) noexcept
{
  return FilePtr{std::fopen(path, mode)};                 // null on failure; caller checks
}
```
Same pattern for `SDL_Window`, `sqlite3`, `CURL`, OS handles. For non-pointer handles (`int fd`),
write a small RAII class (see `04-classes-oop.md` section 4).

### 4.2 Scope guard (one-off cleanup)

```cpp
template <typename F>
class ScopeExit
{
public:
  explicit ScopeExit(F f) noexcept : m_f{std::move(f)} {}
  ~ScopeExit() { if (m_active) { m_f(); } }
  void release() noexcept { m_active = false; }   // commit: cancel the cleanup
  ScopeExit(const ScopeExit&) = delete;
  ScopeExit& operator=(const ScopeExit&) = delete;
private:
  F m_f;
  bool m_active{true};
};
```
Use it for rollback in multi-step operations: create the guard after each step; `release()`
all guards when every step succeeded. Libraries: `gsl::finally`, `std::experimental::scope_exit`.

### 4.3 Other RAII uses

- Locks (`std::scoped_lock`), guards that set and restore a mode, flag, or hardware state.
- State restorers (save a flag/format/precision, restore in the destructor).
- Timers and trace spans (measure a scope).
- Subscription handles (unregister an observer in the destructor).
- Transactions (roll back unless committed).

## 5. Smart pointers in depth

### 5.1 `std::unique_ptr<T>`

- Zero overhead compared with a raw pointer when the deleter is empty (default or stateless functor).
- Move-only. Transfers ownership with `std::move`.
- `std::unique_ptr<T[]>` for arrays (calls `delete[]`) — but prefer `std::vector` or `std::array`.
- Works with **incomplete types** (pImpl), if the destructor of the owner is defined where `T` is complete.
- `release()` gives up ownership without deleting — the caller must take ownership immediately.
  A `release()` without a new owner is a leak. Treat every `release()` as suspicious.
- Polymorphic deletion needs a virtual destructor in the base, or a deleter that knows the real type.
- Factory functions return `std::unique_ptr<T>`; the caller can convert it into a `shared_ptr` if needed.
- `std::make_unique_for_overwrite<T[]>(n)` (C++20): no value-initialization (no zeroing) for buffers you fill at once.

### 5.2 `std::shared_ptr<T>`

Structure: two pointers — the object pointer and the **control block** pointer. The control block
holds the strong count, the weak count, the deleter, and the allocator.

| Creation | Allocations | Notes |
|---|---|---|
| `std::make_shared<T>(args...)` | 1 (object and control block together) | Faster, better locality. But the object memory is freed only when the **last `weak_ptr`** is gone (destructor runs earlier) |
| `std::shared_ptr<T>(new T(...))` | 2 | Object memory freed when the strong count reaches zero. Not exception-safe in some expressions before C++17 |
| `std::allocate_shared<T>(alloc, args...)` | 1, from a custom allocator | Use with pools/pmr |

Costs and rules:
- Reference count changes are **atomic** operations (tens of cycles, cache-line traffic between cores). Copying a `shared_ptr` in a hot path is expensive. Pass `const T&` or `T*` to functions that only use the object.
- The count is thread-safe; the **object** is not. Two threads that change the object need a mutex. Two threads that change the **same `shared_ptr` variable** need `std::atomic<std::shared_ptr<T>>` (C++20) or a mutex.
- Never create two `shared_ptr` from the same raw pointer (double delete). Never `shared_ptr<T>(this)`. Use `std::enable_shared_from_this<T>` and `shared_from_this()` (only after the object is already owned by a `shared_ptr`; otherwise `std::bad_weak_ptr` in C++17).
- **Aliasing constructor** `std::shared_ptr<Member>(owner, &owner->member)` shares ownership of the owner while pointing to a member.
- **Cycles leak**: A owns B, B owns A → neither is destroyed. Break cycles with `weak_ptr` (back-pointers, parent links, observers) or redesign ownership as a tree.
- `std::shared_ptr<const T>` for immutable shared data (snapshots, configuration): safe to read from many threads.
- Destruction happens on the thread that releases the last reference — it can be a real-time thread. Avoid releasing the last reference in time-critical code (hand off to a cleanup thread if needed).

### 5.3 `std::weak_ptr<T>`

- Observes a `shared_ptr`-owned object without keeping it alive.
- Always access through `lock()`, which returns a `shared_ptr` (null if the object is gone):
  ```cpp
  if (const auto target = m_target.lock())   // atomic check-and-acquire
  {
    target->notify();
  }
  ```
- Never check `expired()` and then `lock()` — the object can die between the two calls.
- Use for caches, observers, parent pointers, and callbacks that must not extend the lifetime of `this`.

### 5.4 Passing smart pointers (CG R.30-R.37)

| Signature | Meaning |
|---|---|
| `void f(T&)` / `void f(const T&)` | Uses the object; does not care about ownership (preferred) |
| `void f(T*)` | Uses the object; may be null |
| `void f(std::unique_ptr<T>)` | Takes ownership (sink) |
| `void f(std::unique_ptr<T>&)` | Will reseat the pointer (rare) |
| `void f(std::shared_ptr<T>)` | Shares ownership (keeps a copy) |
| `void f(const std::shared_ptr<T>&)` | Might share ownership (copies it conditionally) |
| `void f(std::shared_ptr<T>&)` | Will reseat the pointer (rare) |

Do not pass a smart pointer only to use the object. Do not pass a dereferenced smart pointer
obtained from a non-local shared object without keeping a local copy of the `shared_ptr` (CG R.37):
another thread or callback could reset it during the call.

### 5.5 Choosing an owner

| Situation | Owner |
|---|---|
| Part with the same lifetime as the whole | Member by value |
| Optional part, polymorphic part, or large part that must not be on the stack | `std::unique_ptr` member |
| Many objects of one type, same lifetime | Container of values (`std::vector<T>`) |
| Objects with stable addresses in a growing set | `std::deque<T>`, `std::vector<std::unique_ptr<T>>`, `std::list`, or a pool with handles |
| Object referenced by several owners with unknown last user | `std::shared_ptr` |
| Immutable shared snapshot | `std::shared_ptr<const T>` |
| Objects that refer to each other (graphs) | One owner for all nodes (arena/vector) + indexes or raw non-owning pointers between nodes |

## 6. Raw pointers, references, and views

- A reference parameter or member means "non-null, non-owning, outlives this use".
- A raw pointer means "non-owning, may be null". Check before use, or use `gsl::not_null`.
- Prefer indexes or handles over pointers into containers that can grow.
- Pointer arithmetic only inside arrays and only within bounds; prefer `std::span` and iterators.
- Views (`std::string_view`, `std::span`, ranges views, iterators) are pointers in disguise.
  They follow all pointer lifetime rules.
  - As parameters: always fine (the caller keeps the data alive during the call).
  - As return values or members of a returned struct: fine when the view points into data that
    the **caller** passed in and owns (a "borrowed" result, for example a parser that returns a
    `std::span` into the input buffer). Document it in the API: "valid while `input` is alive and
    unchanged". Mark such functions so that callers cannot pass a temporary (for example
    `= delete` an rvalue overload).
  - Never return a view into a local, a temporary, or internal state that the object can change
    or reallocate later (return a copy or an owning type instead).
  - As long-lived class members: only with a documented, enforced lifetime relation; otherwise
    store an owning type.
- Reference members make a class non-assignable. Use a pointer member (or `std::reference_wrapper`)
  if the class must be assignable.

## 7. Lifetime in depth

### 7.1 Temporary lifetime extension — exact rules

A temporary bound **directly** to a `const T&` or `T&&` local reference lives as long as the reference:
```cpp
const std::string& name = makeName();   // OK: temporary lives until `name` goes out of scope
```
It is **not** extended in these cases:
- Through a function that returns a reference: `const auto& x = std::max(a(), b());` dangles.
- Through member access of a function result returning by reference: `const auto& v = getObj().items();` where `items()` returns `const std::vector<int>&` — `getObj()` temporary dies, `v` dangles.
- A reference member bound in a **constructor's member initializer list**: dangles (and many compilers reject it).
- A temporary bound to a reference parameter: lives only until the end of the full call expression.
- `new S{temporary}` with a reference member: dangles.

It **is** extended for aggregate initialization of a local: `struct S { const std::string& r; }; S s{std::string{"x"}};`
keeps the temporary alive with `s` — but this is fragile; do not design types this way.

### 7.2 Dangling — complete catalog

| Trap | Example | Fix |
|---|---|---|
| View to a temporary | `std::string_view sv = getName();` (returns `std::string`) | Keep a `std::string` |
| Span to a temporary | `std::span<const int> s = makeVector();` | Keep the vector |
| View member | Class stores `std::string_view m_name` from a constructor argument | Store `std::string`, or document and enforce that the source outlives the object |
| Reference member set in constructor | `S(const std::string& s) : m_ref{s} {}` called with a temporary | Store by value |
| Returning a reference/pointer/view to a local | `std::string_view f() { std::string s; return s; }` | Return by value |
| Range-for over a member of a temporary (before C++23) | `for (auto& x : getObj().items())` | `const auto obj = getObj(); for (auto& x : obj.items())` |
| `std::min`/`std::max`/`std::clamp` with temporaries | `const auto& m = std::min(f(), g());` | Store by value: `const auto m = std::min(f(), g());` |
| Lambda capture by reference outlives scope | `[&] { use(local); }` stored or sent to another thread | Capture by value/move |
| `this` captured in asynchronous callback | Object destroyed before callback | Cancel in destructor; `weak_ptr` guard; subscription handle |
| Coroutine reference parameters | `task f(const std::string& s)` called with a temporary; the coroutine runs later | Coroutine parameters by value |
| `std::initializer_list` returned or stored | `std::initializer_list<int> f() { return {1, 2}; }` | Return `std::array`/`std::vector` |
| Ranges view of a temporary container | `auto v = makeVec() \| std::views::filter(p);` (compile error for many views — good) | Keep the container |
| Pointer to element of a growing vector | `T* p = &v[0]; v.push_back(x); *p;` | Index, `reserve`, or stable container |
| `c_str()`/`data()` after modification | `const char* p = s.c_str(); s += "x"; use(p);` | Get the pointer again |
| `unique_ptr::get()` after owner reset | Stored observer pointer | Clear observers on destruction; clear scopes |
| Pointer into `optional`/`variant` after reset/emplace | | Get it again |
| Static destruction order | Global object uses another global in its destructor | Function-local static or no globals |
| Thread uses stack data of a returned function | `std::thread t{[&] { use(local); }}; t.detach();` | Join before scope end; `jthread`; capture by value |

### 7.3 Iterator, pointer, and reference invalidation

| Container | Insert | Erase | Other |
|---|---|---|---|
| `std::vector`, `std::string` | If reallocation: **all**. Else: at and after the insert point | At and after the erase point | `reserve`/`shrink_to_fit`/`resize` beyond capacity: all |
| `std::deque` | At front/back: all **iterators**; **references stay valid**. In the middle: all | At front/back: only erased (and `end()`). In the middle: all | `shrink_to_fit`: all |
| `std::list`, `std::forward_list` | None | Only erased elements | `splice`: iterators stay valid (now refer into the other list) |
| `std::map`, `std::set`, multi versions | None | Only erased elements | — |
| `std::unordered_*` | If rehash: all **iterators**; **references stay valid** | Only erased elements | `rehash`/`reserve`: all iterators |
| `std::flat_map`/`flat_set` (C++23) | Like `std::vector` (all may be invalidated) | Like `std::vector` | — |
| `std::array` | — | — | Never invalidated (fixed) |

Rule: do not keep iterators, pointers, or references into a container across an operation that
can invalidate them. Erase while you iterate with the returned iterator: `it = c.erase(it);`, or
use `std::erase_if` (C++20).

### 7.4 Use after move

```cpp
std::vector<int> data = load();
consume(std::move(data));
report(data.size());  // compiles, but the value is unspecified: do not depend on it
data = load();        // OK: assignment gives a known state
```
Moved-from objects of your own types must be
destructible and assignable; document any other guarantee.

## 8. Low-level object construction

| Tool | Use |
|---|---|
| Placement new `::new (static_cast<void*>(p)) T(args...)` | Construct an object in existing storage |
| `std::construct_at(p, args...)` (C++20) | Same, usable in `constexpr` |
| `std::destroy_at(p)` / `p->~T()` | Destroy without freeing storage |
| `std::uninitialized_copy/move/fill/default_construct/value_construct` | Construct ranges in raw storage (exception-safe) |
| `std::destroy(first, last)` | Destroy a range |
| `std::launder(p)` (C++17) | Get a usable pointer to a new object created in storage that you access through an old pointer (for example `reinterpret_cast` from a byte buffer) |
| `std::start_lifetime_as<T>(p)` (C++23) | Start the lifetime of an implicit-lifetime object in received bytes (I/O, shared memory) without copying |
| `std::bit_cast<To>(from)` (C++20) | Reinterpret the bits of a trivially copyable value as another type (by copy) |
| `std::memcpy` | Copy trivially copyable objects / type punning before C++20 |
| `alignas(T) std::byte buf[sizeof(T)]` | Raw aligned storage (replaces deprecated `std::aligned_storage`) |

Rules:
- Every placement new has exactly one matching destruction.
- Storage must be large enough and correctly aligned for `T`.
- After placement new into a byte buffer, use the pointer returned by `new`, or `std::launder`.
- Reading an inactive `union` member is UB in C++ (unlike C). Use `std::variant` or `std::bit_cast`.

```cpp
// Fixed storage for one optional object without heap (what std::optional does internally).
template <typename T>
class InPlace
{
public:
  InPlace() noexcept = default;
  ~InPlace() { reset(); }
  InPlace(const InPlace&) = delete;
  InPlace& operator=(const InPlace&) = delete;

  template <typename... Args>
  T& emplace(Args&&... args)
  {
    reset();
    T* const p = ::new (static_cast<void*>(m_storage)) T(std::forward<Args>(args)...);
    m_hasValue = true;
    return *p;
  }
  void reset() noexcept
  {
    if (m_hasValue)
    {
      std::destroy_at(get());
      m_hasValue = false;
    }
  }
  [[nodiscard]] T* get() noexcept { return std::launder(reinterpret_cast<T*>(m_storage)); }
  [[nodiscard]] bool hasValue() const noexcept { return m_hasValue; }

private:
  alignas(T) std::byte m_storage[sizeof(T)];
  bool m_hasValue{false};
};
```
In real code use `std::optional` — this example shows the mechanics.

## 9. Containers and memory

### 9.1 Memory behavior

| Container | Layout | Overhead per element (64-bit, typical) | Allocation pattern |
|---|---|---|---|
| `std::array<T, N>` | Inline, contiguous | 0 | None |
| `std::vector<T>` | One contiguous heap buffer | 0 (+ unused capacity) | Geometric growth (×2 libstdc++/libc++, ×1.5 MSVC); amortized O(1) `push_back` |
| `std::string` | SSO buffer inline; heap when longer | 0 | SSO capacity: 15 chars (libstdc++, MSVC), 22 (libc++); `sizeof`: 32 / 32 / 24 |
| `std::deque<T>` | Array of fixed blocks | Small | Block allocations (512-byte blocks in libstdc++; 16-byte blocks in MSVC — one element per block for large types) |
| `std::list<T>` | Node per element | 2 pointers + allocator header (~32 bytes) | One allocation per insert |
| `std::forward_list<T>` | Node per element | 1 pointer + header | One allocation per insert |
| `std::map`/`std::set` | Red-black tree nodes | 3 pointers + color + header (~40-48 bytes) | One allocation per insert |
| `std::unordered_map` | Nodes + bucket array | Node: 1 pointer (+ cached hash) + header; buckets: 1 pointer each | One allocation per insert + rehash allocations |
| `std::flat_map` (C++23) | Two sorted vectors | 0 | Vector growth; O(n) insert |
| `std::optional<T>` | Inline | 1 flag + padding | None |
| `std::variant<Ts...>` | Inline, size of largest + index | Index + padding | None |
| `std::function` | Small buffer (16-32 bytes) inline; heap for larger callables | — | Heap when the callable is large |

### 9.2 Rules

- `reserve(n)` when the final size is known or bounded — avoids reallocations and invalidation.
- `clear()` keeps the capacity (good for reuse in loops). To release memory:
  `std::vector<T>{}.swap(v);` or `v = {}; v.shrink_to_fit();` (shrink is a non-binding request).
- `push_back(std::move(x))` and `emplace_back(args...)` are both fine; `emplace_back` can call
  `explicit` constructors by accident.
- Growth requires `T`'s move constructor to be `noexcept`, otherwise `std::vector` copies
  (strong exception guarantee).
- Large objects in containers: store values for locality; use `std::unique_ptr<T>` elements only
  for polymorphism or address stability.
- Node-based containers fragment the heap and hurt cache behavior — use flat containers, or a pool/`pmr` resource (`07-memory-allocation.md` sections 3 and 5).
- `std::string` concatenation in loops: `reserve` + `append`, or `std::format`.
- Container choice and general library pitfalls: `12-standard-library.md`.

## 10. Global and static objects

- **Static initialization order fiasco**: the order of dynamic initialization of namespace-scope
  objects in different translation units is unspecified. A global that uses another global during
  its initialization may see it uninitialized.
- Fixes:
  - `constexpr` / `constinit` (C++20): guaranteed static (compile-time) initialization — no order problem.
  - Function-local static: initialized at first use, thread-safe (C++11):
    ```cpp
    [[nodiscard]] Registry& registry()
    {
      static Registry instance;   // constructed at first call; destroyed at exit
      return instance;
    }
    ```
  - Pass dependencies explicitly from `main` (best: no globals).
- **Destruction order fiasco**: statics are destroyed in reverse order of construction; a static
  used in another static's destructor may already be gone. Option: an intentionally leaked object
  (`static auto& instance = *new Registry;`) — document it; it is reported by leak checkers.
- Function-local statics have a guard check on every call (small cost); avoid in very hot code
  (or use `constinit`). `-fno-threadsafe-statics` removes the guard thread safety — only in
  single-threaded programs.
- `thread_local` objects with dynamic initialization have a per-access check on some platforms.
- Non-const globals are shared mutable state (CG I.2): data races, hidden coupling, hard tests.

## 11. Memory bug catalog

| Bug | Effect | Prevention |
|---|---|---|
| Leak | Memory grows until failure | RAII, no naked `new`, no `release()` without owner, no `shared_ptr` cycles |
| Double free | Heap corruption, exploits | Single owner, RAII, null after move (`std::exchange`) |
| Use after free | Random crashes, exploits | Clear ownership, `weak_ptr`, handles with generation counters |
| Dangling reference/view | Wrong data, crashes | Section 7 rules, value members |
| Buffer overflow | Corruption, exploits | `std::array`/`std::span`/containers, bounds checks, hardened library |
| Uninitialized read | Random behavior | Initialize everything |
| Mismatched new/delete | UB, corruption | Smart pointers, containers |
| Delete through base without virtual destructor | Partial destruction, leak | Virtual or protected destructor |
| Stack overflow | Crash, or silent corruption on systems without guard pages | No recursion, small frames, stack budget |
| Heap fragmentation | Allocation fails with free memory available | Pools, arenas, no run-time allocation |
| Out of memory not handled | Crash or undefined state | Static allocation; check `nothrow` results; bounded queues |
| Aliasing/alignment violation | Wrong optimization, bus fault | `std::bit_cast`, `std::memcpy`, `alignas` |
| Data race on memory | UB | Ownership per thread, mutexes, atomics |

## 12. Memory ownership checklist

- [ ] Each resource has exactly one RAII owner; no naked `new`/`delete`/`malloc`/`free`.
- [ ] Ownership is visible in the types; raw pointers and references are non-owning.
- [ ] `unique_ptr` is the default; each `shared_ptr` is justified; no cycles (`weak_ptr` for back links).
- [ ] No view (`string_view`, `span`, iterator, pointer, reference) outlives its data.
- [ ] No use after move; no use of iterators/references after invalidation.
- [ ] Temporaries are not bound to references that outlive the full expression (section 7.1).
- [ ] Placement new / manual lifetime code: alignment, size, one destruction per construction, `std::launder` where needed.
- [ ] Static objects: no order dependencies (`constinit`, function-local statics, or no globals).
- [ ] Containers: `reserve` where size is known; move constructors `noexcept`.
- [ ] Allocation policy of the context respected (see `07-memory-allocation.md`).
