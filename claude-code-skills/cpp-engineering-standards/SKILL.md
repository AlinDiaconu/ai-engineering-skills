---
name: cpp-engineering-standards
description: Unified, generic C++ coding rules and best practices (C++14 to C++26) built from the C++ Core Guidelines, MISRA C++:2023, AUTOSAR C++14, SEI CERT C++, and current industry practice. Use whenever writing, refactoring, optimizing, or designing C++ code (.h, .hh, .hpp, .hxx, .ipp, .inl, .c++, .cc, .cpp, .cxx, .ixx, .cppm) for any target: applications, libraries, servers, desktop, real-time, and safety-critical code. Covers naming and layout, types and undefined behavior, functions, classes, OOP design principles (four pillars, SOLID, GRASP, patterns, code smells), memory ownership and lifetime, allocation strategies and real-time memory, performance, error handling, concurrency, templates and modern C++, the standard library, and code documentation, and API/library design (ABI, versioning, C interop). Adapts to the code context (general, real-time/low-latency, safety-critical) and to existing project conventions. Scope is how to write code only — not testing, tooling, or standards-compliance processes. Do not use for pure C, Python, or build scripts.
---

# C++ Engineering Standards

Rules and best practices for **writing** C++ code. They come from the C++ Core Guidelines (CG),
MISRA C++:2023, AUTOSAR C++14, SEI CERT C++, and current industry practice. The skill adapts to
the code context and to the project. A mandatory project coding standard always wins.

Out of scope: testing, test frameworks, coverage, static-analysis tooling, CI, review
processes, and standards-compliance processes (deviations, certification evidence).

The rules apply to **all** C++ code you write, including examples, demos, and `main` programs —
a race or a leak in a demo teaches the wrong use of the API.

## 0. Prime directives (always apply)

1. **Correct first, then clear, then fast.** Never trade correctness for speed.
2. **No undefined behavior (UB).** Treat any possible UB as a defect that must be removed.
3. **The project wins.** Existing conventions and the mandatory project coding standard
   override the defaults in this skill. Do not mix two styles in one file.
4. **Make wrong code hard to write.** Use the type system, `const`, RAII, and compile-time
   checks before run-time checks.
5. **Explicit ownership and lifetime** for every resource and every reference.
6. **Measure before you optimize.** No performance claim without data.
7. **Design for change and for testability.** Small units, explicit dependencies, no hidden global state.
8. **Scope discipline.** Change only what the task needs. Mention other problems; do not fix them silently.

## 1. Detect the context (before you write code)

Find these facts in the build files, configuration files, and existing code. Ask the user only
when a fact changes the code and you cannot find it.

| Fact | Where to look |
|---|---|
| C++ standard version | `CMakeLists.txt` (`CMAKE_CXX_STANDARD`, `cxx_std_*`), `-std=` flags, Bazel `copts`, `compile_commands.json` |
| Compiler and target | Toolchain files, `-std`/`-m` flags, target platform |
| Code context (section 2) | Real-time threads, latency requirements, safety markers in the code and README |
| Formatting rules | `.clang-format`, `.editorconfig`, existing files |
| Naming convention | 3-5 existing files in the same component |
| Error strategy | Exceptions enabled? (`-fno-exceptions`), `Result`/`expected` types, error codes |
| Language restrictions | `-fno-rtti`, `-fno-exceptions`, forbidden libraries, project coding standard |

## 2. Code context

The same rule can be right in one context and wrong in another. Select the context first.

| Context | When | Main effects on the code |
|---|---|---|
| **GENERAL** | Desktop, server, tools, libraries without safety claims | Full modern C++ and standard library; exceptions if the project uses them; heap allowed (but not wasted) |
| **REAL-TIME / LOW-LATENCY** | Code with timing or latency limits: control loops, audio, games, trading, robotics, device software, limited-memory systems | No allocation in real-time paths (allocate at initialization); no exceptions/RTTI where not supported; bounded loops and execution time; fixed-capacity containers; small stack frames |
| **SAFETY-CRITICAL** | Code whose failure can harm people or cause major damage | REAL-TIME rules plus: no recursion, restricted language subset (MISRA-style), defensive input checks, defined safe state for each failure, low complexity, no hidden control or data flow |

Most code is GENERAL. Apply the stricter rules only to the parts of the code that need them
(for example, only the real-time thread of an application).

## 3. Load the references

Load only what the task needs.

| Topic | Reference |
|---|---|
| Naming, files, headers, includes, namespaces, formatting, macros | `references/01-naming-layout-structure.md` |
| Types, initialization, `const`, strong types, casts, integer and float rules, statements, UB catalog | `references/02-types-expressions-ub.md` |
| Function design, parameters, return values, specifiers, overloading, lambdas, callbacks, contracts, trust boundaries and input validation | `references/03-functions-interfaces.md` |
| Classes, special members, inheritance mechanics, design patterns, dependency injection | `references/04-classes-oop.md` |
| OOP design principles: four pillars (when/how/why), SOLID, GRASP, DRY/KISS/YAGNI, coupling/cohesion, component principles, architecture, code smells | `references/05-oop-design-principles.md` |
| Memory model, ownership, RAII, smart pointers, lifetime, dangling, iterator invalidation, containers, static objects | `references/06-memory-ownership-lifetime.md` |
| Allocation strategies (arena, pool, TLSF), `operator new`, allocators, `pmr`, no-allocation policy, stack, data layout, shared memory | `references/07-memory-allocation.md` |
| Performance: algorithms, data layout, cache, allocation, copies, compile-time work, branches, SIMD | `references/08-performance-optimization.md` |
| Error handling: exceptions, `std::expected`, error codes, assertions and contracts, defensive programming, safe state | `references/09-error-handling.md` |
| Threads, locks, condition variables, atomics, signal handlers | `references/10-concurrency.md` |
| Templates, concepts, CRTP, `constexpr`, features by version, ranges, coroutines, modules, removed/deprecated features | `references/11-templates-modern-cpp.md` |
| Standard library: container choice, algorithms, strings, vocabulary types (`Overloaded` helper), facilities to avoid, pitfalls | `references/12-standard-library.md` |
| Comments, Doxygen, ASD-STE100 | `references/13-documentation.md` |
| API and library design: principles, interface shapes, library headers, ABI stability, export macros, versioning, deprecation, C interoperability, compile-time cost | `references/14-api-library-design.md` |

## 4. Core rules (short form)

The references give the reasons, examples, and exceptions.

### Types and expressions
- Initialize every object at its declaration. Prefer `{}` (no narrowing).
- `const` by default. `constexpr` for compile-time values. `consteval`/`constinit` where useful (C++20).
- No C-style casts. No `reinterpret_cast` except at hardware/serialization boundaries. Never cast away `const`.
- No implicit narrowing, no signed/unsigned mix. Use fixed-width types (`std::int32_t`) when size matters.
- Check integer overflow, division by zero, shift range, and array index before the operation.
- `nullptr`, never `0` or `NULL`. `enum class`, never plain `enum` for new code.
- No magic numbers: named `constexpr` constants; units in the name or, better, in the type.
- Floating point: never compare with `==`; check NaN/Inf at inputs; use explicit suffixes (`1.0F`).

### Functions
- One function, one job. Short. Low cyclomatic complexity (default limit 10; SAFETY-CRITICAL: project limit).
- Parameters: cheap types by value; others by `const&`; sink parameters by value + `std::move`; out values by return value.
- Few parameters (≤ 4 default). Group related parameters in a struct. Use strong types for units and IDs.
- `[[nodiscard]]` on results that must not be ignored. `noexcept` when the function cannot throw.
- Never return a reference or pointer to a local. Never return `const T` by value.
- No recursion in REAL-TIME/SAFETY-CRITICAL code. No variadic C functions (`va_arg`).

### Classes and OOP design
- `struct` for passive data without invariant; `class` with private data when an invariant exists.
- Rule of Zero first. If you declare one special member, declare or `=delete` all five.
- `explicit` on single-argument constructors and conversion operators.
- Polymorphic base: public virtual destructor or protected non-virtual. Use `override`/`final`; never repeat `virtual` with `override`.
- No virtual calls in constructors/destructors. Prevent slicing: no public copy in polymorphic bases.
- Single responsibility, composition over inheritance, Liskov substitution, small client-owned interfaces, injected dependencies, no singletons or hidden globals.

### Memory and resources
- Every resource is owned by an RAII object. No naked `new`/`delete`/`malloc`/`free`.
- Raw pointers and references are non-owning. `std::unique_ptr` by default; `std::shared_ptr` only for real shared ownership.
- Prefer stack and value semantics. Avoid unnecessary heap allocation.
- Watch lifetime traps: dangling `string_view`/`span`, invalidated iterators, lambdas that capture by reference and outlive the scope, temporaries bound to reference members.
- Choose the allocation strategy from the lifetime pattern: static, arena per cycle, object pool, ring buffer; general heap only outside real-time paths.
- REAL-TIME/SAFETY-CRITICAL: no allocation after initialization; fixed capacity; small stack frames; known memory budget.

### Performance
- Choose the right algorithm and data layout first. Contiguous memory (`std::vector`, `std::array`) by default.
- `reserve()` when the size is known. Move instead of copy. Let RVO work (return locals by value, no `std::move` on return).
- Keep hot data small and together; separate hot and cold data; avoid false sharing.
- Move work to compile time (`constexpr`) and out of loops. Avoid virtual calls, allocation, locks, and I/O in hot loops.

### Error handling
- Choose one strategy per component: exceptions, `std::expected`/`Result`, or error codes. Document it.
- Never ignore an error. No empty `catch`. No `catch (...)` without rethrow or a documented reason.
- Throw by value, catch by `const&`. Destructors, `swap`, move operations, and deallocation never fail.
- SAFETY-CRITICAL: validate inputs at boundaries; define and reach a safe state for each failure.

### Concurrency
- No data races. Share immutable data or protect mutable data with one clear owner.
- RAII locks only (`std::scoped_lock`, `std::unique_lock`), always named. Lock several mutexes with one `std::scoped_lock`.
- Never call unknown code (callbacks) while you hold a lock. Keep I/O, blocking, and expensive work outside the lock. Always wait with a predicate.
- `volatile` is for memory-mapped I/O and similar special memory only, never for synchronization.
- Use default `seq_cst` atomics. Weaker orders only with a written correctness argument per operation and a real reason (measurement or a standard pattern).
- Single-producer/single-consumer structures enforce the thread roles through separate endpoint types.

### Structure and style
- Headers: self-contained, include guard or `#pragma once` (project choice), include what you use, no `using namespace` in headers.
- Never use `..` in include paths. Keep public and private headers apart.
- All names in a namespace. No non-const globals.
- Formatting follows `.clang-format` when it exists. Do not format by hand against it.
- Names follow the component convention. If none exists, use the defaults in `references/01-naming-layout-structure.md`.

### Interfaces, APIs, and trust boundaries
- Easy to use correctly, hard to misuse: strong types, options structs, `[[nodiscard]]`, ownership visible in signatures.
- Validate external input once at the boundary (size, range, format), then pass strong internal types.
- No unsafe C string/format functions; check size calculations for overflow before you copy.
- Library APIs: minimal public headers, hidden implementation (pImpl/opaque handles) where ABI must stay stable, explicit export macros, `[[deprecated]]` before removal.

### Documentation
- Comments tell **why**, constraints, invariants, units, and safety reasons — not what the code does.
- Doxygen for every public API: brief, parameters (direction, unit, range), return, errors, thread safety.
- Write comments in clear English. Use ASD-STE100 when the project requires it.

## 5. How to write the code

1. Detect the context (sections 1-2).
2. Search the codebase for existing code to reuse.
3. Confirm the requirement: inputs, outputs, ranges, units, failure modes.
4. Design the interface first: types, ownership, errors, thread safety. Then implement.
5. Apply the rules from the loaded references.
6. Before you report the code as finished, check the coding self-check below.

## 6. Coding self-check

- [ ] No possible UB: initialization, bounds, overflow, lifetime, aliasing, null.
- [ ] Every resource has an RAII owner; ownership is clear in the types.
- [ ] Every error is handled; no ignored results; fallible steps happen before data is changed (commit after success).
- [ ] Custom result types never return fabricated values on a wrong access.
- [ ] Pools and handles detect double release, wrong-pool release, and outstanding handles at destruction; recycled items are reset.
- [ ] Fallible functions do not consume move-only arguments on failure.
- [ ] C boundaries: empty buffers, length narrowing, and embedded `'\0'` handled.
- [ ] No data races; lock order is consistent; no callbacks under locks.
- [ ] Context rules hold (heap, exceptions, recursion, complexity, stack).
- [ ] Interfaces are small, strongly typed, and hard to misuse.
- [ ] No needless copies or allocations in hot paths.
- [ ] Public API documented; comments are correct after the change.
- [ ] Style and naming match the component.
