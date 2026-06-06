# Intermediate C++ — A Practical Guide

**Audience:** You can write working programs, understand loops/classes, and compile small projects — but you want stronger fundamentals, fewer mystery errors, and code that looks like real C++ (not “C with classes”).

**Goal:** By the end, you should comfortably read and write modern C++ (C++17/C++20), use the STL well, understand ownership and linking, and debug typical compiler/linker issues.

---

## Table of Contents

1. [What “Intermediate” Means](#1-what-intermediate-means)
2. [Tooling You Must Know](#2-tooling-you-must-know)
3. [The Compilation Model](#3-the-compilation-model)
4. [Ownership & RAII](#4-ownership--raii)
5. [Modern C++ Essentials](#5-modern-c-essentials)
6. [The STL (Use It, Don’t Reinvent It)](#6-the-stl-use-it-dont-reinvent-it)
7. [Functions, Lambdas & Algorithms](#7-functions-lambdas--algorithms)
8. [Templates (Intermediate Level)](#8-templates-intermediate-level)
9. [Const, References & Value Categories](#9-const-references--value-categories)
10. [Error Handling](#10-error-handling)
11. [Headers, ODR & Project Structure](#11-headers-odr--project-structure)
12. [Debugging Common Errors](#12-debugging-common-errors)
13. [Patterns You’ll See in Real Codebases](#13-patterns-youll-see-in-real-codebases)
14. [Study Plan (8–12 Weeks)](#14-study-plan-812-weeks)
15. [Checklist: Am I Intermediate Yet?](#15-checklist-am-i-intermediate-yet)

---

## 1. What “Intermediate” Means

| Beginner | Intermediate |
|----------|----------------|
| Writes code in one `.cpp` file | Splits code across `.h` / `.cpp` with clean interfaces |
| Uses `new`/`delete` manually | Uses RAII, smart pointers, containers |
| Copies strings/vectors everywhere | Understands references, `const`, move |
| Fights the compiler | Reads error messages and fixes root cause |
| `using namespace std;` in headers | Qualified names, minimal includes |
| “It works on my machine” | Understands compile vs link vs runtime |

You do **not** need to be a template metaprogramming expert to be intermediate. You **do** need solid ownership, STL, and build basics.

---

## 2. Tooling You Must Know

### Compiler & standard

```bash
g++ -std=c++20 -Wall -Wextra -g main.cpp -o app
```

| Flag | Purpose |
|------|---------|
| `-std=c++20` | Enable modern language features |
| `-Wall -Wextra` | Catch common bugs |
| `-g` | Debug symbols (GDB, lldb) |
| `-Ipath` | Extra header search paths |
| `-c` | Compile to `.o` without linking |

### Build systems

- **Small projects:** Makefile or a simple script.
- **Larger projects:** CMake (industry standard).

**Intermediate skill:** Add a new `.cpp` to the build when you add functions used from `main` — otherwise you get `undefined reference` at link time.

### Debuggers

```bash
gdb ./app
# break main
# run
# next / step
# print variable
```

Learn to distinguish:

- **Compile error** — syntax/types (fix before running)
- **Link error** — missing object file or wrong signature
- **Runtime error** — logic, nullptr, UB

---

## 3. The Compilation Model

C++ builds in **translation units** (usually one `.cpp` + its includes).

```
source.cpp  →  [compiler]  →  source.o
other.cpp   →  [compiler]  →  other.o
                    ↓
              [linker]  →  executable
```

### Declarations vs definitions

**Header (`.h`)** — tells the compiler *something exists*:

```cpp
#pragma once
#include <string>
#include <vector>

std::vector<std::string> parseLines(const std::string& path);
```

**Source (`.cpp`)** — *implements* it:

```cpp
#include "parser.h"

std::vector<std::string> parseLines(const std::string& path) {
    // ...
}
```

**Rule:** Every function used across files needs a **declaration** in a header and **one definition** in exactly one `.cpp`.

### Why `undefined reference` happens

`main.cpp` calls `foo()` → compiler OK (sees declaration in `foo.h`) → linker fails because `foo.cpp` was never compiled/linked.

**Fix:** Add `foo.cpp` to your Makefile `SRCS` list.

---

## 4. Ownership & RAII

**RAII** (Resource Acquisition Is Initialization): bind resource lifetime to object lifetime.

```cpp
{
    std::ifstream file("data.txt");
    // file opens here
}   // file closes automatically — destructor runs
```

### Smart pointers (prefer over raw `new`/`delete`)

```cpp
#include <memory>

auto p = std::make_unique<int>(42);   // sole owner
auto sp = std::make_shared<Foo>();    // shared ownership (use sparingly)
```

| Type | Use when |
|------|----------|
| `std::unique_ptr<T>` | One clear owner |
| `std::shared_ptr<T>` | Shared lifetime (graphs, caches) |
| Raw pointer `T*` | Non-owning observer only |

### Intermediate rule

If you’re unsure who deletes memory, you’re not ready for raw owning pointers.

---

## 5. Modern C++ Essentials

### `auto` — let the compiler deduce type

```cpp
auto x = 42;                    // int
auto s = std::string("hi");     // std::string
for (const auto& item : vec)    // prefer const ref in range-for
```

Use `auto` when the type is obvious or long (iterators, lambdas).

### Range-based for

```cpp
for (const auto& line : lines) {
    // read-only, no copy
}
```

### `nullptr` not `NULL`

```cpp
int* p = nullptr;
```

### `enum class` (scoped enums)

```cpp
enum class Status { Ok, Error };
Status s = Status::Ok;
```

### `std::optional` — value or nothing

```cpp
#include <optional>

std::optional<int> parseInt(const std::string& s) {
    try {
        return std::stoi(s);
    } catch (...) {
        return std::nullopt;
    }
}
```

### `std::string_view` — non-owning string slice (C++17)

```cpp
void work(std::string_view sv);  // accepts string, literal, substring — no copy
```

---

## 6. The STL (Use It, Don’t Reinvent It)

### Containers — pick the right one

| Container | When |
|-----------|------|
| `std::vector<T>` | Default dynamic array; cache-friendly |
| `std::string` | Text |
| `std::unordered_map<K,V>` | Fast lookup, order irrelevant |
| `std::map<K,V>` | Sorted keys |
| `std::set<T>` | Unique sorted values |
| `std::array<T,N>` | Fixed size, stack |

### Vector growth pattern

```cpp
std::vector<std::string> v;
v.reserve(100);           // optional: avoid reallocations
v.push_back("item");
```

### Map / find idioms

```cpp
if (auto it = m.find(key); it != m.end()) {
    use(it->second);
}
```

### Don’t confuse `vector` with a function

```cpp
// WRONG — declares a vector, doesn't call the function
std::vector<std::string> results = myFunction;

// RIGHT
std::vector<std::string> results = myFunction();
```

This naming collision caused real bugs when a variable shadowed a function name.

---

## 7. Functions, Lambdas & Algorithms

### Pass by value / const reference

```cpp
void cheap(int x);                          // small types
void readBig(const std::string& s);         // read-only, no copy
void modify(std::string& s);                // mutates caller's string
void takeOwnership(std::string s);          // copy/move into function
void maybeCopy(std::string_view sv);        // read-only slice
```

### Lambdas

```cpp
auto hasSuid = [](const std::string& path) -> bool {
    struct stat st{};
    return stat(path.c_str(), &st) == 0 && (st.st_mode & S_ISUID);
};
```

Capture:

- `[&]` — reference capture (careful with lifetime)
- `[=]` — copy capture
- `[&x]` — capture specific variable

### `<algorithm>`

```cpp
#include <algorithm>

std::sort(v.begin(), v.end());
bool any = std::any_of(v.begin(), v.end(), predicate);
auto it = std::find(v.begin(), v.end(), target);
```

If you write a manual loop that “searches” or “counts,” check if an algorithm already exists.

---

## 8. Templates (Intermediate Level)

You don’t need heavy metaprogramming. Know:

### Function templates

```cpp
template<typename T>
T max(T a, T b) {
    return (a < b) ? b : a;
}
```

### Class templates

```cpp
std::vector<int> nums;
std::vector<std::string> names;
```

### When templates bite you

Error messages explode — read **the first concrete error**, not the bottom of the template stack.

**Intermediate goal:** Write a small template function (e.g. `printContainer`) and use `std::vector`, `std::list` with it.

---

## 9. Const, References & Value Categories

### `const` correctness

```cpp
class Buffer {
public:
    size_t size() const { return data_.size(); }  // won't modify object
private:
    std::vector<char> data_;
};
```

### lvalues vs rvalues (intuition)

- **lvalue** — has a name, you can take address (`int x`)
- **rvalue** — temporary (`42`, return value of function)

### Move semantics (C++11+)

```cpp
std::vector<std::string> build();
std::vector<std::string> v = build();           // may move (cheap)
v.push_back(std::move(bigString));              // explicit move
```

**Intermediate insight:** Copying large containers is expensive; moves transfer ownership without duplicating heap data.

---

## 10. Error Handling

### Exceptions — know the rules

```cpp
try {
    risky();
} catch (const std::exception& e) {
    std::cerr << e.what() << '\n';
}
```

Don’t catch everything silently unless you have a reason.

### Error codes / `std::optional`

For parsers and system calls, returning `std::optional<T>` or `bool` + output param is often clearer than exceptions.

### Initialize variables

```cpp
bool found = false;   // not: bool found;
int count = 0;
```

Uninitialized locals are a top source of “random” behavior.

---

## 11. Headers, ODR & Project Structure

### Include guards

```cpp
#pragma once
```

### What belongs in headers

- Declarations (functions, classes)
- `inline` small functions
- Template definitions (usually)

### What belongs in `.cpp`

- Function bodies that don’t need to be inline
- Anything that should not be recompiled when unrelated code changes

### Minimal includes in headers

Prefer forward declarations when possible; include what you use.

### Suggested layout (like ShadowHarvester)

```
project/
  main.cpp           # entry, orchestration
  core/              # app logic
  modules/           # reusable utilities
  tools/             # feature modules
  config/
  Makefile
```

Each `tools/foo.cpp` pairs with `tools/foo.h` when used from elsewhere.

---

## 12. Debugging Common Errors

### `no matching constructor for vector<...>`

Often: wrong type in initializer list — e.g. passing `bool` or `string` where `vector<string>` expected.

### `does not provide a call operator`

Variable shadowed a function name — you’re trying to “call” a `vector`.

### `use of undeclared identifier`

Missing `#include` or missing forward declaration.

### `undefined reference to 'foo()'`

Declaration seen, definition not linked — add `.cpp` to build.

### `invalid operands to binary expression`

Wrong types in expression — e.g. concatenating string literal with `vector`.

### Logic bugs (not compiler errors)

- `if/else` **inside** a loop that should report once **after** the loop
- Comparing whole multiline output to a single token (`env == "PATH"`)
- Forgetting to `trim()` command output before `==`

**Intermediate habit:** For each bug, classify it: compile, link, or logic.

---

## 13. Patterns You’ll See in Real Codebases

### Parser + executor split

```cpp
std::string runCommand(const char* cmd);  // executils
```

Wrap `popen`/`system` once; every tool reuses it.

### “Checker returns vector” pattern

```cpp
std::vector<std::string> findIssues() {
    if (!exposed) return {};
    return {"detail"};
}
```

Empty = safe; non-empty = hit. Used for config-based vuln checks.

### `#ifdef STANDALONE`

Same `.cpp` compiles as a small CLI tool for testing **and** links into the main binary.

### Regex scan pipeline

```cpp
std::vector<std::string> lines = loadLines();
auto hits = scanSensitiveData(lines);
```

Compose small pure functions instead of one giant `main`.

---

## 14. Study Plan (8–12 Weeks)

### Weeks 1–2: Build & ownership

- Compile multi-file projects with Makefile
- Replace `new`/`delete` with `vector`, `unique_ptr`
- Practice: split one big `.cpp` into 3 files

### Weeks 3–4: STL deep dive

- `vector`, `string`, `map`, `unordered_map`, `set`
- Iterators, `find`, `sort`, `erase` + remove idiom
- Practice: log parser, word counter, config reader

### Weeks 5–6: Modern C++11–20

- Move semantics, `auto`, lambdas, `enum class`, `optional`
- Practice: refactor copies to moves where obvious

### Weeks 7–8: Systems-ish C++

- `stat`, file I/O, reading `/proc`, `std::filesystem`
- Practice: mini “enum tools” project (like a security recon CLI)

### Weeks 9–10: Templates & generics

- Function templates, simple class templates
- Practice: generic `trim`, `split`, `readLines<T>`

### Weeks 11–12: Polish

- `-Wall -Wextra` clean build
- GDB on a segfault
- Read 2–3 medium open-source C++ files (not tutorials)

### Weekly rhythm

1. **Read** one concept (30–60 min)
2. **Code** one exercise (1–2 hrs)
3. **Break** one existing program intentionally and fix the error message

---

## 15. Checklist: Am I Intermediate Yet?

You’re in solid intermediate territory when you can honestly say yes to most of these:

- [ ] I compile projects with multiple `.cpp` files and fix linker errors without guessing
- [ ] I use `vector`, `string`, `map`, and algorithms instead of C arrays and manual loops
- [ ] I know when to pass `const&` vs by value
- [ ] I use RAII (`fstream`, `unique_ptr`) and rarely call `delete`
- [ ] I initialize locals and understand undefined behavior basics
- [ ] I can read a template error and find the real mistake
- [ ] I use `const` on methods and parameters deliberately
- [ ] I avoid `using namespace std` in headers
- [ ] I can explain what happens from source → `.o` → executable
- [ ] I can debug a logic bug in a loop (not just syntax errors)

---

## Recommended Resources

| Resource | Why |
|----------|-----|
| [cppreference.com](https://en.cppreference.com/) | Accurate standard library reference |
| *A Tour of C++* (Stroustrup) | Short modern overview |
| *Effective Modern C++* (Meyers) | Practical C++11/14 items |
| *C++ Primer* (Lippman) | Deep fundamentals if you want a textbook |
| Compiler Explorer (godbolt.org) | See what your code generates |

---

## Final Advice

Intermediate C++ is less about memorizing syntax and more about **ownership, composition, and the build model**. Write small tools, read compiler errors literally, and refactor when names collide (functions vs variables).

The fastest growth path: **one real multi-file project** maintained for weeks — not fifty disconnected 50-line snippets.

---

*Author note: This guide aligns with patterns used in multi-module C++ projects (headers + Makefile + tool modules). Adapt the study plan to your pace.*
