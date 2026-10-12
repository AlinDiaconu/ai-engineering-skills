# 01 — Naming, layout, files, headers, includes, namespaces

## 1. Convention detection (do this first)

1. Read 3-5 files in the same component (headers and sources).
2. Write down: type names, function names, variable names, member marker
   (`m_`, `m`, trailing `_`, none), constant style, enum value style, file
   names, brace style, indentation, include guard style.
3. Follow what you find, also when it differs from the defaults below.
4. If the component has no clear convention, check the project standard. If
   none exists, use the defaults below and keep them the same in the whole
   component.
5. Never mix two conventions in one file. Never rename unrelated code.

## 2. Default naming (when no convention exists)

| Entity | Default | Example | Alternative styles seen in industry |
|---|---|---|---|
| Namespace | `lower_snake_case` | `shop::billing` | — |
| Class, struct, enum, alias, concept | `PascalCase` | `OrderRepository`, `PriceCents` | `lower_snake_case` (STL style) |
| Function, method | `camelCase`, verb first | `computeInvoiceTotal()` | `PascalCase` (Google, AUTOSAR), `snake_case` (STL) |
| Local variable, parameter | `camelCase`, noun | `orderCount` | `snake_case` |
| Private data member | `m_` + `camelCase` | `m_orderCount` | trailing `_` (`order_count_`), `m` prefix |
| Public struct field | `camelCase`, no prefix | `point.x` | `snake_case` |
| Constant (`constexpr`/`const` at namespace or class scope) | `k` + `PascalCase` | `kMaxRetries` | `kCamelCase`, `UPPER_SNAKE` (avoid: clashes with macros) |
| `enum class` value | `PascalCase` or `kPascalCase` | `State::Active` | `UPPER_SNAKE` only if the project uses it |
| Macro | `UPPER_SNAKE_CASE` with project prefix | `MYLIB_ASSERT` | — (macros only) |
| Template parameter | `PascalCase`, `T` for one simple type | `typename Allocator` | — |
| File | `lower_snake_case` | `order_repository.hpp` | match project |

### Naming rules (all conventions)

- Name what the thing **is** or **does**, not how it is implemented.
- Functions: verbs (`computeX`, `updateX`, `isX`). Values and types: nouns.
- Boolean names read as a statement: `isValid`, `hasData`, `wasReset`, `shouldRetry`.
- Put the physical unit in the name when the type does not carry it: `distanceM`, `timeoutMs`, `angleRad`. Better: use a strong type (`Meters`, `std::chrono::milliseconds`).
- No cryptic abbreviations. Use common ones the same way everywhere (`id`, `ptr`, `cfg`, `msg`).
- Do not repeat the class or namespace name in a member name (`Socket::socketRead` → `Socket::read`).
- Getters: follow the project (`count()` or `getCount()`). Boolean getters: `isX()`/`hasX()`.
- No Hungarian type prefixes (`iCount`, `strName`) unless the project mandates them (some safety
  standards require `p_` for pointers). If mandated, apply them everywhere in the component.
- No names that differ only by case or by `l`/`1`/`O`/`0`.
- No reserved identifiers: no names that start with `_` + uppercase letter, no names with `__`
  anywhere, no names that start with `_` at global scope. Do not define names in `std`.
- Do not reuse a name in a nested scope (no shadowing).

## 3. Formatting

- If the project has a formatter configuration (for example `.clang-format`), it is the
  authority. Do not format by hand against it.
- If the project has no formatting rules, use these defaults and keep them the same in the
  whole component:
  - Indentation: 2 spaces (or 4, as the project uses). No tabs.
  - Line length: 100-120 characters.
  - Braces: one style per project (Allman or attached). Always use braces for `if`, `else`,
    `for`, `while`, `do`, also for one statement.
  - One declaration per line. One statement per line.
  - No trailing whitespace. The file ends with a newline. At most one blank line in a row
    inside functions.
  - Spaces around binary operators; no space between a unary operator and its operand.
  - `*` and `&` attach to the type (`int* p`) or to the name (`int *p`): one style per project.
  - A space after `if`, `for`, `while`, `switch`; no space between a function name and `(`.
  - Close each namespace with a comment: `}  // namespace shop::billing`.
  - Short functions on one line only when they are trivial accessors.

## 4. Files

- One main class or one cohesive module per header/source pair.
- Extensions: follow the project (`.hpp`/`.cpp` or `.h`/`.cpp`). Do not mix in one component.
- File names: lowercase, descriptive, no repetition of the folder name.
- Configuration headers: one clear name pattern (for example `<module>_cfg.hpp`).
- Keep source files small (advisory: < 750 non-comment lines). Split by responsibility.
- Public headers in an `include/<project>/<component>/` tree; private headers next to the sources.

## 5. Headers

- **Self-contained**: a header compiles alone. Check: one `.cpp` that includes only that header.
  Make the header the first include in its own `.cpp` file to prove this.
- **Include guard** or `#pragma once` (follow the project; MISRA-oriented projects often require
  classic guards). Guard name: `<PROJECT>_<PATH>_<FILE>_HPP`.
- **Include what you use** (IWYU): include every header whose names you use; remove unused includes.
  Do not depend on transitive includes.
- **Forward declare** classes to cut dependencies when you only need pointers or references.
  Never forward declare names from `std`.
- No `using namespace` at namespace scope in a header. No `using` declarations that leak into users.
- No non-`inline` variable or function definitions in headers (ODR). Use `inline` variables (C++17)
  or `constexpr` (implicitly `inline` for static data members since C++17).
- At namespace scope in a header, `const`/`constexpr` variables have internal linkage. Prefer
  `inline constexpr` (C++17) so all translation units share one object. Pre-C++17: `static constexpr`
  inside a class or a function that returns the value.
- Keep implementation details out of public headers: pImpl, `detail` namespace, or a private header.
- Templates: definitions in the header or in an `.ipp`/`.tpp` file included at the end.

## 6. Include order and paths

```cpp
#include "component/this_file.hpp"   // 1. own header first (proves self-containment)

#include "component/other.hpp"       // 2. same project
#include "other_component/api.hpp"

#include <third_party/lib.hpp>       // 3. third-party libraries

#include <cstdint>                   // 4. standard library
#include <vector>
```

(Some projects use the reverse order. Follow the project.)

- Never use `..` in include paths. Use paths from the component or project root.
- Use `"..."` for project headers and `<...>` for system and third-party headers (or follow the project).
- Use C++ headers (`<cstdint>`, `<cmath>`) and `std::` names, not `<stdint.h>`.

## 7. Namespaces

- Every name is in a namespace. Layout: `<company_or_project>::<component>[::<sub>]`.
- Use nested namespace definitions (C++17): `namespace proj::comp { ... }  // namespace proj::comp`.
- Use an unnamed namespace (not `static`) for internal-linkage helpers in `.cpp` files.
- Use a `detail` namespace for implementation parts of header-only code.
- Namespace aliases in `.cpp` files or in function scope only: `namespace billing = shop::billing;`.
- Never add names to `std` (exception: allowed specializations such as `std::hash<MyType>`).

## 8. Class layout

Order inside a class (common practice; follow the project if it differs):
1. `public:` types and aliases, constants, constructors, destructor, assignment, public methods.
2. `protected:` members (rare).
3. `private:` methods, then data members.

Group data members by meaning and size (see `08-performance-optimization.md` for layout). Document
which mutex protects which data.

## 9. Macros

- Avoid macros. Use `constexpr`, `inline` functions, templates, `enum class`, and `if constexpr`.
- Macros are acceptable for: include guards, conditional compilation for platforms, assertions that
  need `__FILE__`/`__LINE__` (prefer `std::source_location` in C++20), and X-macro tables when no
  alternative exists.
- Macro rules: `UPPER_SNAKE` with a project prefix; put every parameter and the whole body in
  parentheses; wrap statement macros in `do { ... } while (false)`; never use a parameter twice when
  it can have side effects; `#undef` local helper macros.
- Conditional compilation: `#if defined(X)` and `#ifdef X`; an undefined `X` in `#if X` silently
  evaluates to 0, so check that it is defined. Indent nested directives consistently. Keep `#if` blocks small.
