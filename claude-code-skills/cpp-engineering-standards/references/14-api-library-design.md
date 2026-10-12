# 14 — API and library design

This file covers the design of interfaces that other code uses: public classes, functions,
headers, and libraries. Function-level rules are in `03-functions-interfaces.md`; error handling
details are in `09-error-handling.md`; documentation format is in `13-documentation.md`.

An API is a contract. Code inside a component can change every day; an API that other teams or
customers use is expensive to change. Design it with more care than internal code.

## 1. API design principles

| Principle | Rule | Example |
|---|---|---|
| **Easy to use correctly, hard to misuse** | The simplest call is the correct call; wrong calls do not compile | Strong types instead of `int, int`; RAII handles instead of `open`/`close` pairs |
| **Minimal and complete** | Every operation a user needs, and nothing more | Add a function when a real user needs it; removing one later breaks users |
| **Consistency** | Same names, argument order, error style, and ownership rules everywhere | All functions take `(source, destination)`, never sometimes `(destination, source)` |
| **Least astonishment** | The API does what its name and types promise | `size()` is cheap and has no side effects; `const` functions do not change visible state |
| **Domain names** | Names come from the user's problem, not from the implementation | `cache.evict(key)`, not `cache.removeNodeFromLruList(key)` |
| **Symmetric operations** | Pairs have matching names and behavior | `open`/`close`, `begin`/`end`, `serialize`/`deserialize`, `lock`/`unlock` (but prefer RAII) |
| **Progressive disclosure** | Simple use is simple; advanced control is available but not required | `client.get(url)` works with defaults; `client.get(url, options)` for control |
| **No surprises in cost** | Expensive operations look expensive | `computeHistogram()`, not a getter `histogram()` that scans 10 million pixels |
| **Stable concepts** | Expose concepts that will not change; hide decisions that may change | Expose `Image`; hide the pixel storage format |

### Easy to use correctly — example

```cpp
// BAD: two adjacent ints, a bool flag, and a raw owning pointer.
//   Image* resize(Image* img, int w, int h, bool keep);

// GOOD: strong types, an enum, value semantics, a result that must be used.
struct Width { std::uint32_t value; };
struct Height { std::uint32_t value; };
enum class AspectRatio : std::uint8_t { Keep, Ignore };

class Image
{
public:
  Image(Width width, Height height) : m_width{width.value}, m_height{height.value},
                                      m_pixels(static_cast<std::size_t>(width.value) * height.value) {}
  [[nodiscard]] std::uint32_t width() const noexcept { return m_width; }
  [[nodiscard]] std::uint32_t height() const noexcept { return m_height; }
private:
  std::uint32_t m_width;
  std::uint32_t m_height;
  std::vector<std::uint32_t> m_pixels;
};

[[nodiscard]] Image resize(const Image& source, Width width, Height height, AspectRatio mode);
// resize(img, Height{480}, Width{640}, ...) does not compile: the argument order error is caught.
```

## 2. Interface shapes

### 2.1 Free function or member function?

| Use a member function when | Use a free (non-member) function when |
|---|---|
| It needs access to private data | It can work through the public interface |
| It changes the object's invariant | It is an algorithm over one or more objects |
| It is a virtual customization point | It is symmetric in its operands (`operator==`, `operator+`, `swap`) |
| It is part of the core concept | It is a convenience built on top of the core |

Free functions in the same namespace are found by argument-dependent lookup (ADL) and keep classes
small. A class with 50 member functions is usually a class with 10 member functions and 40 helpers.

### 2.2 Types at the boundary

- Use **value types** (copyable, comparable, no hidden sharing) for data that crosses the API:
  configuration, results, identifiers, small records.
- Use **strong types** for units, identifiers, and indexes (see `02-types-expressions-ub.md`).
- Use **views** for read-only inputs: `std::string_view`, `std::span<const T>`. The caller keeps
  ownership; the function must not store the view.
- Use **owning types** for stored inputs and for outputs: `std::string`, `std::vector<T>`, the
  library's own value types.
- Do not expose implementation types (internal containers, third-party types, platform handles)
  in public signatures. They tie every user to your implementation.
- Do not expose `std::shared_ptr` unless shared ownership is part of the concept.

### 2.3 Options structs instead of long parameter lists

```cpp
struct RequestOptions
{
  std::chrono::milliseconds timeout{std::chrono::seconds{30}};
  std::uint8_t maxRedirects{5U};
  bool verifyCertificate{true};
  std::string userAgent{"example-http/1.0"};
};

class HttpClient
{
public:
  [[nodiscard]] Response get(std::string_view url, const RequestOptions& options = {});
};

// Call sites stay readable; new options do not break existing calls.
//   client.get(url);
//   client.get(url, {.timeout = std::chrono::seconds{5}, .verifyCertificate = false});
```
Rules: every field has a safe default; field order is stable (designated initializers must follow
declaration order); validate the options in the function and report invalid values.

### 2.4 Builder — only when needed

Use a builder only when fields depend on each other, need validation in steps, or when the
object is immutable and has many parts. Otherwise an options struct is simpler. A builder's
`build()` returns `std::expected<T, Error>` (or throws) when validation can fail.

### 2.5 Callbacks

| Need | API parameter type |
|---|---|
| Call during the function only | Template parameter (`std::invocable<Event> auto&& f`) or `std::function_ref` (C++26) |
| Store the callback | `std::function<Sig>` (copyable) or `std::move_only_function<Sig>` (C++23) |
| Several related callbacks, or a stable ABI | Abstract interface (`class Listener { virtual void onData(...) = 0; ... }`) |
| C interoperability | Function pointer + `void* userData` |

State in the documentation: on which thread the callback runs, whether it may call back into the
API (re-entrancy), and how to unregister it (prefer an RAII subscription handle).

### 2.6 Ownership in signatures

| Signature | Meaning to the caller |
|---|---|
| `void add(Item item)` | Copies or moves the value in; the caller keeps nothing |
| `void add(std::unique_ptr<Item> item)` | The library takes ownership |
| `Item& get(Id id)` | The library owns; the reference is valid while the object exists (document until when) |
| `const Item* find(Id id) const` | May return null; non-owning |
| `std::unique_ptr<Item> release(Id id)` | The caller takes ownership |
| `std::span<const Item> items() const` | View; invalid after the next mutating call (document it) |

### 2.7 Errors at API level

Choose one error mechanism for the whole library and use it everywhere: exceptions,
`std::expected<T, Error>`, or error codes. A library used by projects that disable exceptions
must not throw. Details: `09-error-handling.md`.

### 2.8 Attributes and specifiers on public APIs

- `[[nodiscard]]` on functions whose result must be used, and on error/result types.
- `noexcept` only when you promise never to throw **forever** — removing it later breaks users
  who depend on it (for example in move operations).
- `explicit` on single-argument constructors and conversion operators.
- `const` on every member function that does not change the visible state.
- `[[deprecated("reason")]]` before you remove anything (section 5).

## 3. Header design for libraries

- **Public headers** (installed, under `include/<library>/`) contain only the API. **Private
  headers** (next to the sources) contain the implementation details. Users never include
  private headers.
- Include only what the public declarations need. Every include in a public header adds compile
  time and coupling to every user.
- Forward declare your own types when the header only needs pointers or references to them.
  Never forward declare names from `std` or from third-party libraries.
- Put implementation helpers that must be in headers (templates, inline code) in a `detail`
  namespace. Users must not use `detail` names; you may change them at any time.
- Use a project prefix for macros (`IMGLIB_`), and define as few macros as possible in public headers.
- Every public header compiles alone and does not depend on include order.

### Header-only vs compiled library

| Aspect | Header-only | Compiled (static/shared) |
|---|---|---|
| Ease of use | Copy and include | Build and link |
| Compile time for users | Higher (all code in every TU) | Lower |
| Optimization | Full inlining across the boundary | Inlining only with LTO |
| ABI | No binary boundary (every user recompiles) | Binary boundary; ABI must be managed for shared libraries |
| ODR risk | Higher: different macro settings or versions in different TUs | Lower |
| Implementation hiding | None | Good (pImpl, private headers) |
| Good for | Small utilities, templates | Large libraries, stable binary distribution |

### Inline namespaces for versioning

```cpp
namespace imglib
{
  inline namespace v2
  {
    struct Pixel { std::uint8_t r, g, b, a; };   // users write imglib::Pixel
  }
  namespace v1
  {
    struct Pixel { std::uint8_t r, g, b; };      // old layout stays available as imglib::v1::Pixel
  }
}
```
The mangled symbol names contain `v2`. A program that mixes objects built against v1 and v2 of the
library fails at link time instead of crashing at run time.

## 4. ABI (Application Binary Interface)

ABI compatibility means: code compiled against version N of a shared library still works with
version N+1 **without recompilation**. API compatibility only means that it still compiles.

### 4.1 What breaks ABI

| Change | Why |
|---|---|
| Add, remove, reorder, or change the type of a data member of an exposed class | Size and offsets change; user code has the old layout inlined |
| Add, remove, or reorder virtual functions | Vtable slots change |
| Change a base class, or add virtual inheritance | Layout and `this` adjustments change |
| Change an inline function or a template in a public header | Old copies are already compiled into user binaries (ODR mismatch) |
| Change default arguments | They are compiled into the call site; old binaries keep the old value |
| Change an enum's underlying type or the values of existing enumerators | Size and meaning change |
| Change a function signature (parameter types, `const`, `noexcept` in C++17+, return type) | Mangled name or calling convention changes |
| Change thrown exception types | Old `catch` handlers no longer match |
| Pass standard library types across the boundary when user and library use different compilers, standard library versions, or build modes | `std::string`, `std::vector`, etc. have different layouts (for example the old/new libstdc++ `std::string` ABI, MSVC debug vs release iterators) |
| Change alignment or packing settings | Layout changes |

Safe changes: add a new non-virtual function; add a new class; add a new overload (if no call
becomes ambiguous); change the body of a non-inline function; add new enumerators at the end
(if users do not depend on the count).

### 4.2 Techniques for a stable ABI

| Technique | How | Cost |
|---|---|---|
| pImpl | Public class holds only `std::unique_ptr<Impl>`; all data in `Impl` in the `.cpp` | One allocation, one indirection |
| Opaque handle (C style) | `struct kv_store;` declared but never defined in the header; functions take `kv_store*` | No inline access to data |
| C ABI wrapper | `extern "C"` functions with C types only; C++ wrapper header on top (compiled into the user) | Manual wrapper code |
| Abstract interface + factory | Header exposes only an interface with virtual functions and a factory function; never add virtual functions to a published interface — add a new interface (`IStoreV2 : IStore`) | Virtual calls; interface versioning |
| Inline namespaces | Version in the mangled names (section 3) | Discipline in releases |
| Symbol visibility | Export only the API; hide everything else (section 4.4) | Export macro on API declarations |

### 4.3 pImpl example

```cpp
// --- public header: http_client.hpp ---
class HttpClient
{
public:
  explicit HttpClient(std::string baseUrl);
  ~HttpClient();                                   // defined in the .cpp, where Impl is complete
  HttpClient(HttpClient&&) noexcept;
  HttpClient& operator=(HttpClient&&) noexcept;
  HttpClient(const HttpClient&) = delete;
  HttpClient& operator=(const HttpClient&) = delete;

  [[nodiscard]] std::string get(std::string_view path) const;

private:
  struct Impl;                                     // layout can change without breaking ABI
  std::unique_ptr<Impl> m_impl;
};

// --- source file: http_client.cpp ---
struct HttpClient::Impl
{
  std::string baseUrl;
  int connectionPoolSize{4};                       // new members do not change sizeof(HttpClient)
};

HttpClient::HttpClient(std::string baseUrl) : m_impl{std::make_unique<Impl>()}
{
  m_impl->baseUrl = std::move(baseUrl);
}
HttpClient::~HttpClient() = default;
HttpClient::HttpClient(HttpClient&&) noexcept = default;
HttpClient& HttpClient::operator=(HttpClient&&) noexcept = default;

std::string HttpClient::get(std::string_view path) const
{
  return m_impl->baseUrl + std::string{path};      // placeholder for the real request
}
```

### 4.4 Symbol visibility and export macros

Build shared libraries with hidden visibility by default, and mark only the API as exported.
Benefits: smaller symbol tables, faster loading, no accidental API, fewer symbol clashes.

```cpp
// --- imglib/export.hpp ---
#if defined(_WIN32)
  #if defined(IMGLIB_BUILDING)                     // defined only when the library itself is built
    #define IMGLIB_API __declspec(dllexport)
  #else
    #define IMGLIB_API __declspec(dllimport)
  #endif
#elif defined(__GNUC__)
  #define IMGLIB_API __attribute__((visibility("default")))
#else
  #define IMGLIB_API
#endif

// --- imglib/image_io.hpp ---
namespace imglib
{
  class IMGLIB_API Decoder                         // exported class
  {
  public:
    [[nodiscard]] bool canDecode(std::string_view fileName) const;
  };

  IMGLIB_API std::uint32_t version() noexcept;    // exported function
}
```
Templates and inline functions do not need the macro; they are compiled into the user.

## 5. Versioning and evolution

Semantic versioning for C++ libraries (MAJOR.MINOR.PATCH):

| Change | API | ABI | Version |
|---|---|---|---|
| Bug fix in a non-inline function body | Compatible | Compatible | PATCH |
| New function, class, or overload | Compatible | Compatible | MINOR |
| New data member in a pImpl class | Compatible | Compatible | MINOR |
| New data member in an exposed class, new virtual function, changed inline code | Compatible | **Breaks** | MAJOR (for shared libraries) |
| Removed or renamed function, changed signature | **Breaks** | **Breaks** | MAJOR |

### Deprecation before removal

```cpp
namespace kvstore
{
  class Store
  {
  public:
    [[deprecated("Use put(key, value, PutOptions) instead; removal planned for v4.0")]]
    void insert(std::string_view key, std::string_view value) { put(key, value, {}); }

    struct PutOptions { bool overwrite{true}; };
    void put(std::string_view key, std::string_view value, const PutOptions& options);
  };
}
```
Keep a deprecated function for at least one MINOR release; say what to use instead and when it
goes away.

### Evolving without breaking

- **New parameter**: add an overload or a field in the options struct; do not add a parameter
  with a default value to an exported function (default arguments are compiled into callers and
  the signature changes).
- **New behavior of an existing function**: add a new function or an option; do not silently
  change the meaning of the old one.
- **Enums**: add new enumerators at the end; never renumber; tell users to handle unknown values
  (`default:` in a `switch` over a library enum from another version).
- **Interfaces with virtual functions**: never add a virtual function to a published interface
  in a shared library; derive a new interface (`IStoreV2`) and let the factory return it.
- **Structs passed by value across the ABI**: never add fields. Use pImpl, or reserve a version
  field and a size field (`std::uint32_t structSize;`) like many C APIs.

## 6. C interoperability

Use a C boundary for stable binary APIs, for use from other languages (Python, C#, Rust, Go),
and for plug-ins.

Rules at the `extern "C"` boundary:
- Only C types: fundamental types, pointers, C structs with standard layout, enums with fixed values.
  No references, classes with constructors, templates, overloads, or `std` types.
- No exception may cross the boundary: catch everything inside and convert it to an error code.
- Explicit ownership: one create function and one destroy function per object type; the
  library frees what the library allocates.
- Report errors by return code; give a function to get a message for an error code.
- Buffers: an empty `std::span`/`std::vector` can have `data() == nullptr`. Many C APIs reject a
  null pointer even with length 0. Handle the empty case before the call (return early), or pass a
  valid dummy pointer when the API requires one. Check that the length fits the C parameter type
  (`int`, `size_t`) before you narrow it.
- Strings: `c_str()` ends at the first embedded `'\0'`. Reject (do not truncate) a `std::string` or
  `std::string_view` that contains `'\0'` before you pass it to a C API; a `std::string_view` is not
  NUL-terminated at all — copy it into a `std::string` first.
- Document the thread-safety of every function.

```cpp
// --- kvstore_c.h (C header, usable from C and C++) ---
extern "C"
{
  typedef struct kv_store kv_store;                // opaque: never defined in the header
  typedef enum { KV_OK = 0, KV_NOT_FOUND = 1, KV_INVALID_ARGUMENT = 2, KV_INTERNAL_ERROR = 3 } kv_status;

  kv_store* kv_create(void);
  void kv_destroy(kv_store* store);                // accepts NULL
  kv_status kv_put(kv_store* store, const char* key, const char* value);
  kv_status kv_get(const kv_store* store, const char* key, char* buffer, size_t bufferSize);
}

// --- kvstore_c.cpp (C++ implementation) ---
struct kv_store
{
  std::map<std::string, std::string, std::less<>> data;
};

extern "C" kv_store* kv_create(void)
{
  return new (std::nothrow) kv_store{};            // owning pointer handed to C by design
}

extern "C" void kv_destroy(kv_store* store)
{
  delete store;
}

extern "C" kv_status kv_put(kv_store* store, const char* key, const char* value)
{
  if ((store == nullptr) || (key == nullptr) || (value == nullptr))
  {
    return KV_INVALID_ARGUMENT;
  }
  try
  {
    store->data.insert_or_assign(std::string{key}, std::string{value});
    return KV_OK;
  }
  catch (...)                                      // no exception may reach C code
  {
    return KV_INTERNAL_ERROR;
  }
}

extern "C" kv_status kv_get(const kv_store* store, const char* key, char* buffer, size_t bufferSize)
{
  if ((store == nullptr) || (key == nullptr) || (buffer == nullptr) || (bufferSize == 0U))
  {
    return KV_INVALID_ARGUMENT;
  }
  const auto it = store->data.find(std::string_view{key});
  if (it == store->data.end())
  {
    return KV_NOT_FOUND;
  }
  if (it->second.size() >= bufferSize)
  {
    return KV_INVALID_ARGUMENT;                    // buffer too small; no truncation
  }
  std::memcpy(buffer, it->second.c_str(), it->second.size() + 1U);
  return KV_OK;
}
```

On the C++ side, wrap a C API in an RAII type before you use it (see `06-memory-ownership-lifetime.md`):

```cpp
struct KvStoreDeleter
{
  void operator()(kv_store* store) const noexcept { kv_destroy(store); }
};
using KvStorePtr = std::unique_ptr<kv_store, KvStoreDeleter>;

[[nodiscard]] inline KvStorePtr makeKvStore() { return KvStorePtr{kv_create()}; }
```

## 7. Contracts in the API documentation

Every public function and class documents (format: `13-documentation.md`):

| Item | Example wording |
|---|---|
| Preconditions | "`key` must not be empty." |
| Postconditions / result | "Returns the value, or `Error::NotFound`." |
| Errors | "Throws `IoError` when the file cannot be read." / "Returns `KV_INVALID_ARGUMENT` when ..." |
| Ownership | "The caller owns the returned object." / "The store keeps a copy of `value`." |
| Lifetime of returned references/views | "The span is valid until the next call to `put()` or `erase()`." |
| Thread safety | "Thread-safe." / "Const functions may run concurrently; other functions need external synchronization." |
| Re-entrancy of callbacks | "The callback runs on the network thread. It must not call `close()`." |
| Complexity / cost | "O(log n)." / "Blocks until the response arrives or the timeout expires." |
| Units | "Timeout in milliseconds." (better: a `std::chrono` type) |

## 8. Compile-time cost of an API

- Every line in a public header is compiled by every user, in every translation unit that
  includes it. Keep public headers small.
- Move implementation out of headers when it is not a template and not performance-critical
  (pImpl also removes private members and their includes from the header).
- Templates: keep the template thin and move type-independent code into non-template functions
  in the `.cpp` file.
- Explicit instantiation for the common template arguments, so users do not instantiate them again:

```cpp
// --- matrix.hpp ---
template <typename T>
class Matrix
{
public:
  Matrix(std::size_t rows, std::size_t cols) : m_rows{rows}, m_cols{cols}, m_data(rows * cols) {}
  [[nodiscard]] T trace() const
  {
    T sum{};
    for (std::size_t i = 0U; (i < m_rows) && (i < m_cols); ++i)
    {
      sum += m_data[(i * m_cols) + i];
    }
    return sum;
  }
private:
  std::size_t m_rows;
  std::size_t m_cols;
  std::vector<T> m_data;
};

extern template class Matrix<float>;               // users do not instantiate Matrix<float>
extern template class Matrix<double>;

// --- matrix.cpp ---
template class Matrix<float>;                      // instantiated once, here
template class Matrix<double>;
```

## 9. API design checklist

- [ ] The simplest call is the correct call; wrong calls (argument order, units, ownership) do not compile.
- [ ] The API is minimal and complete; names come from the domain and are consistent.
- [ ] Inputs: views or values; outputs: owning values; ownership is clear in every signature.
- [ ] No implementation, third-party, or platform types in public signatures.
- [ ] Options structs (with safe defaults) instead of long parameter lists and `bool` flags.
- [ ] One error mechanism for the whole library; error types are `[[nodiscard]]`.
- [ ] `noexcept`, `explicit`, `const`, and `[[nodiscard]]` used deliberately.
- [ ] Callbacks: thread, re-entrancy, and unregistration documented.
- [ ] Public headers: minimal includes, self-contained, `detail` for internals, prefixed macros.
- [ ] Shared libraries: ABI strategy chosen (pImpl, opaque handles, C ABI, interfaces); symbols hidden by default; export macro on the API.
- [ ] Evolution: no ABI-breaking changes in MINOR releases; deprecation before removal; enums extended only at the end.
- [ ] C boundary: C types only, no exceptions across it, create/destroy pairs, error codes.
- [ ] Contracts documented: preconditions, errors, ownership, lifetime of views, thread safety, cost.
