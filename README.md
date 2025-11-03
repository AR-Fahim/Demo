# C++ Exception Handling — Complete Guide

> A compact, practical, start-to-advanced reference for exception handling in C++: syntax, semantics, best practices, examples, edge cases, and related APIs.

---

## Quick map (what you'll find here)
- Basic syntax: `throw`, `try`, `catch`
- Standard exception types (`std::exception` family)
- Custom exceptions (how & why)
- Exception propagation & stack unwinding
- Exception safety guarantees (no-throw, basic, strong)
- `noexcept`, `std::terminate`, `std::exception_ptr`, nested exceptions
- Interop with `new` and `std::nothrow`
- Patterns: RAII, scope guards, translate-and-throw, error codes vs exceptions
- Threading and exceptions (futures, `std::exception_ptr`)
- Edge cases and gotchas (destructors, noexcept violations, slicing)
- Checklist, tips, and concise API reference

---

# 1. Basic syntax & semantics

### Throwing and catching
```cpp
#include <iostream>
#include <stdexcept>

void f() {
    throw std::runtime_error("boom");
}

int main() {
    try {
        f();
    } catch (const std::exception& e) {
        std::cerr << "caught: " << e.what() << '\n';
    }
}
```

- `throw <expression>;` creates (or rethrows) an exception object.
- `try { /* code */ } catch(type& e) { /* handler */ }` handles thrown exceptions that match the declared type.
- Catch **by reference** (`const T&`) to avoid slicing and unnecessary copies.
- The catch-list is tested **in order**. The first matching handler runs.
- `catch (...)` matches any exception (use sparingly for last-resort cleanup).

### Rethrowing
- `throw;` inside a `catch` rethrows the current exception (preserves original type and stack).
- `throw e;` (where `e` is the caught object) makes a new copy — less desirable.


# 2. Standard exception types (important ones)
- `std::exception` — base class, `virtual const char* what() const noexcept;`
- `std::bad_alloc` — memory allocation failure (`new`)
- `std::bad_exception` — used with throw specifications (rare / legacy)
- `std::runtime_error`, `std::logic_error` and their derived types:
  - `std::out_of_range`, `std::invalid_argument`, `std::domain_error`, `std::length_error`, `std::overflow_error`, `std::underflow_error`.
- I/O: `std::ios_base::failure` (streams)

Use the standard hierarchy when appropriate — they integrate with libraries and convey intent.


# 3. Custom exception classes
Prefer deriving from `std::exception` (or `std::runtime_error`) and override `what()`.

```cpp
#include <stdexcept>

class MyError : public std::runtime_error {
public:
    explicit MyError(const std::string& msg) : runtime_error(msg) {}
};
```

Guidelines:
- Make constructors `explicit` to avoid implicit conversions.
- Keep them lightweight and copyable.
- Make `what()` return a null-terminated string; reuse `std::string`'s `c_str()` via base class.


# 4. Exception propagation and stack unwinding
- When an exception is thrown and not caught in the current function, the function's stack frame is unwound: local objects are destroyed (destructors run).
- This guarantees RAII cleanup for objects with automatic storage duration.
- If a destructor throws during stack unwinding, `std::terminate()` is called (see edge cases).

Example: RAII ensures resources free even when exceptions occur.


# 5. Exception safety guarantees (levels)
Design functions with one of these guarantees:

1. **No-throw / Strong guarantee** (`noexcept`): Operation never throws. Use when failure must not propagate. E.g., destructors, `swap()` for movable types if possible.
2. **Strong guarantee**: If operation fails, program state is unchanged (transaction-like).
3. **Basic guarantee**: Invariants remain; no resource leaks, but state may be modified.
4. **No guarantee**: Nothing promised — avoid in library interfaces.

Example (strong guarantee using copy-and-swap idiom):
```cpp
void assign(MyType& dest, MyType const& src) {
    MyType tmp(src);        // may throw, leaves dest unchanged
    swap(dest, tmp);        // noexcept swap
}
```


# 6. `noexcept` and its effects
- `noexcept` marks a function as not throwing: `void f() noexcept`.
- If an exception leaves a `noexcept` function, `std::terminate()` runs.
- `noexcept(expr)` can be conditional; useful with templates.
- The compiler may optimize code assuming `noexcept` functions do not throw.
- Destructors are implicitly `noexcept` in modern C++ (unless defaulted differently).

Tip: mark move constructors/`swap` as `noexcept` to enable optimizations in containers.


# 7. `new`, `std::nothrow` and allocation failures
- Default `new` throws `std::bad_alloc` on failure.
- `new (std::nothrow) T` returns `nullptr` instead (no exception).
- Prefer throwing `new` in idiomatic C++ unless you need super-low-level handling.


# 8. Catching rules & best practices
- Catch **specific exceptions** first, then broader ones.
- Catch by `const&` unless you need to modify or rethrow by copy.
- Avoid catching `std::exception` and swallowing it — at least log/rethrow.
- Use `catch(...)` only for last-resort cleanup (or to rethrow as a typed exception).

Example order:
```cpp
try { ... }
catch (const MyError& e) { }
catch (const std::exception& e) { }
catch (...) { }
```


# 9. Exceptions in constructors and destructors
- If a constructor throws, the destructor is **not** called for that (partially constructed) object, but destructors are called for already-constructed members.
- Throwing from a destructor during stack unwinding ⇒ `std::terminate()`.
- Keep destructors `noexcept` and avoid throwing. If unavoidable, catch and swallow or call `std::terminate()` explicitly.


# 10. std::exception_ptr, nested exceptions, and rethrowing
- `std::exception_ptr` (capture current exception):
  ```cpp
  std::exception_ptr eptr;
  try { throw; } catch(...) { eptr = std::current_exception(); }
  // later: std::rethrow_exception(eptr);
  ```
- `std::nested_exception` and `std::throw_with_nested` attach nested causes. Use `std::rethrow_if_nested` to inspect.
- Useful for preserving exception information across thread boundaries or layered APIs.


# 11. Threads and exceptions
- Exceptions cannot cross thread boundaries automatically. If a thread function throws and isn’t caught, `std::terminate()` is called.
- `std::async` and `std::thread` with `std::promise`/`future` propagate exceptions via `std::exception_ptr` — call `future.get()` to rethrow in caller thread.

Example with `async`:
```cpp
auto fut = std::async(std::launch::async, []{ throw std::runtime_error("fail"); return 42; });
try { fut.get(); } catch (const std::exception& e) { /* handle */ }
```


# 12. Performance and when not to use exceptions
- Throwing is relatively expensive; code that throws rarely is fine.
- Avoid using exceptions for normal control flow (e.g., in tight loops).
- For performance-critical or low-level code (embedded systems), prefer error codes.


# 13. Interoperability: C API, noexcept, and cross-language
- If C functions return error codes, wrap them and translate to exceptions at boundary.
- Avoid letting exceptions cross C ABI boundaries — wrap and convert exceptions to error codes when crossing.


# 14. Edge cases and gotchas
- **Destructor throws during stack unwinding** → `std::terminate()`.
- **Throwing in `noexcept` function** → `std::terminate()`.
- **Object slicing**: catching by concrete type causes slicing; catch by reference.
- **Order of catch clauses** matters; a base-class catch before derived-class prevents derived-handler.
- **Exceptions in `constexpr` context**: not allowed in constant evaluation.
- **Static initialization / exceptions**: if thrown during static init, `std::terminate()` may be called.


# 15. Useful idioms & patterns

### RAII (Resource Acquisition Is Initialization)
Use destructors to release resources in case of exceptions.
```cpp
struct FileHandle { FILE* f; ~FileHandle() { if (f) fclose(f); } };
```

### Scope guard (simple)
```cpp
struct ScopeGuard { std::function<void()> onExit; ~ScopeGuard(){ if(onExit) onExit(); } };
```

### Translate-and-throw
Catch low-level exceptions and rethrow a higher-level exception for API clarity.

```cpp
try { /* low-level */ }
catch (const std::exception& e) {
    throw MyAPIError(std::string("low-level: ") + e.what());
}
```

### Copy-and-swap for strong exception-safety
See section 5 for example.

### Preserve exception context across threads
Capture `std::current_exception()` and send via `std::promise`/`future` or store `exception_ptr`.


# 16. Minimal reference: functions and types
- `throw expr;` — throw expression
- `try { } catch (...) { }` — catch clauses
- `throw;` — rethrow current
- `std::exception` and derived types
- `std::bad_alloc`, `std::runtime_error`, `std::logic_error`, `std::out_of_range`, `std::invalid_argument`, `std::ios_base::failure`
- `std::nothrow` overload for `new`
- `noexcept` operator and specifier
- `std::current_exception()`, `std::rethrow_exception()`
- `std::exception_ptr`, `std::nested_exception`, `std::throw_with_nested`, `std::rethrow_if_nested`
- `std::terminate()`, `std::set_terminate()`
- `std::uncaught_exceptions()` (count of active uncaught exceptions)


# 17. Quick checklist (what to do / avoid)
**Do:**
- Catch by `const&`.
- Prefer `std::exception`-derived types.
- Use `noexcept` for move ops and swap when safe.
- Use RAII for resource management.
- Provide meaningful messages in exceptions.
- Translate exceptions at API boundaries.

**Avoid:**
- Using exceptions for normal control flow.
- Throwing from destructors during stack unwinding.
- Catching everything and swallowing errors silently.
- Letting exceptions cross C boundaries.


# 18. Compact examples (patterns)

### 1. Basic throw & catch
```cpp
try { throw std::runtime_error("err"); }
catch (const std::exception& e) { std::cerr<<e.what(); }
```

### 2. Custom exception
```cpp
struct E : std::runtime_error { explicit E(std::string s):runtime_error(std::move(s)){} };
```

### 3. Preserve exception across thread
```cpp
std::exception_ptr ep;
try { throw std::runtime_error("x"); }
catch(...) { ep = std::current_exception(); }
// send ep to another thread and rethrow there
std::rethrow_exception(ep);
```

### 4. `noexcept` example
```cpp
void f() noexcept { throw std::runtime_error("oops"); } // calls std::terminate()
```

### 5. Copy-and-swap
```cpp
void assign(Foo& a, Foo const& b) {
    Foo tmp(b);
    swap(a, tmp); // noexcept
}
```


# 19. Further reading (concept pointers you may want to study next)
- Move semantics & `noexcept` interplay (why move should be `noexcept`)
- Exception-safe container design
- `std::future`/`std::promise` and exception propagation
- Advanced diagnostics: `std::nested_exception`, stack traces (platform-specific)

---

## End notes
This document aims to be compact and practical. If you want, I can:
- produce **short runnable examples** for any section (pick one),
- create a **cheat-sheet** printable page (one-page summary), or
- explain **specific failure scenarios** (e.g., exceptions in constructors/destructors) with annotated code.

Which one would you like next?

