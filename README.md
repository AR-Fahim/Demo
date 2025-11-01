# Complete Guide: C++ Exception Handling

---

## Table of Contents

1. [Exception Handling Fundamentals](#1-exception-handling-fundamentals)
2. [Throwing Exceptions](#2-throwing-exceptions)
3. [Catching Exceptions](#3-catching-exceptions)
4. [Standard Exception Classes](#4-standard-exception-classes)
5. [Custom Exception Classes](#5-custom-exception-classes)
6. [Exception Specifications](#6-exception-specifications)
7. [Stack Unwinding](#7-stack-unwinding)
8. [Constructor and Destructor Exceptions](#8-constructor-and-destructor-exceptions)
9. [Function-Try-Blocks](#9-function-try-blocks)
10. [noexcept Specifier](#10-noexcept-specifier)
11. [Exception Safety Guarantees](#11-exception-safety-guarantees)
12. [RAII and Exception Safety](#12-raii-and-exception-safety)
13. [Advanced Topics](#13-advanced-topics)
14. [Best Practices](#14-best-practices)
15. [Performance Considerations](#15-performance-considerations)
16. [Common Mistakes](#16-common-mistakes)
17. [Quick Reference](#17-quick-reference)

---

## 1. Exception Handling Fundamentals

### 1.1 What is Exception Handling?

**Exception handling** is a mechanism to respond to runtime errors in a controlled way. It separates error-handling code from normal code, making programs more readable and maintainable.

**Key Concepts:**
- **Exception:** An object representing an error or unexpected condition
- **Throw:** Signal that an error has occurred
- **Catch:** Handle the thrown exception
- **Stack Unwinding:** Automatic cleanup during exception propagation

### 1.2 Why Exception Handling?

**Advantages:**
- ✓ Separates error handling from business logic
- ✓ Propagates errors through call stack automatically
- ✓ Cannot be ignored (unlike return codes)
- ✓ Provides detailed error information
- ✓ Enables RAII for resource management
- ✓ Type-safe error handling

**Disadvantages:**
- ✗ Runtime overhead (when exceptions thrown)
- ✗ Code size increase
- ✗ Can be misused
- ✗ Learning curve
- ✗ Debugging can be harder

### 1.3 Basic Syntax

**Three Keywords:**
```
try {
    // Code that might throw exception
    riskyOperation();
}
catch (ExceptionType& e) {
    // Handle exception
    handleError(e);
}
```

**Simple Example:**
```cpp
#include <iostream>
#include <stdexcept>
using namespace std;

int divide(int a, int b) {
    if (b == 0) {
        throw runtime_error("Division by zero!");
    }
    return a / b;
}

int main() {
    try {
        int result = divide(10, 0);
        cout << "Result: " << result << endl;
    }
    catch (const runtime_error& e) {
        cout << "Error: " << e.what() << endl;
    }
    
    cout << "Program continues..." << endl;
    return 0;
}

// Output:
// Error: Division by zero!
// Program continues...
```

### 1.4 Control Flow

**Normal Execution:**
```
main() → try block → function call → return → continue after catch
```

**Exception Thrown:**
```
main() → try block → function call → throw → 
stack unwinding → catch block → continue after catch
```

**Uncaught Exception:**
```
main() → try block → function call → throw → 
no matching catch → std::terminate() → program abort
```

---

## 2. Throwing Exceptions

### 2.1 throw Statement

**Syntax:**
```cpp
throw expression;
```

**Can Throw:**
- Built-in types: `throw 42;`, `throw "error";`
- Standard exceptions: `throw runtime_error("msg");`
- Custom classes: `throw MyException();`
- Current exception: `throw;` (re-throw)

### 2.2 Throwing Different Types

**Primitive Types (Not Recommended):**
```cpp
void func() {
    throw 404;           // int
    throw 3.14;          // double
    throw 'E';           // char
    throw "Error!";      // const char* (dangerous!)
}

try {
    func();
}
catch (int code) {
    cout << "Error code: " << code << endl;
}
catch (const char* msg) {
    cout << "Error: " << msg << endl;
}
```

**Standard Exception Classes (Recommended):**
```cpp
void validateAge(int age) {
    if (age < 0) {
        throw invalid_argument("Age cannot be negative");
    }
    if (age > 150) {
        throw out_of_range("Age too large");
    }
}

try {
    validateAge(-5);
}
catch (const invalid_argument& e) {
    cout << "Invalid: " << e.what() << endl;
}
catch (const out_of_range& e) {
    cout << "Out of range: " << e.what() << endl;
}
```

**Custom Exception Objects (Most Flexible):**
```cpp
class FileException {
    string filename;
    string message;
public:
    FileException(const string& file, const string& msg) 
        : filename(file), message(msg) {}
    
    string getFilename() const { return filename; }
    string getMessage() const { return message; }
};

void openFile(const string& name) {
    if (!fileExists(name)) {
        throw FileException(name, "File not found");
    }
}
```

### 2.3 Re-throwing Exceptions

**Re-throw Current Exception:**
```cpp
void processData() {
    try {
        riskyOperation();
    }
    catch (const exception& e) {
        // Log error
        logError(e.what());
        
        // Re-throw same exception
        throw;  // Preserves original exception type and data
    }
}
```

**Why Use throw; (Not throw e)?**
```cpp
class Base { };
class Derived : public Base { };

try {
    throw Derived();
}
catch (Base& e) {
    // throw e;     // WRONG: Slices to Base, loses Derived info
    throw;          // CORRECT: Preserves Derived type
}
```

### 2.4 Exception Objects and Lifetime

**Exception Object Creation:**
- Created when `throw` executed
- Lives until exception caught and handled
- Copied during stack unwinding (copy constructor must not throw!)

**Important:**
```cpp
void badThrow() {
    string localMsg = "Error message";
    throw &localMsg;  // WRONG: localMsg destroyed, dangling pointer!
}

void goodThrow() {
    throw runtime_error("Error message");  // CORRECT: exception object copied
}
```

### 2.5 Throwing from Functions

**Declare Potential Exceptions (Documentation):**
```cpp
// Modern C++: Use comments
// @throws invalid_argument if value is negative
// @throws out_of_range if value exceeds maximum
void setValue(int value);
```

**When to Throw:**
- ✓ Precondition violations (invalid arguments)
- ✓ Resource acquisition failures (file, memory, network)
- ✓ Invariant violations (object in invalid state)
- ✓ Unrecoverable errors
- ✗ Expected conditions (use return values)
- ✗ Performance-critical paths
- ✗ Destructors (usually)

---

## 3. Catching Exceptions

### 3.1 Basic Catch Syntax

**Single Catch:**
```cpp
try {
    operation();
}
catch (const exception& e) {
    cout << "Error: " << e.what() << endl;
}
```

**Multiple Catch Blocks:**
```cpp
try {
    riskyOperation();
}
catch (const out_of_range& e) {
    cout << "Out of range: " << e.what() << endl;
}
catch (const invalid_argument& e) {
    cout << "Invalid argument: " << e.what() << endl;
}
catch (const exception& e) {
    cout << "General error: " << e.what() << endl;
}
```

### 3.2 Catch Order (CRITICAL!)

**Catch Blocks Checked in Order:**
```cpp
// WRONG: Base class first catches everything
try {
    throw invalid_argument("test");
}
catch (const exception& e) {        // Catches all exceptions
    // This catches everything!
}
catch (const invalid_argument& e) { // Never reached!
    // Dead code
}
```

```cpp
// CORRECT: Most specific first
try {
    throw invalid_argument("test");
}
catch (const invalid_argument& e) {  // Specific
    // Catches invalid_argument
}
catch (const logic_error& e) {       // Less specific
    // Catches other logic_error types
}
catch (const exception& e) {         // Most general
    // Catches all other std::exception types
}
```

**Rule:** Order catch blocks from most specific to most general (derived to base).

### 3.3 Catch All (...)

**Catch Any Exception:**
```cpp
try {
    operation();
}
catch (const exception& e) {
    cout << "Standard exception: " << e.what() << endl;
}
catch (...) {
    cout << "Unknown exception caught!" << endl;
    // Cannot access exception object here
}
```

**When to Use:**
- Last resort catch-all
- Cleanup before re-throwing
- Logging unknown errors
- Preventing program termination

**Pattern:**
```cpp
try {
    criticalOperation();
}
catch (const exception& e) {
    handleKnownException(e);
}
catch (...) {
    handleUnknownException();
    throw;  // Re-throw after cleanup
}
```

### 3.4 Catching by Value, Reference, or Pointer

| Method | Syntax | Pros | Cons | Recommendation |
|--------|--------|------|------|----------------|
| **By Value** | `catch (Exception e)` | Simple | Slicing, extra copy | ✗ Don't use |
| **By Reference** | `catch (Exception& e)` | No slicing, no copy | None | ✓ Use non-const for modification |
| **By Const Reference** | `catch (const Exception& e)` | No slicing, no copy, safe | Cannot modify | ✓✓ **BEST** - Use this |
| **By Pointer** | `catch (Exception* e)` | Can be null | Memory management, can be null | ✗ Avoid |

**Examples:**
```cpp
// BAD: Catch by value (slicing)
try {
    throw DerivedError();
}
catch (BaseError e) {  // Slices to BaseError!
    // Lost DerivedError information
}

// GOOD: Catch by const reference
try {
    throw DerivedError();
}
catch (const BaseError& e) {  // Preserves DerivedError type
    // Can use polymorphic methods
}
```

### 3.5 Nested Try-Catch

**Try-Catch Inside Try:**
```cpp
try {
    try {
        innerRiskyOperation();
    }
    catch (const SpecificError& e) {
        // Handle specific error locally
        handleLocally(e);
    }
    
    outerRiskyOperation();
}
catch (const exception& e) {
    // Handle all other exceptions
    handleGenerally(e);
}
```

**Try-Catch Inside Catch:**
```cpp
try {
    operation();
}
catch (const exception& e) {
    try {
        // Try to recover
        recovery();
    }
    catch (const exception& recovery_error) {
        // Recovery failed
        logError("Recovery failed");
        throw;  // Re-throw original
    }
}
```

### 3.6 Conditional Re-throwing

**Re-throw Based on Condition:**
```cpp
try {
    operation();
}
catch (const NetworkError& e) {
    if (e.isRetryable()) {
        retryOperation();
    } else {
        throw;  // Re-throw if cannot handle
    }
}
```

---

## 4. Standard Exception Classes

### 4.1 Exception Hierarchy

```
std::exception (base class)
├── std::bad_alloc              // Memory allocation failed
├── std::bad_cast               // dynamic_cast failed
├── std::bad_typeid             // typeid on null pointer
├── std::bad_exception          // Unexpected exception
├── std::bad_weak_ptr           // bad weak_ptr access
├── std::bad_function_call      // Empty std::function called
│
├── std::logic_error            // Errors detectable before runtime
│   ├── std::invalid_argument       // Invalid function argument
│   ├── std::domain_error           // Math domain error
│   ├── std::length_error           // Container size exceeded
│   ├── std::out_of_range           // Index out of bounds
│   └── std::future_error           // Future/promise error
│
└── std::runtime_error          // Errors detectable at runtime
    ├── std::range_error            // Result out of range
    ├── std::overflow_error         // Arithmetic overflow
    ├── std::underflow_error        // Arithmetic underflow
    └── std::system_error           // OS/system error
        └── std::ios_base::failure      // Stream operation failed
```

### 4.2 Common Standard Exceptions

**std::exception (Base Class)**
```cpp
class exception {
public:
    virtual const char* what() const noexcept;
    // Returns error message
};
```

**Usage:**
```cpp
try {
    throw exception();
}
catch (const exception& e) {
    cout << e.what() << endl;  // "std::exception"
}
```

**std::runtime_error**
```cpp
class runtime_error : public exception {
public:
    explicit runtime_error(const string& what_arg);
    explicit runtime_error(const char* what_arg);
};
```

**Usage:**
```cpp
void processData(const string& data) {
    if (data.empty()) {
        throw runtime_error("Data cannot be empty");
    }
    // Process...
}

try {
    processData("");
}
catch (const runtime_error& e) {
    cout << "Runtime error: " << e.what() << endl;
}
```

**std::logic_error**
```cpp
class logic_error : public exception {
public:
    explicit logic_error(const string& what_arg);
    explicit logic_error(const char* what_arg);
};
```

**When to Use:**
- `logic_error`: Preventable programming errors (invariant violations)
- `runtime_error`: Unpredictable runtime conditions (I/O, network)

### 4.3 Specific Exception Types

**std::invalid_argument**
```cpp
void setAge(int age) {
    if (age < 0 || age > 150) {
        throw invalid_argument("Age must be between 0 and 150");
    }
}
```

**std::out_of_range**
```cpp
int& Array::at(int index) {
    if (index < 0 || index >= size) {
        throw out_of_range("Index out of range");
    }
    return data[index];
}
```

**std::length_error**
```cpp
void String::reserve(size_t n) {
    if (n > max_size()) {
        throw length_error("Requested size exceeds maximum");
    }
}
```

**std::bad_alloc**
```cpp
try {
    int* huge = new int[1000000000000];  // Too large
}
catch (const bad_alloc& e) {
    cout << "Memory allocation failed: " << e.what() << endl;
}
```

**std::bad_cast**
```cpp
class Base { virtual ~Base() {} };
class Derived : public Base {};

Base* b = new Base();
try {
    Derived& d = dynamic_cast<Derived&>(*b);  // Throws
}
catch (const bad_cast& e) {
    cout << "Bad cast: " << e.what() << endl;
}
```

### 4.4 Using Standard Exceptions

**Choosing the Right Exception:**

| Situation | Exception | Example |
|-----------|-----------|---------|
| Invalid function parameter | `invalid_argument` | Negative array size |
| Index out of bounds | `out_of_range` | `vec.at(100)` on size 10 vector |
| Size limit exceeded | `length_error` | Container max_size exceeded |
| Memory allocation failed | `bad_alloc` | `new` fails |
| Type conversion failed | `bad_cast` | `dynamic_cast` fails |
| Math domain error | `domain_error` | sqrt(-1) |
| Math range error | `range_error` | Result too large to represent |
| File/IO error | `ios_base::failure` or `runtime_error` | File not found |
| Generic runtime error | `runtime_error` | Network timeout |
| Generic logic error | `logic_error` | Precondition violated |

---

## 5. Custom Exception Classes

### 5.1 Why Custom Exceptions?

**Benefits:**
- ✓ Domain-specific error information
- ✓ Multiple error details (not just message)
- ✓ Catch specific exception types
- ✓ Better error context
- ✓ Type-safe error handling

### 5.2 Deriving from std::exception

**Basic Custom Exception:**
```cpp
#include <exception>
#include <string>

class MyException : public std::exception {
private:
    std::string message;
    
public:
    explicit MyException(const std::string& msg) : message(msg) {}
    
    // Override what() - MUST be noexcept!
    const char* what() const noexcept override {
        return message.c_str();
    }
};

// Usage
throw MyException("Something went wrong");
```

**CRITICAL:** `what()` must be `noexcept` to match base class!

### 5.3 Rich Exception Classes

**Exception with Multiple Fields:**
```cpp
class FileException : public std::exception {
private:
    std::string filename;
    std::string operation;
    int errorCode;
    mutable std::string fullMessage;  // Cached message
    
public:
    FileException(const std::string& file, 
                  const std::string& op, 
                  int code)
        : filename(file), operation(op), errorCode(code) {}
    
    const char* what() const noexcept override {
        fullMessage = operation + " failed for file '" + 
                      filename + "' with code " + 
                      std::to_string(errorCode);
        return fullMessage.c_str();
    }
    
    const std::string& getFilename() const { return filename; }
    const std::string& getOperation() const { return operation; }
    int getErrorCode() const { return errorCode; }
};

// Usage
try {
    throw FileException("data.txt", "open", 404);
}
catch (const FileException& e) {
    cout << "File error: " << e.what() << endl;
    cout << "Filename: " << e.getFilename() << endl;
    cout << "Error code: " << e.getErrorCode() << endl;
}
```

### 5.4 Exception Hierarchy

**Create Exception Hierarchy:**
```cpp
// Base exception for all database errors
class DatabaseException : public std::runtime_error {
public:
    using runtime_error::runtime_error;  // Inherit constructors
};

// Specific database exceptions
class ConnectionException : public DatabaseException {
    std::string host;
    int port;
public:
    ConnectionException(const std::string& h, int p, const std::string& msg)
        : DatabaseException(msg), host(h), port(p) {}
    
    const std::string& getHost() const { return host; }
    int getPort() const { return port; }
};

class QueryException : public DatabaseException {
    std::string query;
public:
    QueryException(const std::string& q, const std::string& msg)
        : DatabaseException(msg), query(q) {}
    
    const std::string& getQuery() const { return query; }
};

// Usage
try {
    executeQuery("SELECT * FROM users");
}
catch (const QueryException& e) {
    cout << "Query failed: " << e.what() << endl;
    cout << "Query: " << e.getQuery() << endl;
}
catch (const ConnectionException& e) {
    cout << "Connection failed to " << e.getHost() 
         << ":" << e.getPort() << endl;
}
catch (const DatabaseException& e) {
    cout << "Database error: " << e.what() << endl;
}
```

### 5.5 Exception Best Practices

**DO:**
```cpp
class GoodException : public std::exception {
    std::string message;
public:
    explicit GoodException(const std::string& msg) : message(msg) {}
    
    // MUST be noexcept
    const char* what() const noexcept override {
        return message.c_str();
    }
    
    // Copy/move operations should not throw
    GoodException(const GoodException&) = default;
    GoodException(GoodException&&) noexcept = default;
    
    // Destructor must not throw
    virtual ~GoodException() noexcept = default;
};
```

**DON'T:**
```cpp
class BadException : public std::exception {
    std::string message;
public:
    // WRONG: what() not noexcept
    const char* what() const override {  // Missing noexcept!
        return message.c_str();
    }
    
    // WRONG: Constructor might throw
    BadException() {
        throw std::runtime_error("Bad!");  // Never do this!
    }
    
    // WRONG: Destructor might throw
    ~BadException() {
        throw std::runtime_error("Worse!");  // Disaster!
    }
};
```

---

## 6. Exception Specifications

### 6.1 Dynamic Exception Specifications (Deprecated in C++11, Removed in C++17)

**Old Style (Don't Use):**
```cpp
// Deprecated - don't use!
void func() throw(int, std::exception);  // Can throw int or exception
void func() throw();                     // Cannot throw (like noexcept)
```

**Problems:**
- Runtime checking (overhead)
- Calls `unexpected()` if violated
- Deprecated in C++11
- Removed in C++17

### 6.2 noexcept Specification (Modern C++)

**Syntax:**
```cpp
void func() noexcept;              // Cannot throw
void func() noexcept(true);        // Cannot throw (explicit)
void func() noexcept(false);       // Can throw
void func() noexcept(condition);   // Conditionally noexcept
```

**Examples:**
```cpp
// Simple noexcept
void swap(int& a, int& b) noexcept {
    int temp = a;
    a = b;
    b = temp;
}

// Conditional noexcept
template<typename T>
void swap(T& a, T& b) noexcept(std::is_nothrow_move_constructible<T>::value 
                              && std::is_nothrow_move_assignable<T>::value) {
    T temp = std::move(a);
    a = std::move(b);
    b = std::move(temp);
}
```

### 6.3 When to Use noexcept

**MUST be noexcept:**
- Destructors (implicitly noexcept)
- Move constructors and move assignment operators
- swap functions
- Memory deallocation (delete, operator delete)

**SHOULD be noexcept:**
- Exception class copy constructors
- Exception class `what()` method
- Leaf functions (simple operations)
- Performance-critical code

**Example:**
```cpp
class String {
    char* data;
public:
    // Move constructor - should be noexcept for optimization
    String(String&& other) noexcept 
        : data(other.data) {
        other.data = nullptr;
    }
    
    // Move assignment - should be noexcept
    String& operator=(String&& other) noexcept {
        if (this != &other) {
            delete[] data;
            data = other.data;
            other.data = nullptr;
        }
        return *this;
    }
    
    // Destructor - implicitly noexcept
    ~String() {
        delete[] data;
    }
};
```

### 6.4 noexcept and std::move_if_noexcept

**Why noexcept Matters:**
```cpp
std::vector<String> vec;
vec.push_back(String("hello"));
// If move constructor is noexcept, uses move
// Otherwise, uses copy (for exception safety)
```

**std::move_if_noexcept:**
```cpp
template<typename T>
void resize(T* old_data, T* new_data, size_t size) {
    for (size_t i = 0; i < size; ++i) {
        // Uses move if noexcept, otherwise copy
        new_data[i] = std::move_if_noexcept(old_data[i]);
    }
}
```

### 6.5 Checking noexcept at Compile Time

**noexcept operator:**
```cpp
void func1() noexcept { }
void func2() { }

// noexcept operator (returns bool)
static_assert(noexcept(func1()), "func1 should be noexcept");
static_assert(!noexcept(func2()), "func2 can throw");

// Use in templates
template<typename T>
void process() noexcept(noexcept(T())) {
    T obj;  // noexcept if T's constructor is noexcept
}
```

---

## 7. Stack Unwinding

### 7.1 What is Stack Unwinding?

**Definition:** Process of exiting stack frames (function calls) and calling destructors for automatic objects when exception propagates up the call stack.

**Process:**
1. Exception thrown
2. Control transfers to matching catch block
3. All automatic objects between throw and catch are destroyed
4. Destructors called in reverse order of construction
5. Resources released automatically (if using RAII)

### 7.2 Stack Unwinding Example

```cpp
class Resource {
    string name;
public:
    Resource(const string& n) : name(n) {
        cout << "Constructing " << name << endl;
    }
    ~Resource() {
        cout << "Destroying " << name << endl;
    }
};

void func3() {
    Resource r3("Resource3");
    throw runtime_error("Error in func3");
    cout << "After throw" << endl;  // Never executed
}

void func2() {
    Resource r2("Resource2");
    func3();
    cout << "After func3" << endl;  // Never executed
}

void func1() {
    Resource r1("Resource1");
    try {
        func2();
    }
    catch (const exception& e) {
        cout << "Caught: " << e.what() << endl;
    }
}

int main() {
    func1();
    return 0;
}

/* Output:
Constructing Resource1
Constructing Resource2
Constructing Resource3
Destroying Resource3    ← Stack unwinding starts
Destroying Resource2    ← Automatic cleanup
Caught: Error in func3
Destroying Resource1    ← Normal scope exit
*/
```

### 7.3 Stack Unwinding Rules

**Automatic Objects:**
- ✓ Destructors called automatically
- ✓ Called in reverse order of construction
- ✓ RAII ensures resource cleanup

**Not Automatic:**
- ✗ Manual memory: `new` without `delete`
- ✗ C-style resources: file handles, sockets
- ✗ Manual cleanup needed

**Example of Problem:**
```cpp
void badFunction() {
    int* data = new int[1000];
    
    riskyOperation();  // Throws exception
    
    delete[] data;  // Never reached - MEMORY LEAK!
}
```

**Solution - Use RAII:**
```cpp
void goodFunction() {
    std::vector<int> data(1000);  // RAII container
    
    riskyOperation();  // Throws exception
    
    // data automatically cleaned up by destructor
}
```

### 7.4 Destructor Exceptions During Unwinding

**DANGER: Exception in Destructor During Unwinding:**
```cpp
class BadResource {
public:
    ~BadResource() {
        throw runtime_error("Destructor error");  // VERY BAD!
    }
};

void dangerous() {
    BadResource r1;
    throw runtime_error("Function error");
}

// Result: TWO active exceptions → std::terminate() → program abort!
```

**Rule:** **NEVER throw from destructors**, especially during stack unwinding.

**Safe Destructor:**
```cpp
class SafeResource {
public:
    ~SafeResource() noexcept {  // Explicitly noexcept
        try {
            cleanup();  // Might throw
        }
        catch (...) {
            // Log error, but don't propagate
            logError("Cleanup failed");
        }
    }
};
```

---

## 8. Constructor and Destructor Exceptions

### 8.1 Constructor Exceptions

**Constructors CAN Throw:**
```cpp
class File {
    FILE* handle;
public:
    File(const char* filename) {
        handle = fopen(filename, "r");
        if (!handle) {
            throw runtime_error("Cannot open file");
        }
    }
    
    ~File() {
        if (handle) fclose(handle);
    }
};
```

**Object Not Created if Constructor Throws:**
```cpp
try {
    File f("nonexistent.txt");  // Constructor throws
    // f doesn't exist here
}
catch (const exception& e) {
    // Destructor NOT called (object never fully constructed)
}
```

### 8.2 Constructor Exception Safety

**Problem: Partial Construction:**
```cpp
class Widget {
    int* data1;
    int* data2;
public:
    Widget() {
        data1 = new int[100];
        // Exception here!
        data2 = new int[100];  // Not reached if exception thrown
    }
    
    ~Widget() {
        delete[] data1;  // Never called if constructor fails!
        delete[] data2;
    }
};
```

**Solution 1: Use RAII Members:**
```cpp
class Widget {
    std::vector<int> data1;  // RAII
    std::vector<int> data2;  // RAII
public:
    Widget() : data1(100), data2(100) {
        // If exception thrown, data1 automatically cleaned up
    }
    // Destructor automatically cleans up
};
```

**Solution 2: Smart Pointers:**
```cpp
class Widget {
    std::unique_ptr<int[]> data1;
    std::unique_ptr<int[]> data2;
public:
    Widget() {
        data1 = std::make_unique<int[]>(100);
        // If exception here, data1 automatically cleaned up
        data2 = std::make_unique<int[]>(100);
    }
};
```

**Solution 3: Function-Try-Block (Constructor):**
```cpp
class Widget {
    Resource* resource;
public:
    Widget() try : resource(new Resource()) {
        // Constructor body
    }
    catch (...) {
        // Cleanup if needed
        delete resource;  // Manual cleanup
        throw;  // Must re-throw
    }
};
```

### 8.3 Destructor Exceptions

**Destructors Should NOT Throw:**
```cpp
class BadWidget {
public:
    ~BadWidget() {
        throw runtime_error("Destructor error");  // NEVER DO THIS!
    }
};
```

**Why Destructors Shouldn't Throw:**
1. During stack unwinding, destructor exception → `std::terminate()`
2. If two exceptions active → program abort
3. Cannot recover from destructor failure
4. Breaks RAII principle

**Safe Destructor Pattern:**
```cpp
class SafeWidget {
    FILE* file;
public:
    SafeWidget(const char* name) : file(fopen(name, "w")) {
        if (!file) throw runtime_error("Cannot open file");
    }
    
    // Option 1: Swallow exceptions
    ~SafeWidget() noexcept {
        if (file) {
            if (fclose(file) != 0) {
                // Log error, but don't throw
                std::cerr << "Warning: fclose failed\n";
            }
        }
    }
    
    // Option 2: Provide explicit close method
    void close() {  // Can throw
        if (file) {
            if (fclose(file) != 0) {
                file = nullptr;
                throw runtime_error("Cannot close file");
            }
            file = nullptr;
        }
    }
};

// Usage
try {
    SafeWidget w("output.txt");
    // ... work with file ...
    w.close();  // Explicit close, can handle errors
}
catch (const exception& e) {
    // Handle close error
}
// If close() not called, destructor closes silently
```

### 8.4 Member Initialization Exceptions

**Initialization List Exceptions:**
```cpp
class Container {
    std::vector<int> data;
    std::string name;
public:
    Container(size_t size, const std::string& n)
    try 
        : data(size),  // Might throw bad_alloc
          name(n)      // Might throw bad_alloc
    {
        // Constructor body
    }
    catch (const std::bad_alloc& e) {
        // Handle allocation failure
        // Note: Cannot access members here (not constructed)
        std::cerr << "Allocation failed\n";
        throw;  // Must re-throw
    }
};
```

**Key Points:**
- Exception in member initialization stops construction
- Already-constructed members automatically destroyed
- Function-try-block can catch initialization exceptions
- Must re-throw from constructor catch block

---

## 9. Function-Try-Blocks

### 9.1 What are Function-Try-Blocks?

**Function-try-blocks** wrap entire function (including initialization) in try-catch.

**Regular Function:**
```cpp
void regularFunction(int value) 
try {
    // Function body
    if (value < 0) throw invalid_argument("Negative value");
}
catch (const exception& e) {
    cout << "Error: " << e.what() << endl;
    // Can handle and continue
}
```

**Constructor Function-Try-Block:**
```cpp
class MyClass {
    int* data;
public:
    MyClass(int size) 
    try 
        : data(new int[size])  // Initializer list
    {
        // Constructor body
        if (size <= 0) throw invalid_argument("Invalid size");
    }
    catch (const std::bad_alloc& e) {
        // Catches initialization and body exceptions
        cerr << "Memory allocation failed\n";
        throw;  // MUST re-throw in constructor catch
    }
    catch (const invalid_argument& e) {
        cerr << "Invalid argument: " << e.what() << endl;
        throw;  // MUST re-throw
    }
};
```

### 9.2 Function-Try-Block Rules

**For Constructors:**
- Implicitly re-throws if no explicit throw
- MUST re-throw (cannot suppress exception)
- Reason: object not fully constructed
- Cannot return from catch block

**For Regular Functions:**
- Can handle exception and return normally
- Don't have to re-throw
- Can return value from catch block

**For Destructors:**
- Should never use (destructors shouldn't throw)
- If used, must not re-throw

### 9.3 Use Cases

**Valid Use Case:**
```cpp
class Logger {
    std::ofstream logFile;
public:
    Logger(const std::string& filename)
    try 
        : logFile(filename)
    {
        if (!logFile) throw runtime_error("Cannot open log");
        logFile << "Log started\n";
    }
    catch (const exception& e) {
        // Log to stderr instead
        std::cerr << "Logging initialization failed: " << e.what() << endl;
        throw;
    }
};
```

**When to Use:**
- Catching exceptions from member initialization
- Adding context to constructor exceptions
- Centralized exception handling for entire function

**When NOT to Use:**
- Regular exception handling (use normal try-catch)
- Destructors (never throw from destructors)
- Complex control flow (hard to read)

---

## 10. noexcept Specifier

### 10.1 noexcept Benefits

**Why Use noexcept:**
1. **Optimization:** Compiler can optimize better
2. **Move Semantics:** Enables move in containers
3. **Documentation:** Clearly states function won't throw
4. **Safety:** Calls `std::terminate()` if violated (fail-fast)

### 10.2 noexcept Examples

**Simple noexcept:**
```cpp
void swap(int& a, int& b) noexcept {
    int temp = a;
    a = b;
    b = temp;
}

// If exception thrown inside, std::terminate() called
```

**Conditional noexcept:**
```cpp
template<typename T>
class Vector {
public:
    // noexcept if T's move constructor is noexcept
    void push_back(T&& value) 
        noexcept(std::is_nothrow_move_constructible<T>::value) {
        // ... implementation
    }
};
```

**Complex Conditional:**
```cpp
template<typename T>
void swap(T& a, T& b) 
    noexcept(noexcept(T(std::move(a))) && 
             noexcept(a = std::move(b))) {
    T temp = std::move(a);
    a = std::move(b);
    b = std::move(temp);
}
```

### 10.3 noexcept and std::terminate()

**Violation Behavior:**
```cpp
void willTerminate() noexcept {
    throw runtime_error("This terminates!");  // Calls std::terminate()
}

int main() {
    try {
        willTerminate();
    }
    catch (...) {
        // Never reached!
        cout << "Caught\n";
    }
    // Program already terminated
}
```

**Setting Terminate Handler:**
```cpp
#include <exception>

void myTerminate() {
    cout << "Terminating due to noexcept violation\n";
    abort();
}

int main() {
    std::set_terminate(myTerminate);
    
    willTerminate();  // Calls myTerminate(), then aborts
}
```

### 10.4 When to Use noexcept

**ALWAYS noexcept:**
```cpp
class Widget {
public:
    // Destructors (implicitly noexcept)
    ~Widget() noexcept { }
    
    // Move operations (for optimization)
    Widget(Widget&&) noexcept { }
    Widget& operator=(Widget&&) noexcept { }
    
    // swap functions
    friend void swap(Widget& a, Widget& b) noexcept { }
    
    // Memory deallocation
    void operator delete(void* ptr) noexcept { }
};
```

**USUALLY noexcept:**
```cpp
// Simple getters
int getValue() const noexcept { return value; }

// Simple setters (no validation)
void setValue(int v) noexcept { value = v; }

// Leaf functions (no calls to throwing functions)
bool isEmpty() const noexcept { return size == 0; }
```

**DON'T Use noexcept:**
```cpp
// Functions that validate and throw
void setAge(int age) {  // NOT noexcept
    if (age < 0) throw invalid_argument("Negative age");
    this->age = age;
}

// Functions calling potentially throwing code
void process() {  // NOT noexcept
    std::vector<int> vec;  // Can throw bad_alloc
    vec.push_back(42);
}
```

### 10.5 Checking noexcept

**Compile-Time Check:**
```cpp
static_assert(noexcept(swap(a, b)), "swap must be noexcept");

// Using in templates
template<typename T>
void foo() {
    if constexpr (noexcept(T())) {
        // T's constructor is noexcept
    } else {
        // T's constructor can throw
    }
}
```

**Type Traits:**
```cpp
#include <type_traits>

// Check if type has noexcept move constructor
static_assert(std::is_nothrow_move_constructible_v<std::string>);

// Check if type has noexcept destructor
static_assert(std::is_nothrow_destructible_v<std::vector<int>>);
```

---

## 11. Exception Safety Guarantees

### 11.1 Four Levels of Exception Safety

**1. No-throw Guarantee (Strongest)**
- Operation cannot fail
- Marked `noexcept`
- Examples: destructors, swap, move operations

```cpp
void swap(int& a, int& b) noexcept {
    int temp = a;
    a = b;
    b = temp;
}
```

**2. Strong Exception Safety**
- Operation succeeds completely or has no effect
- "Commit or rollback" semantics
- State unchanged if exception thrown

```cpp
void Vector::push_back(const T& value) {
    if (size == capacity) {
        // Create new buffer
        T* newData = new T[capacity * 2];
        
        // Copy old data (if this fails, nothing changed)
        std::copy(data, data + size, newData);
        
        // Only now commit changes
        delete[] data;
        data = newData;
        capacity *= 2;
    }
    data[size++] = value;
}
```

**3. Basic Exception Safety**
- No resource leaks
- Object in valid (but unspecified) state
- Invariants maintained

```cpp
void String::append(const char* str) {
    size_t len = strlen(str);
    reserve(size + len);  // Might throw
    
    // If exception thrown here, object still valid
    // (but might have extra capacity)
    strcat(data, str);
    size += len;
}
```

**4. No Exception Safety (Weakest)**
- Resources might leak
- Object might be in invalid state
- Avoid this!

```cpp
void bad() {
    int* p = new int[100];
    riskyOperation();  // Throws, p leaks!
    delete[] p;
}
```

### 11.2 Achieving Strong Exception Safety

**Copy-and-Swap Idiom:**
```cpp
class String {
    char* data;
    size_t size;
public:
    String& operator=(const String& other) {
        // Create copy (might throw)
        String temp(other);
        
        // Swap (noexcept) - commits changes
        swap(*this, temp);
        
        // Old data destroyed (noexcept)
        return *this;
    }
    
    friend void swap(String& a, String& b) noexcept {
        using std::swap;
        swap(a.data, b.data);
        swap(a.size, b.size);
    }
};
```

**Benefits:**
- Strong exception safety
- Elegant and simple
- Reuses copy constructor

### 11.3 Exception Safety in Practice

**Design Principle:**
- Do all operations that might throw BEFORE modifying state
- Modify state only with noexcept operations
- Separate "dangerous" from "safe" operations

**Example:**
```cpp
class Database {
    std::vector<Record> records;
public:
    void addRecord(const Record& rec) {
        // Phase 1: Validate (can throw)
        validateRecord(rec);
        
        // Phase 2: Prepare (can throw)
        Record prepared = prepareRecord(rec);
        
        // Phase 3: Commit (noexcept or strong guarantee)
        records.push_back(std::move(prepared));
    }
};
```

### 11.4 Documenting Exception Safety

**In Comments:**
```cpp
class Container {
public:
    /**
     * @brief Add element to container
     * @param value Element to add
     * @throws std::bad_alloc if memory allocation fails
     * @guarantee Strong exception safety
     */
    void add(const T& value);
    
    /**
     * @brief Swap two containers
     * @param other Container to swap with
     * @throws Never throws
     * @guarantee No-throw guarantee
     */
    void swap(Container& other) noexcept;
};
```

---

## 12. RAII and Exception Safety

### 12.1 RAII Principle

**Resource Acquisition Is Initialization:**
- Acquire resource in constructor
- Release resource in destructor
- Automatic cleanup via stack unwinding

**Manual Resource Management (BAD):**
```cpp
void bad() {
    FILE* file = fopen("data.txt", "r");
    
    processData();  // Might throw
    
    fclose(file);  // Never reached if exception thrown!
}
```

**RAII Approach (GOOD):**
```cpp
class FileHandle {
    FILE* file;
public:
    FileHandle(const char* name) : file(fopen(name, "r")) {
        if (!file) throw runtime_error("Cannot open file");
    }
    
    ~FileHandle() {
        if (file) fclose(file);
    }
    
    FILE* get() { return file; }
};

void good() {
    FileHandle file("data.txt");
    
    processData();  // If throws, destructor still called!
    
    // Automatic cleanup
}
```

### 12.2 Standard RAII Classes

**Smart Pointers:**
```cpp
// unique_ptr - exclusive ownership
void example1() {
    auto ptr = std::make_unique<int>(42);
    riskyOperation();
    // Automatic cleanup, no leak
}

// shared_ptr - shared ownership
void example2() {
    auto ptr = std::make_shared<Widget>();
    riskyOperation();
    // Automatic cleanup
}
```

**Containers:**
```cpp
void example3() {
    std::vector<int> vec;
    vec.reserve(1000);
    
    riskyOperation();
    // Vector automatically cleaned up
}
```

**File Streams:**
```cpp
void example4() {
    std::ifstream file("data.txt");
    
    riskyOperation();
    // File automatically closed
}
```

**Locks:**
```cpp
void example5() {
    std::mutex mtx;
    {
        std::lock_guard<std::mutex> lock(mtx);
        
        riskyOperation();
        // Mutex automatically unlocked
    }
}
```

### 12.3 Custom RAII Classes

**Database Connection:**
```cpp
class DatabaseConnection {
    Connection* conn;
public:
    DatabaseConnection(const string& host) {
        conn = connectToDatabase(host);
        if (!conn) throw runtime_error("Connection failed");
    }
    
    ~DatabaseConnection() {
        if (conn) disconnect(conn);
    }
    
    // Delete copy, allow move
    DatabaseConnection(const DatabaseConnection&) = delete;
    DatabaseConnection& operator=(const DatabaseConnection&) = delete;
    
    DatabaseConnection(DatabaseConnection&& other) noexcept 
        : conn(other.conn) {
        other.conn = nullptr;
    }
    
    Connection* get() { return conn; }
};
```

**Temporary Directory:**
```cpp
class TempDirectory {
    std::string path;
public:
    TempDirectory() {
        path = createTempDir();
        if (path.empty()) throw runtime_error("Cannot create temp dir");
    }
    
    ~TempDirectory() {
        if (!path.empty()) {
            try {
                removeDirectory(path);
            } catch (...) {
                // Log but don't throw from destructor
                std::cerr << "Failed to remove temp dir\n";
            }
        }
    }
    
    const std::string& getPath() const { return path; }
};
```

### 12.4 Scope Guards

**Simple Scope Guard:**
```cpp
template<typename F>
class ScopeGuard {
    F cleanup;
    bool active;
public:
    ScopeGuard(F f) : cleanup(std::move(f)), active(true) {}
    
    ~ScopeGuard() {
        if (active) cleanup();
    }
    
    void dismiss() { active = false; }
    
    ScopeGuard(const ScopeGuard&) = delete;
    ScopeGuard& operator=(const ScopeGuard&) = delete;
};

template<typename F>
ScopeGuard<F> makeScopeGuard(F f) {
    return ScopeGuard<F>(std::move(f));
}

// Usage
void example() {
    int* data = new int[100];
    auto guard = makeScopeGuard([&]{ delete[] data; });
    
    riskyOperation();  // If throws, guard cleans up
    
    guard.dismiss();  // Success, don't cleanup
    useData(data);
}
```

---

## 13. Advanced Topics

### 13.1 Exception Translation

**Catch and Re-throw Different Exception:**
```cpp
void lowLevelFunction() {
    throw std::runtime_error("Low-level error");
}

void highLevelFunction() {
    try {
        lowLevelFunction();
    }
    catch (const std::runtime_error& e) {
        // Translate to domain-specific exception
        throw DatabaseException("Database operation failed: " + 
                                std::string(e.what()));
    }
}
```

### 13.2 Exception Chaining (C++11 with nested_exception)

**Store Original Exception:**
```cpp
#include <exception>

try {
    try {
        lowLevelOperation();
    }
    catch (const std::exception& e) {
        std::throw_with_nested(
            std::runtime_error("High-level error"));
    }
}
catch (const std::exception& e) {
    cout << "Current: " << e.what() << endl;
    
    try {
        std::rethrow_if_nested(e);
    }
    catch (const std::exception& nested) {
        cout << "Nested: " << nested.what() << endl;
    }
}
```

### 13.3 std::exception_ptr

**Store and Re-throw Later:**
```cpp
#include <exception>
#include <thread>

std::exception_ptr globalException = nullptr;

void workerThread() {
    try {
        riskyOperation();
    }
    catch (...) {
        globalException = std::current_exception();
    }
}

int main() {
    std::thread t(workerThread);
    t.join();
    
    if (globalException) {
        try {
            std::rethrow_exception(globalException);
        }
        catch (const std::exception& e) {
            cout << "Worker threw: " << e.what() << endl;
        }
    }
}
```

### 13.4 std::uncaught_exceptions() (C++17)

**Count Active Exceptions:**
```cpp
class Guard {
    int exceptionCount;
public:
    Guard() : exceptionCount(std::uncaught_exceptions()) {}
    
    ~Guard() {
        if (std::uncaught_exceptions() > exceptionCount) {
            // Destructor called during stack unwinding
            rollback();
        } else {
            // Normal destruction
            commit();
        }
    }
};
```

### 13.5 Custom Terminate and Unexpected Handlers

**std::set_terminate:**
```cpp
void myTerminate() {
    cout << "Terminating... performing cleanup\n";
    logFatalError();
    abort();
}

int main() {
    std::set_terminate(myTerminate);
    
    // If uncaught exception or noexcept violation
    throw runtime_error("Unhandled");  // Calls myTerminate
}
```

### 13.6 Exception in Multi-threaded Code

**Exceptions Don't Cross Thread Boundaries:**
```cpp
#include <thread>
#include <future>

// WRONG: Exception lost
void bad() {
    std::thread t([]{ 
        throw runtime_error("Error");  // Calls std::terminate!
    });
    t.join();
}

// CORRECT: Use std::async and std::future
void good() {
    auto future = std::async(std::launch::async, []{ 
        throw runtime_error("Error");
        return 42;
    });
    
    try {
        int result = future.get();  // Re-throws exception
    }
    catch (const std::exception& e) {
        cout << "Caught: " << e.what() << endl;
    }
}
```

---

## 14. Best Practices

### 14.1 When to Use Exceptions

**✓ USE Exceptions For:**
- Error conditions (not expected flow)
- Constructor failures
- Resource acquisition failures
- Invariant violations
- Errors that skip multiple stack frames
- Separating error handling from business logic

**✗ DON'T Use Exceptions For:**
- Normal control flow
- Expected conditions (validation)
- Performance-critical hot paths
- Simple functions (prefer error codes)
- Across library boundaries (C APIs)

### 14.2 Throwing Best Practices

**DO:**
```cpp
// Throw by value
throw std::runtime_error("Error message");

// Throw derived exceptions
throw DatabaseException("Connection failed");

// Include context
throw FileException("config.txt", "open", errno);
```

**DON'T:**
```cpp
// Don't throw pointers
throw new std::runtime_error("Error");  // Who deletes?

// Don't throw built-in types
throw 404;  // Not descriptive
throw "error";  // const char* issues

// Don't throw from destructors
~MyClass() {
    throw std::runtime_error("Bad!");  // Terminate!
}
```

### 14.3 Catching Best Practices

**DO:**
```cpp
// Catch by const reference
catch (const std::exception& e)

// Order specific to general
catch (const FileException& e) { }
catch (const std::exception& e) { }

// Re-throw with throw;
catch (const std::exception& e) {
    log(e);
    throw;  // Preserves exception type
}
```

**DON'T:**
```cpp
// Don't catch by value
catch (std::exception e)  // Slicing!

// Don't catch base before derived
catch (const std::exception& e) { }  // Catches everything
catch (const FileException& e) { }   // Never reached

// Don't catch and ignore
catch (...) { }  // Dangerous!
```

### 14.4 Exception Class Design

**DO:**
```cpp
class MyException : public std::exception {
public:
    // Explicit constructor
    explicit MyException(const std::string& msg);
    
    // noexcept what()
    const char* what() const noexcept override;
    
    // noexcept destructor
    virtual ~MyException() noexcept = default;
    
    // noexcept copy/move
    MyException(const MyException&) = default;
    MyException(MyException&&) noexcept = default;
};
```

### 14.5 Documentation

**Document Exceptions:**
```cpp
/**
 * @brief Opens a file for reading
 * @param filename Path to file
 * @return File handle
 * @throws FileNotFoundException if file doesn't exist
 * @throws PermissionDeniedException if no read permission
 * @throws std::bad_alloc if memory allocation fails
 */
FileHandle openFile(const std::string& filename);
```

---

## 15. Performance Considerations

### 15.1 Exception Overhead

**Zero-Cost When Not Thrown:**
- Modern C++ uses "zero-cost" exception handling
- No runtime cost in normal path (no try-catch overhead)
- Cost only when exception thrown

**Cost When Thrown:**
- Stack unwinding
- Destructor calls
- Exception object construction
- Catch block lookup
- **Significant performance impact**

### 15.2 Performance Tips

**Minimize Exception Throwing:**
```cpp
// SLOW: Throws in loop
for (int i = 0; i < 1000000; ++i) {
    try {
        if (i % 2 == 0) throw runtime_error("Even");
    }
    catch (...) { }
}

// FAST: Avoid exceptions
for (int i = 0; i < 1000000; ++i) {
    if (i % 2 == 0) continue;
}
```

**Use noexcept:**
```cpp
// Enables optimizations
void swap(int& a, int& b) noexcept {
    int temp = a;
    a = b;
    b = temp;
}
```

**Avoid Deep Call Stacks:**
```cpp
// SLOW: Exception travels through many frames
void f1() { f2(); }
void f2() { f3(); }
// ... 100 more functions
void f100() { throw runtime_error("Error"); }

// FASTER: Shallower stacks or error codes
```

### 15.3 When to Avoid Exceptions

**Performance-Critical Code:**
```cpp
// Hot path - avoid exceptions
inline int fastDivide(int a, int b) noexcept {
    return a / b;  // Caller must ensure b != 0
}

// Non-critical path - exceptions OK
int safeDivide(int a, int b) {
    if (b == 0) throw std::invalid_argument("Divide by zero");
    return a / b;
}
```

---

## 16. Common Mistakes

### 16.1 Throwing Pointers

```cpp
// WRONG
throw new std::runtime_error("Error");

// Who calls delete? Memory leak!
```

### 16.2 Catching by Value

```cpp
// WRONG: Object slicing
catch (std::exception e) {
    // Derived class info lost
}

// CORRECT
catch (const std::exception& e) {
    // Preserves derived class
}
```

### 16.3 Empty Catch Blocks

```cpp
// WRONG: Hides errors
try {
    criticalOperation();
}
catch (...) {
    // Silently fail - DANGEROUS!
}

// CORRECT
catch (const std::exception& e) {
    logError(e.what());
    // Handle or re-throw
}
```

### 16.4 Throwing from Destructors

```cpp
// WRONG
~MyClass() {
    if (condition) throw std::runtime_error("Error");
}

// CORRECT
~MyClass() noexcept {
    try {
        cleanup();
    } catch (...) {
        // Log but don't throw
    }
}
```

### 16.5 Not Re-throwing Correctly

```cpp
// WRONG: Loses exception type
catch (const std::exception& e) {
    log(e);
    throw e;  // Slices to std::exception!
}

// CORRECT
catch (const std::exception& e) {
    log(e);
    throw;  // Re-throws original exception
}
```

### 16.6 Resource Leaks

```cpp
// WRONG
void bad() {
    int* p = new int[100];
    riskyOperation();  // Throws
    delete[] p;  // Never reached
}

// CORRECT
void good() {
    std::vector<int> v(100);  // RAII
    riskyOperation();  // If throws, v cleaned up
}
```

---

## 17. Quick Reference

### 17.1 Syntax Summary

```cpp
// Throwing
throw std::runtime_error("Error");
throw;  // Re-throw current exception

// Try-Catch
try {
    riskyOperation();
}
catch (const SpecificException& e) {
    // Handle specific
}
catch (const std::exception& e) {
    // Handle general
}
catch (...) {
    // Catch all
}

// Function-Try-Block
MyClass::MyClass(int value)
try : member(value) {
    // Constructor body
}
catch (const std::exception& e) {
    // Handle initialization exceptions
    throw;  // Must re-throw
}

// noexcept
void func() noexcept;
void func() noexcept(condition);
```

### 17.2 Standard Exceptions Hierarchy

```
std::exception
├── std::bad_alloc
├── std::bad_cast
├── std::bad_typeid
├── std::bad_exception
├── std::logic_error
│   ├── std::invalid_argument
│   ├── std::domain_error
│   ├── std::length_error
│   └── std::out_of_range
└── std::runtime_error
    ├── std::range_error
    ├── std::overflow_error
    └── std::underflow_error
```

### 17.3 Exception Safety Guarantees

| Guarantee | Description | Example |
|-----------|-------------|---------|
| **No-throw** | Never fails | Destructors, swap, move |
| **Strong** | Commit or rollback | Copy-and-swap assignment |
| **Basic** | No leaks, valid state | Reserve with RAII |
| **None** | Might leak | Raw pointers, no RAII |

### 17.4 Checklist

**Exception Class Design:**
- [ ] Derive from `std::exception` or subclass
- [ ] Override `what()` as `const noexcept`
- [ ] Make destructor `noexcept`
- [ ] Make copy/move operations `noexcept`
- [ ] Provide meaningful error messages

**Throwing:**
- [ ] Throw by value, not pointer
- [ ] Use standard exceptions when appropriate
- [ ] Include context in exception message
- [ ] Never throw from destructors
- [ ] Never throw from `noexcept` functions

**Catching:**
- [ ] Catch by const reference
- [ ] Order catch blocks specific to general
- [ ] Re-throw using `throw;` (not `throw e;`)
- [ ] Don't catch and ignore silently
- [ ] Use catch-all `(...)` only as last resort

**Exception Safety:**
- [ ] Use RAII for resource management
- [ ] Mark move operations `noexcept`
- [ ] Provide strong or basic guarantee
- [ ] Document exception behavior
- [ ] Test exception paths

**Performance:**
- [ ] Don't use exceptions for control flow
- [ ] Use `noexcept` where possible
- [ ] Avoid exceptions in hot paths
- [ ] Minimize call stack depth

### 17.5 Common Patterns

**RAII Resource Management:**
```cpp
class Resource {
    Handle* handle;
public:
    Resource() : handle(acquire()) {
        if (!handle) throw std::runtime_error("Failed");
    }
    ~Resource() { if (handle) release(handle); }
    
    Resource(const Resource&) = delete;
    Resource(Resource&& r) noexcept : handle(r.handle) {
        r.handle = nullptr;
    }
};
```

**Strong Exception Safety:**
```cpp
Container& operator=(const Container& other) {
    Container temp(other);  // Copy (may throw)
    swap(*this, temp);      // Swap (noexcept)
    return *this;           // Destroy old (noexcept)
}
```

**Exception Translation:**
```cpp
try {
    lowLevelOperation();
}
catch (const LowLevelError& e) {
    throw HighLevelError("Operation failed: " + 
                         std::string(e.what()));
}
```

**Cleanup on Exception:**
```cpp
void operation() {
    Resource* r = acquire();
    try {
        use(r);
    }
    catch (...) {
        release(r);  // Cleanup
        throw;       // Re-throw
    }
    release(r);
}

// Better: Use RAII
void operation() {
    Resource r;  // Automatic cleanup
    use(r);
}
```

### 17.6 Decision Tree: When to Use Exceptions

```
Is this an error condition?
│
├─ NO → Use normal return value/control flow
│
└─ YES → Is it recoverable?
    │
    ├─ NO → std::terminate() or assert
    │
    └─ YES → Is it a programming error?
        │
        ├─ YES (logic error) → std::logic_error or derived
        │   │
        │   └─ Examples:
        │       • Invalid argument → std::invalid_argument
        │       • Out of range → std::out_of_range
        │       • Precondition violated → Custom or std::logic_error
        │
        └─ NO (runtime error) → std::runtime_error or derived
            │
            └─ Examples:
                • I/O error → std::ios_base::failure or custom
                • Network error → Custom exception
                • Resource unavailable → std::runtime_error
                • Memory exhausted → std::bad_alloc
```

### 17.7 Interview Questions & Answers

**Q1: What happens if an exception is thrown and not caught?**
- `std::terminate()` is called
- Calls current terminate handler
- Default: calls `std::abort()`
- Program terminates abnormally

**Q2: Can you throw exceptions from a destructor?**
- Technically yes, but DON'T
- During stack unwinding: two active exceptions → terminate
- Destructors implicitly `noexcept`
- Violates RAII principle

**Q3: What is stack unwinding?**
- Process of exiting stack frames when exception thrown
- Calls destructors for automatic objects
- Continues until matching catch block found
- Enables RAII and automatic cleanup

**Q4: Difference between throw; and throw e;?**
- `throw;` re-throws current exception (preserves type)
- `throw e;` throws copy of e (may slice derived class)
- Always use `throw;` to re-throw

**Q5: What is exception safety?**
- Four guarantees: no-throw, strong, basic, none
- No-throw: cannot fail
- Strong: commit or rollback (transactional)
- Basic: no leaks, valid state
- None: avoid this

**Q6: Why catch by const reference?**
- Avoids slicing (preserves derived class)
- No copy overhead
- Can call virtual functions correctly
- `const` prevents accidental modification

**Q7: What is RAII?**
- Resource Acquisition Is Initialization
- Acquire in constructor, release in destructor
- Automatic cleanup via stack unwinding
- Exception-safe resource management

**Q8: Can constructors throw?**
- Yes, constructors can throw
- Object not created if constructor throws
- Destructor not called
- Already-constructed members cleaned up automatically

**Q9: What is noexcept?**
- Specifies function cannot throw
- Enables compiler optimizations
- Calls `std::terminate()` if violated
- Required for move operations, destructors

**Q10: What is std::exception_ptr?**
- Type-erased exception holder
- Store exception for later re-throw
- Transfer exceptions between threads
- Use `std::current_exception()` to capture

---

## 18. Real-World Examples

### 18.1 File Processing with Exceptions

```cpp
#include <fstream>
#include <string>
#include <stdexcept>

class FileProcessor {
public:
    std::string processFile(const std::string& filename) {
        std::ifstream file(filename);
        
        if (!file.is_open()) {
            throw std::runtime_error("Cannot open file: " + filename);
        }
        
        std::string content;
        std::string line;
        
        try {
            while (std::getline(file, line)) {
                if (line.empty()) {
                    throw std::invalid_argument("Empty line not allowed");
                }
                content += processLine(line) + "\n";
            }
        }
        catch (const std::exception& e) {
            // File automatically closed by ifstream destructor
            throw std::runtime_error(
                "Processing failed for " + filename + ": " + e.what());
        }
        
        return content;
        // File automatically closed here
    }
    
private:
    std::string processLine(const std::string& line) {
        if (line.length() > 1000) {
            throw std::length_error("Line too long");
        }
        // Process...
        return line;
    }
};

// Usage
int main() {
    FileProcessor processor;
    
    try {
        std::string result = processor.processFile("data.txt");
        std::cout << "Processed: " << result << std::endl;
    }
    catch (const std::length_error& e) {
        std::cerr << "Length error: " << e.what() << std::endl;
    }
    catch (const std::invalid_argument& e) {
        std::cerr << "Invalid input: " << e.what() << std::endl;
    }
    catch (const std::runtime_error& e) {
        std::cerr << "Runtime error: " << e.what() << std::endl;
    }
    catch (const std::exception& e) {
        std::cerr << "Unknown error: " << e.what() << std::endl;
    }
    
    return 0;
}
```

### 18.2 Database Connection with RAII

```cpp
#include <memory>
#include <string>
#include <stdexcept>

class DatabaseException : public std::runtime_error {
public:
    using std::runtime_error::runtime_error;
};

class Connection {
    // Actual database connection
    void* handle;
    bool connected;
    
public:
    Connection(const std::string& connectionString) 
        : handle(nullptr), connected(false) {
        
        // Simulate connection
        handle = connectToDatabase(connectionString);
        if (!handle) {
            throw DatabaseException(
                "Failed to connect: " + connectionString);
        }
        connected = true;
    }
    
    ~Connection() noexcept {
        if (connected && handle) {
            try {
                disconnect(handle);
            }
            catch (...) {
                // Log but don't throw
                std::cerr << "Disconnect failed\n";
            }
        }
    }
    
    // Delete copy, allow move
    Connection(const Connection&) = delete;
    Connection& operator=(const Connection&) = delete;
    
    Connection(Connection&& other) noexcept 
        : handle(other.handle), connected(other.connected) {
        other.handle = nullptr;
        other.connected = false;
    }
    
    void executeQuery(const std::string& query) {
        if (!connected) {
            throw DatabaseException("Not connected");
        }
        
        if (query.empty()) {
            throw std::invalid_argument("Empty query");
        }
        
        // Execute query (may throw)
        if (!execute(handle, query)) {
            throw DatabaseException("Query failed: " + query);
        }
    }
    
private:
    void* connectToDatabase(const std::string& str) {
        // Simulate
        return str.empty() ? nullptr : (void*)1;
    }
    
    void disconnect(void* h) noexcept {
        // Simulate
    }
    
    bool execute(void* h, const std::string& q) {
        // Simulate
        return true;
    }
};

class Database {
    std::unique_ptr<Connection> connection;
    
public:
    Database(const std::string& connectionString) {
        try {
            connection = std::make_unique<Connection>(connectionString);
        }
        catch (const DatabaseException& e) {
            // Add context
            throw DatabaseException(
                "Database initialization failed: " + 
                std::string(e.what()));
        }
    }
    
    void query(const std::string& sql) {
        if (!connection) {
            throw DatabaseException("No connection");
        }
        
        try {
            connection->executeQuery(sql);
        }
        catch (const std::invalid_argument& e) {
            throw DatabaseException(
                "Invalid query: " + std::string(e.what()));
        }
        // Let DatabaseException propagate
    }
};

// Usage
int main() {
    try {
        Database db("server=localhost;database=test");
        db.query("SELECT * FROM users");
        std::cout << "Query successful\n";
    }
    catch (const DatabaseException& e) {
        std::cerr << "Database error: " << e.what() << std::endl;
        return 1;
    }
    catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << std::endl;
        return 1;
    }
    
    return 0;
}
```

### 18.3 Transaction with Strong Exception Safety

```cpp
#include <vector>
#include <string>
#include <algorithm>
#include <stdexcept>

class Account {
    std::string id;
    double balance;
    
public:
    Account(std::string id, double bal) 
        : id(std::move(id)), balance(bal) {
        if (balance < 0) {
            throw std::invalid_argument("Negative balance");
        }
    }
    
    void deposit(double amount) {
        if (amount <= 0) {
            throw std::invalid_argument("Invalid deposit amount");
        }
        balance += amount;
    }
    
    void withdraw(double amount) {
        if (amount <= 0) {
            throw std::invalid_argument("Invalid withdrawal amount");
        }
        if (amount > balance) {
            throw std::runtime_error("Insufficient funds");
        }
        balance -= amount;
    }
    
    double getBalance() const { return balance; }
    const std::string& getId() const { return id; }
};

class Bank {
    std::vector<Account> accounts;
    
public:
    void addAccount(const Account& account) {
        accounts.push_back(account);
    }
    
    // Strong exception safety guarantee
    void transfer(const std::string& fromId, 
                  const std::string& toId, 
                  double amount) {
        
        // Find accounts
        auto fromIt = std::find_if(accounts.begin(), accounts.end(),
            [&](const Account& a) { return a.getId() == fromId; });
        auto toIt = std::find_if(accounts.begin(), accounts.end(),
            [&](const Account& a) { return a.getId() == toId; });
        
        if (fromIt == accounts.end() || toIt == accounts.end()) {
            throw std::invalid_argument("Account not found");
        }
        
        // Save state for rollback
        double fromBalance = fromIt->getBalance();
        double toBalance = toIt->getBalance();
        
        try {
            // Perform transaction
            fromIt->withdraw(amount);
            toIt->deposit(amount);
            
            // Transaction successful
            std::cout << "Transfer successful: " << amount 
                      << " from " << fromId << " to " << toId << "\n";
        }
        catch (const std::exception& e) {
            // Rollback on any error (strong guarantee)
            // Note: In real implementation, you'd need to properly
            // restore state; this is simplified
            
            std::cerr << "Transfer failed: " << e.what() 
                      << " - rolling back\n";
            throw std::runtime_error(
                "Transfer failed: " + std::string(e.what()));
        }
    }
};

// Usage
int main() {
    try {
        Bank bank;
        bank.addAccount(Account("ACC001", 1000.0));
        bank.addAccount(Account("ACC002", 500.0));
        
        bank.transfer("ACC001", "ACC002", 200.0);  // Success
        bank.transfer("ACC001", "ACC002", 2000.0); // Fails (insufficient)
    }
    catch (const std::exception& e) {
        std::cerr << "Banking error: " << e.what() << std::endl;
        return 1;
    }
    
    return 0;
}
```

### 18.4 Network Request with Retries

```cpp
#include <string>
#include <chrono>
#include <thread>
#include <stdexcept>

class NetworkException : public std::runtime_error {
    bool retryable_;
    
public:
    NetworkException(const std::string& msg, bool retryable = false)
        : std::runtime_error(msg), retryable_(retryable) {}
    
    bool isRetryable() const { return retryable_; }
};

class HttpClient {
public:
    std::string get(const std::string& url, int maxRetries = 3) {
        int attempt = 0;
        
        while (attempt <= maxRetries) {
            try {
                return performRequest(url);
            }
            catch (const NetworkException& e) {
                attempt++;
                
                if (!e.isRetryable() || attempt > maxRetries) {
                    throw NetworkException(
                        "Request failed after " + 
                        std::to_string(attempt) + " attempts: " + e.what(),
                        false);
                }
                
                // Exponential backoff
                int delayMs = 100 * (1 << attempt);
                std::cerr << "Retry " << attempt << "/" << maxRetries 
                          << " after " << delayMs << "ms\n";
                std::this_thread::sleep_for(
                    std::chrono::milliseconds(delayMs));
            }
            catch (const std::exception& e) {
                // Non-retryable error
                throw NetworkException(
                    "Fatal error: " + std::string(e.what()), false);
            }
        }
        
        throw NetworkException("Max retries exceeded", false);
    }
    
private:
    std::string performRequest(const std::string& url) {
        if (url.empty()) {
            throw std::invalid_argument("Empty URL");
        }
        
        // Simulate network issues
        static int callCount = 0;
        callCount++;
        
        if (callCount < 3) {
            throw NetworkException("Connection timeout", true);
        }
        
        return "Response from " + url;
    }
};

// Usage
int main() {
    HttpClient client;
    
    try {
        std::string response = client.get("https://api.example.com/data");
        std::cout << "Success: " << response << std::endl;
    }
    catch (const NetworkException& e) {
        std::cerr << "Network error: " << e.what() << std::endl;
        if (e.isRetryable()) {
            std::cerr << "(Could retry later)\n";
        }
        return 1;
    }
    catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << std::endl;
        return 1;
    }
    
    return 0;
}
```

---

## 19. Exception Handling Patterns

### 19.1 Guard Pattern

**Problem:** Need cleanup whether exception occurs or not.

```cpp
class Guard {
    std::function<void()> cleanup;
    bool dismissed = false;
    
public:
    explicit Guard(std::function<void()> f) : cleanup(std::move(f)) {}
    
    ~Guard() {
        if (!dismissed) cleanup();
    }
    
    void dismiss() { dismissed = true; }
};

void example() {
    File* f = openFile("data.txt");
    Guard g([f]{ closeFile(f); });  // Cleanup on any exit
    
    processFile(f);  // May throw
    
    g.dismiss();  // Success, don't close (or keep cleanup)
}
```

### 19.2 Two-Phase Initialization

**Problem:** Constructor might fail after partial initialization.

```cpp
class Widget {
    Resource* resource;
    
    // Phase 1: Basic initialization (noexcept)
    Widget() noexcept : resource(nullptr) {}
    
public:
    // Factory method
    static std::unique_ptr<Widget> create() {
        auto widget = std::make_unique<Widget>();
        
        // Phase 2: Complex initialization (may throw)
        widget->initialize();
        
        return widget;
    }
    
private:
    void initialize() {
        resource = new Resource();  // May throw
        // If throws, Widget destructor handles cleanup
    }
    
    ~Widget() {
        delete resource;
    }
};
```

### 19.3 Error Accumulation

**Problem:** Continue processing despite errors.

```cpp
class ErrorAccumulator {
    std::vector<std::exception_ptr> errors;
    
public:
    template<typename F>
    void tryExecute(F&& func) {
        try {
            func();
        }
        catch (...) {
            errors.push_back(std::current_exception());
        }
    }
    
    void throwIfErrors() {
        if (!errors.empty()) {
            throw std::runtime_error(
                "Operations failed: " + std::to_string(errors.size()));
        }
    }
    
    void rethrowAll() {
        for (auto& e : errors) {
            try {
                std::rethrow_exception(e);
            }
            catch (const std::exception& ex) {
                std::cerr << "Error: " << ex.what() << "\n";
            }
        }
    }
};

// Usage
void processMultiple(const std::vector<std::string>& files) {
    ErrorAccumulator accumulator;
    
    for (const auto& file : files) {
        accumulator.tryExecute([&]{ 
            processFile(file); 
        });
    }
    
    accumulator.throwIfErrors();  // Fail if any errors
}
```

---

## 20. Testing Exception Handling

### 20.1 Testing Exceptions

```cpp
#include <cassert>

void testDivisionByZero() {
    try {
        divide(10, 0);
        assert(false && "Should have thrown");
    }
    catch (const std::runtime_error& e) {
        assert(std::string(e.what()) == "Division by zero");
    }
}

void testNoException() {
    try {
        int result = divide(10, 2);
        assert(result == 5);
    }
    catch (...) {
        assert(false && "Should not throw");
    }
}
```

### 20.2 Testing Exception Safety

```cpp
// Test strong exception safety
void testStrongGuarantee() {
    Container c;
    c.add(1);
    c.add(2);
    
    Container backup = c;  // Save state
    
    try {
        c.riskyOperation();  // May throw
    }
    catch (...) {
        assert(c == backup);  // State unchanged
    }
}

// Test basic exception safety
void testBasicGuarantee() {
    Container c;
    c.add(1);
    
    try {
        c.riskyOperation();
    }
    catch (...) {
        assert(c.isValid());  // Still valid
        // State may differ, but no leaks
    }
}
```

---

## 21. Summary & Final Tips

### 21.1 Golden Rules

1. **Use RAII everywhere** - Automatic resource management
2. **Destructors never throw** - Mark `noexcept`, catch and log
3. **Catch by const reference** - Avoid slicing and copying
4. **Re-throw with throw;** - Preserve exception type
5. **Order catches correctly** - Specific to general
6. **Provide exception safety** - At least basic guarantee
7. **Use standard exceptions** - Don't reinvent the wheel
8. **Document exceptions** - What, when, why
9. **Mark noexcept** - Move operations, swap, destructors
10. **Test exception paths** - Don't ignore error cases

### 21.2 Exception Hierarchy Strategy

```
MyAppException (base)
├── LogicException (programming errors)
│   ├── InvalidConfigException
│   └── InvalidStateException
└── RuntimeException (runtime errors)
    ├── DatabaseException
    │   ├── ConnectionException
    │   └── QueryException
    ├── NetworkException
    └── FileException
```

### 21.3 When NOT to Use Exceptions

- Expected conditions (use return codes)
- Performance-critical tight loops
- Real-time systems with strict timing
- Embedded systems with no exception support
- Interfacing with C code
- Signal handlers

### 21.4 Quick Decision Guide

```
Exception or Return Code?
│
├─ Frequent occurrence? → Return code
├─ Expected condition? → Return code
├─ Performance critical? → Return code
├─ Cross-language boundary? → Return code
├─ Constructor/operator? → Exception
├─ Error requires cleanup? → Exception
├─ Error skips stack frames? → Exception
└─ Otherwise → Either (prefer exceptions for clarity)
```

### 21.5 Exam/Interview Key Points

**Must Know:**
- Exception handling mechanism (throw/try/catch)
- Standard exception hierarchy
- Stack unwinding process
- Exception safety guarantees
- RAII principle
- Why catch by const reference
- Why destructors shouldn't throw
- What `noexcept` does

**Common Questions:**
1. What happens to local objects when exception thrown?
2. Can you catch multiple exception types?
3. What's the difference between throw; and throw e;?
4. Why mark move operations noexcept?
5. How to provide strong exception safety?
6. What is std::exception_ptr used for?

**Code Patterns to Remember:**
- RAII resource wrapper
- Copy-and-swap assignment
- Exception translation
- Scope guard
- Custom exception class

---

## Final Checklist ✓

**Implementation:**
- [ ] Use RAII for all resources
- [ ] Make destructors `noexcept`
- [ ] Make move operations `noexcept`
- [ ] Catch by `const&`
- [ ] Order catch blocks correctly
- [ ] Re-throw with `throw;`
- [ ] Provide exception safety guarantees
- [ ] Document what exceptions are thrown

**Design:**
- [ ] Create meaningful exception hierarchy
- [ ] Derive from `std::exception`
- [ ] Override `what()` as `const noexcept`
- [ ] Include error context in exceptions
- [ ] Use standard exceptions when appropriate
- [ ] Don't throw from destructors

**Testing:**
- [ ] Test exception paths
- [ ] Verify resource cleanup
- [ ] Check exception safety
- [ ] Test with invalid inputs
- [ ] Verify error messages

**This completes the comprehensive guide to C++ Exception Handling!**
