# C++ Templates: Complete Guide - Part 1
## Fundamentals & Core Concepts

---

## Table of Contents (Part 1)
1. What Are Templates?
2. Why Templates?
3. Function Templates
4. Class Templates
5. Template Parameters
6. Template Instantiation

---

## 1. What Are Templates?

### Definition
A **template** is a C++ entity that defines one of the following:
- A **family of classes** (class template)
- A **family of functions** (function template)
- A **family of variables** (variable template - C++14+)
- A **family of alias types** (alias template - C++11+)

### Key Concept: Templates are NOT Code
**Critical Understanding:**
- A template is **NOT** a type, object, or actual entity
- **No code is generated** from a source file containing only template definitions
- Templates are **blueprints** or **recipes** for generating actual code
- Code is generated only when the template is **instantiated** with concrete arguments

```cpp
// This generates NO actual code
template<typename T>
class Vector {
    T* data;
    size_t size;
};

// Code is generated ONLY when instantiated
Vector<int> v1;      // NOW code for Vector<int> is generated
Vector<double> v2;   // NOW code for Vector<double> is generated
```

---

## 2. Why Templates?

### Problem Without Templates
```cpp
// Without templates, you need separate functions for each type
int max(int a, int b) { return a > b ? a : b; }
double max(double a, double b) { return a > b ? a : b; }
std::string max(std::string a, std::string b) { return a > b ? a : b; }
// ... endless repetition!
```

### Solution With Templates
```cpp
// ONE template works for ALL types!
template<typename T>
T max(T a, T b) {
    return a > b ? a : b;
}

// Usage - compiler generates specific versions
int x = max(5, 10);           // T = int
double y = max(3.5, 2.1);     // T = double
std::string z = max("abc"s, "xyz"s); // T = std::string
```

### Benefits
1. **Code Reusability** - Write once, use for any type
2. **Type Safety** - Compile-time type checking (unlike C's void*)
3. **Performance** - Zero runtime overhead (unlike virtual functions)
4. **Generic Programming** - Write algorithms independent of data types

---

## 3. Function Templates

### Basic Syntax
```cpp
template<typename T>        // Template parameter declaration
T functionName(T param) {   // Function using T
    return param;
}
```

### Complete Example
```cpp
#include <iostream>

// Function template declaration
template<typename T>
T add(T a, T b) {
    return a + b;
}

int main() {
    std::cout << add(5, 10) << '\n';        // T deduced as int
    std::cout << add(3.5, 2.1) << '\n';     // T deduced as double
    std::cout << add<int>(5, 10) << '\n';   // Explicit: T = int
}
```

### Multiple Template Parameters
```cpp
template<typename T, typename U>
auto multiply(T a, U b) -> decltype(a * b) {
    return a * b;
}

// Usage
auto result1 = multiply(5, 3.5);    // T=int, U=double, returns double
auto result2 = multiply(2.5, 4);    // T=double, U=int, returns double
```

### Template Parameter Variations

#### 1. Type Parameters (Most Common)
```cpp
template<typename T>  // 'typename' keyword
void func1(T param) { }

template<class T>     // 'class' keyword (equivalent to typename)
void func2(T param) { }

// Both are identical! 'typename' is preferred in modern C++
```

#### 2. Multiple Parameters
```cpp
template<typename T, typename U, typename V>
V convert(T from, U intermediate) {
    return static_cast<V>(from * intermediate);
}
```

#### 3. Default Template Arguments
```cpp
template<typename T = int>
T getValue() {
    return T{};
}

auto x = getValue();      // Uses default: int
auto y = getValue<double>(); // Explicit: double
```

---

## 4. Class Templates

### Basic Syntax
```cpp
template<typename T>
class ClassName {
    T member;
public:
    void method(T param);
};
```

### Complete Example
```cpp
template<typename T>
class Box {
private:
    T value;
    
public:
    // Constructor
    Box(T v) : value(v) {}
    
    // Getter
    T getValue() const { return value; }
    
    // Setter
    void setValue(T v) { value = v; }
};

// Usage
Box<int> intBox(42);
Box<std::string> strBox("Hello");

std::cout << intBox.getValue() << '\n';    // 42
std::cout << strBox.getValue() << '\n';    // Hello
```

### Member Functions Outside Class Definition
```cpp
template<typename T>
class Point {
    T x, y;
public:
    Point(T x, T y);           // Declaration
    T getX() const;            // Declaration
    void print() const;        // Declaration
};

// Definition OUTSIDE class - note the syntax!
template<typename T>              // Template declaration needed
Point<T>::Point(T x, T y)         // Point<T>:: not just Point::
    : x(x), y(y) {}

template<typename T>
T Point<T>::getX() const {
    return x;
}

template<typename T>
void Point<T>::print() const {
    std::cout << "(" << x << ", " << y << ")\n";
}

// Usage
Point<int> p(10, 20);
Point<double> q(3.5, 7.8);
```

**Key Points for Out-of-Class Definitions:**
1. Must repeat `template<typename T>` before each definition
2. Use `ClassName<T>::` not just `ClassName::`
3. Must be in the **same header file** (explained later)

---

## 5. Template Parameters

C++ supports **three kinds** of template parameters:

### 5.1 Type Parameters (Most Common)
```cpp
template<typename T>        // Type parameter
class Container {
    T* data;
};

template<class U>           // Same as typename
void process(U item) { }
```

### 5.2 Non-Type Parameters (Constants)
Values known at compile-time:

```cpp
template<typename T, int Size>  // Type + Constant
class Array {
    T data[Size];               // Size must be compile-time constant
public:
    static constexpr int size() { return Size; }
};

// Usage
Array<int, 10> arr1;     // Array of 10 ints
Array<double, 5> arr2;   // Array of 5 doubles

// Size is part of the type!
// Array<int, 10> and Array<int, 20> are DIFFERENT types
```

#### Valid Non-Type Parameters:
```cpp
// Integral types
template<int N> class A { };
template<char C> class B { };
template<bool Flag> class C { };

// Enumerations
enum Color { RED, GREEN, BLUE };
template<Color C> class D { };

// Pointers (to objects/functions with linkage)
template<int* Ptr> class E { };
extern int globalVar;
E<&globalVar> obj;  // OK

// References
template<int& Ref> class F { };

// nullptr_t
template<std::nullptr_t> class G { };

// C++20: Literal class types
template<std::array<int, 3> Arr> class H { };
```

### 5.3 Template Template Parameters
A template that takes another template as a parameter:

```cpp
// Basic template template parameter
template<typename T, template<typename> class Container>
class Stack {
    Container<T> data;
};

// Usage
template<typename T> class Vector { };
template<typename T> class List { };

Stack<int, Vector> s1;  // Stack using Vector<int>
Stack<int, List> s2;    // Stack using List<int>
```

**More Complex Example:**
```cpp
template<typename Key, 
         typename Value, 
         template<typename, typename> class Map>
class Database {
    Map<Key, Value> storage;
};

// Works with any map-like container
Database<int, std::string, std::map> db1;
Database<int, std::string, std::unordered_map> db2;
```

---

## 6. Template Instantiation

### What is Instantiation?
**Instantiation** is the process where the compiler generates actual code from a template by substituting template parameters with concrete arguments.

### 6.1 Implicit Instantiation (Automatic)
Happens automatically when you use a template:

```cpp
template<typename T>
class MyClass {
    T value;
public:
    void method() { }
};

MyClass<int> obj1;      // Compiler generates MyClass for int
MyClass<double> obj2;   // Compiler generates MyClass for double

obj1.method();          // method() is instantiated for MyClass<int>
// If you never call method(), it's never instantiated!
```

**Important:** Only the members you actually **use** are instantiated!

```cpp
template<typename T>
class LazyClass {
public:
    void usedMethod() { }
    void unusedMethod() {
        // This would cause error for some types
        // But it's never instantiated if never called!
    }
};

LazyClass<int> obj;
obj.usedMethod();  // Only this is instantiated
// unusedMethod() is NOT instantiated, so no error!
```

### 6.2 Explicit Instantiation
Force instantiation without using the template:

```cpp
// Explicit instantiation definition
template class MyClass<int>;      // Forces generation of MyClass<int>
template void func<double>(double); // Forces generation of func<double>

// Explicit instantiation declaration (extern)
extern template class MyClass<float>;  // Don't instantiate here
                                      // (it's defined elsewhere)
```

**Use Case:** Speed up compilation by instantiating common types in one .cpp file:

```cpp
// MyTemplate.h
template<typename T>
class MyTemplate { /* ... */ };

// MyTemplate.cpp (implementation file)
#include "MyTemplate.h"
template class MyTemplate<int>;     // Explicit instantiation
template class MyTemplate<double>;  // Explicit instantiation

// main.cpp
#include "MyTemplate.h"
extern template class MyTemplate<int>; // Don't instantiate here

MyTemplate<int> obj; // Uses instantiation from MyTemplate.cpp
```

### 6.3 Point of Instantiation (POI)
The location in code where the compiler generates the template code:

```cpp
template<typename T>
void func() { }

void foo() {
    func<int>();  // POI is here for func<int>
}
```

---

## Key Concepts Summary

### 1. Template Definition Must Be Visible
```cpp
// ❌ WRONG - Won't Link!
// header.h
template<typename T> class MyClass;  // Declaration only

// implementation.cpp
template<typename T> 
class MyClass { /* definition */ };  // Not visible to other files!

// main.cpp
MyClass<int> obj;  // Linker error!
```

```cpp
// ✅ CORRECT - Put in Header
// header.h
template<typename T>
class MyClass {
    // Full definition must be in header
};

// main.cpp
#include "header.h"
MyClass<int> obj;  // Works!
```

**Why?** The compiler needs to see the complete template definition to instantiate it.

### 2. Template != Type
```cpp
template<typename T>
class Vector { };

// Vector is NOT a type
// Vector<int> IS a type
// Vector<double> IS a different type

Vector v;        // ❌ Error! Must specify T
Vector<int> v;   // ✅ OK
```

### 3. Each Instantiation is a Separate Type
```cpp
Array<int, 10> a1;
Array<int, 20> a2;
// a1 and a2 are COMPLETELY DIFFERENT types!
// Cannot assign between them

Array<int, 10> a3;
// a1 and a3 ARE the same type
a3 = a1;  // OK
```

---

## Common Pitfalls & Edge Cases

### Pitfall 1: Angle Bracket Parsing
```cpp
vector<vector<int>> v;  // C++11: OK
vector<vector<int> > v; // Old C++: Need space between >>
```

### Pitfall 2: Dependent Names
```cpp
template<typename T>
class MyClass {
    typename T::value_type x;  // Need 'typename' keyword!
    
    template<typename U>
    void method() {
        T::template innerTemplate<U>();  // Need 'template' keyword!
    }
};
```

### Pitfall 3: Two-Phase Lookup
```cpp
template<typename T>
void func() {
    helper();  // If helper() not found now, error!
               // Won't wait for instantiation
}
```

---

## Things to Remember

✅ **Templates are compile-time entities** - all resolved before runtime
✅ **No runtime overhead** - as efficient as hand-written code
✅ **Definition must be visible** - keep in headers
✅ **Each instantiation is separate** - can lead to code bloat
✅ **Only used members are instantiated** - enables lazy evaluation
✅ **Type deduction is powerful** - often don't need explicit types
✅ **Error messages can be cryptic** - take time to learn

---

# C++ Templates: Complete Guide - Part 2
## Template Specialization & Argument Deduction

---

## Table of Contents (Part 2)
1. Template Specialization
2. Template Argument Deduction
3. Class Template Argument Deduction (CTAD)
4. Forwarding References & Perfect Forwarding

---

## 1. Template Specialization

### 1.1 Full (Explicit) Specialization

**Purpose:** Provide a completely custom implementation for specific template arguments.

#### Function Template Specialization
```cpp
// Primary template
template<typename T>
void print(T value) {
    std::cout << "Generic: " << value << '\n';
}

// Full specialization for bool
template<>
void print<bool>(bool value) {
    std::cout << "Boolean: " << (value ? "true" : "false") << '\n';
}

// Usage
print(42);      // Uses primary: "Generic: 42"
print(true);    // Uses specialization: "Boolean: true"
```

#### Class Template Specialization
```cpp
// Primary template
template<typename T>
class Storage {
    T value;
public:
    Storage(T v) : value(v) {}
    void print() { std::cout << "Value: " << value << '\n'; }
};

// Full specialization for bool
template<>
class Storage<bool> {
    unsigned char value;  // Store as bit
public:
    Storage(bool v) : value(v ? 1 : 0) {}
    void print() { 
        std::cout << "Boolean: " << (value ? "true" : "false") << '\n'; 
    }
};

// Usage
Storage<int> s1(42);    // Uses primary template
Storage<bool> s2(true); // Uses specialized version
```

**Real-World Example: std::vector<bool>**
```cpp
// This is what the standard library does:
template<typename T>
class vector {
    T* data;  // Normal array
};

template<>
class vector<bool> {
    // Special implementation using bits
    // to save space
};
```

### 1.2 Partial Specialization (Class Templates Only)

**Purpose:** Specialize for a subset of template parameters or parameter patterns.

#### Basic Partial Specialization
```cpp
// Primary template
template<typename T, typename U>
class Pair {
public:
    void identify() { std::cout << "Generic Pair\n"; }
};

// Partial specialization: both types are the same
template<typename T>
class Pair<T, T> {
public:
    void identify() { std::cout << "Same Type Pair\n"; }
};

// Partial specialization: second type is int
template<typename T>
class Pair<T, int> {
public:
    void identify() { std::cout << "Second is int\n"; }
};

// Usage
Pair<double, std::string> p1; p1.identify(); // "Generic Pair"
Pair<int, int> p2;           p2.identify(); // "Same Type Pair"
Pair<double, int> p3;        p3.identify(); // "Second is int"
```

#### Pointer Specialization
```cpp
// Primary template
template<typename T>
class SmartPtr {
    T value;
public:
    T& get() { return value; }
};

// Partial specialization for pointers
template<typename T>
class SmartPtr<T*> {
    T* ptr;
public:
    T* get() { return ptr; }
    T& operator*() { return *ptr; }  // Dereference operator
    T* operator->() { return ptr; }  // Arrow operator
};

// Usage
SmartPtr<int> s1;        // Uses primary
SmartPtr<int*> s2;       // Uses pointer specialization
```

#### Array Specialization
```cpp
// Primary template
template<typename T>
class Container {
public:
    void info() { std::cout << "Regular type\n"; }
};

// Partial specialization for arrays
template<typename T, size_t N>
class Container<T[N]> {
public:
    void info() { std::cout << "Array of " << N << " elements\n"; }
};

// Usage
Container<int> c1;       c1.info(); // "Regular type"
Container<int[10]> c2;   c2.info(); // "Array of 10 elements"
```

### 1.3 Specialization Rules & Gotchas

#### Rule 1: Specialization Must Be Declared Before First Use
```cpp
template<typename T>
void func(T) { std::cout << "Primary\n"; }

void test() {
    func(42);  // Uses primary (specialization not yet seen)
}

// ❌ TOO LATE! Already instantiated above
template<>
void func<int>(int) { std::cout << "Specialized\n"; }
```

#### Rule 2: Function Template Specialization vs Overloading
```cpp
// Primary template
template<typename T>
void process(T value) {
    std::cout << "Template: " << value << '\n';
}

// ❌ WRONG: Specialization
template<>
void process<int*>(int* ptr) {
    std::cout << "Specialized: " << *ptr << '\n';
}

// ✅ BETTER: Overload (preferred for functions)
void process(int* ptr) {
    std::cout << "Overload: " << *ptr << '\n';
}

int x = 42;
process(&x);  // Calls the overload, not the specialization!
```

**Key Insight:** For **function templates**, prefer **overloading** over specialization!

#### Rule 3: Partial Specialization Not Allowed for Functions
```cpp
template<typename T, typename U>
void func(T, U) { }

// ❌ ERROR: Can't partially specialize function templates
template<typename T>
void func<T, int>(T, int) { }

// ✅ SOLUTION: Use overloading
template<typename T>
void func(T, int) { }
```

---

## 2. Template Argument Deduction

### 2.1 Basic Deduction Rules

**Deduction happens** when the compiler determines template parameters from function arguments.

```cpp
template<typename T>
void func(T param);

int x = 42;
func(x);  // T deduced as 'int'
```

### 2.2 Deduction with Different Parameter Types

#### Case 1: Pass by Value (T)
```cpp
template<typename T>
void func(T param);

int x = 42;
const int cx = x;
const int& rx = x;

func(x);   // T = int,       param is int
func(cx);  // T = int,       param is int (const removed)
func(rx);  // T = int,       param is int (const & ref removed)
```

**Rule:** Top-level `const` and references are **ignored** for pass-by-value.

```cpp
const char* const ptr = "hello";
func(ptr);  // T = const char* (top-level const removed)
            // ptr still points to const char
```

#### Case 2: Pass by Reference (T&)
```cpp
template<typename T>
void func(T& param);

int x = 42;
const int cx = x;
const int& rx = x;

func(x);   // T = int,       param is int&
func(cx);  // T = const int, param is const int&
func(rx);  // T = const int, param is const int&
```

**Rule:** `const` and references are **preserved** but reference is not part of T.

#### Case 3: Pass by Pointer (T*)
```cpp
template<typename T>
void func(T* param);

int x = 42;
const int* px = &x;

func(&x);  // T = int,       param is int*
func(px);  // T = const int, param is const int*
```

#### Case 4: Universal/Forwarding References (T&&)
```cpp
template<typename T>
void func(T&& param);

int x = 42;
const int cx = x;

func(x);   // x is lvalue → T = int&,       param is int&
func(cx);  // cx is lvalue → T = const int&, param is const int&
func(42);  // 42 is rvalue → T = int,        param is int&&
```

**Special Rule:** For `T&&`:
- **Lvalue** argument → T deduced as lvalue reference
- **Rvalue** argument → T deduced as non-reference

### 2.3 Array and Function Decay

#### Arrays Decay to Pointers
```cpp
template<typename T>
void func1(T param);

template<typename T>
void func2(T& param);

const char name[] = "Hello";

func1(name);  // T = const char* (array decays to pointer)
func2(name);  // T = const char[6] (no decay, param is reference)
```

**Deducing Array Size:**
```cpp
// Deduce array size!
template<typename T, size_t N>
constexpr size_t arraySize(T (&)[N]) noexcept {
    return N;
}

int arr[10];
constexpr auto size = arraySize(arr);  // size = 10
```

#### Functions Decay to Function Pointers
```cpp
void someFunc(int);

template<typename T>
void func1(T param);

template<typename T>
void func2(T& param);

func1(someFunc);  // T = void(*)(int) - function pointer
func2(someFunc);  // T = void(int) - function type
```

### 2.4 Non-Deduced Contexts

Some contexts prevent deduction:

```cpp
template<typename T>
void func(typename std::vector<T>::iterator it);

std::vector<int> vec;
func(vec.begin());  // ❌ ERROR: Can't deduce T
                    // Must explicitly specify: func<int>(vec.begin())
```

**Why?** Compiler would need to examine `iterator` to find T, which is too complex.

**Common Non-Deduced Contexts:**
- Nested type names (`typename T::type`)
- Template template parameters
- Parameters that don't directly use T

---

## 3. Class Template Argument Deduction (CTAD) (C++17+)

### 3.1 Basic CTAD

Before C++17:
```cpp
std::pair<int, double> p{1, 3.14};  // Must specify types
std::vector<int> v{1, 2, 3};        // Must specify type
```

C++17 with CTAD:
```cpp
std::pair p{1, 3.14};       // Deduced: pair<int, double>
std::vector v{1, 2, 3};     // Deduced: vector<int>
std::array arr{1, 2, 3, 4}; // Deduced: array<int, 4>
```

### 3.2 How CTAD Works

Compiler creates fictional function templates from constructors:

```cpp
template<typename T>
class MyClass {
public:
    MyClass(T value);
};

// Compiler creates fictional function:
// template<typename T>
// MyClass<T> DeductionGuide(T value);

MyClass obj(42);  // Deduces T = int, creates MyClass<int>
```

### 3.3 Deduction Guides (Custom CTAD)

When automatic deduction doesn't work, provide hints:

```cpp
// Class with ambiguous constructor
template<typename T>
class Container {
    T* data;
    size_t size;
public:
    // Constructor from iterators
    template<typename Iter>
    Container(Iter begin, Iter end);
};

// Deduction guide
template<typename Iter>
Container(Iter, Iter) 
    -> Container<typename std::iterator_traits<Iter>::value_type>;

// Now works!
std::vector<int> vec{1, 2, 3};
Container c(vec.begin(), vec.end());  // Deduced: Container<int>
```

### 3.4 Real-World Example: std::array

```cpp
// std::array requires size as template parameter
// But with deduction guide:

template<typename T, typename... U>
array(T, U...) -> array<T, 1 + sizeof...(U)>;

// Now we can write:
std::array arr{1, 2, 3, 4, 5};  // Deduced: array<int, 5>
```

### 3.5 Aggregate Deduction (C++20)

```cpp
template<typename T>
struct Point {
    T x, y;  // Aggregate
};

// C++20: Works without deduction guide!
Point p{1.5, 2.5};  // Deduced: Point<double>
```

### 3.6 CTAD Gotchas

#### Gotcha 1: Copy Construction
```cpp
std::vector v1{1, 2, 3};         // vector<int>
std::vector v2{v1};              // ❌ NOT vector<int>!
                                 // Deduced: vector<vector<int>>!

std::vector v3(v1);              // ✅ OK: Copy constructor
                                 // vector<int>
```

#### Gotcha 2: Partial Deduction Not Allowed
```cpp
std::tuple t{1, 2.5};           // ✅ OK: Full deduction

std::tuple<int> t2{1, 2.5};     // ❌ ERROR: Can't partially deduce
std::tuple<int, double> t3{1, 2.5}; // ✅ OK: Fully specified
```

---

## 4. Forwarding References & Perfect Forwarding

### 4.1 What Are Forwarding References?

**Also called:** Universal References (term by Scott Meyers)

```cpp
template<typename T>
void func(T&& param);  // This is a forwarding reference!

// But only in a deduced context:
template<typename T>
class MyClass {
    void method(T&& param);  // This is an rvalue reference!
                            // (T is already known)
};
```

**Key Rule:** `T&&` is a forwarding reference ONLY if:
1. T is a template parameter being deduced
2. No cv-qualifiers on T (no const, volatile)

### 4.2 Reference Collapsing Rules

When references are formed from references:
```cpp
T&  &   →  T&      // lvalue ref to lvalue ref → lvalue ref
T&  &&  →  T&      // rvalue ref to lvalue ref → lvalue ref
T&& &   →  T&      // lvalue ref to rvalue ref → lvalue ref
T&& &&  →  T&&     // rvalue ref to rvalue ref → rvalue ref
```

**In Practice:**
```cpp
template<typename T>
void func(T&& param);

int x = 42;
func(x);   // T = int&, param is int& && → int&
func(42);  // T = int,  param is int&&
```

### 4.3 Perfect Forwarding

**Problem:** Preserve value category (lvalue/rvalue) when passing to another function.

```cpp
// ❌ WRONG: Loses rvalue-ness
template<typename T>
void wrapper(T&& arg) {
    someFunc(arg);  // arg is always an lvalue here!
}

// ✅ CORRECT: Perfect forwarding
template<typename T>
void wrapper(T&& arg) {
    someFunc(std::forward<T>(arg));  // Preserves value category
}
```

**How std::forward Works:**
```cpp
// Conceptual implementation
template<typename T>
T&& forward(remove_reference_t<T>& arg) {
    return static_cast<T&&>(arg);
}

// If T = int (rvalue): returns int&&
// If T = int& (lvalue): returns int& && → int&
```

### 4.4 Complete Perfect Forwarding Example

```cpp
#include <utility>
#include <iostream>

class Widget {
public:
    Widget() { std::cout << "Default ctor\n"; }
    Widget(const Widget&) { std::cout << "Copy ctor\n"; }
    Widget(Widget&&) { std::cout << "Move ctor\n"; }
};

void process(Widget& w) { std::cout << "Lvalue overload\n"; }
void process(Widget&& w) { std::cout << "Rvalue overload\n"; }

template<typename T>
void wrapper(T&& arg) {
    process(std::forward<T>(arg));  // Perfect forwarding
}

int main() {
    Widget w;
    wrapper(w);              // Calls lvalue overload
    wrapper(Widget{});       // Calls rvalue overload
    wrapper(std::move(w));   // Calls rvalue overload
}
```

### 4.5 Variadic Perfect Forwarding

```cpp
// Forward arbitrary number of arguments
template<typename... Args>
auto make_unique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

// Usage
auto ptr = make_unique<Widget>(arg1, arg2, arg3);
// All arguments perfectly forwarded to Widget constructor
```

---

## Key Takeaways

### Specialization
✅ Full specialization: Complete custom implementation
✅ Partial specialization: Only for class templates
✅ Prefer overloading for function templates
✅ Declare specializations before first use

### Deduction
✅ Pass-by-value: Ignores top-level const and references
✅ Pass-by-reference: Preserves const
✅ Arrays and functions decay (usually)
✅ Universal references with T&& enable perfect forwarding

### CTAD (C++17+)
✅ Automatically deduce class template arguments
✅ Use deduction guides for complex cases
✅ Watch out for copy constructor issues
✅ Partial deduction not allowed

### Perfect Forwarding
✅ Use T&& for forwarding references
✅ Always use std::forward<T>() when forwarding
✅ Preserves value category (lvalue vs rvalue)
✅ Essential for generic wrapper functions

---

# C++ Templates: Complete Guide - Part 3
## Variadic Templates, SFINAE, and Advanced Techniques

---

## Table of Contents (Part 3)
1. Variadic Templates & Parameter Packs
2. SFINAE (Substitution Failure Is Not An Error)
3. Type Traits & Meta-programming
4. Concepts (C++20)
5. Advanced Topics

---

## 1. Variadic Templates & Parameter Packs (C++11+)

### 1.1 What Are Parameter Packs?

**Parameter packs** allow templates to accept any number of template arguments.

```cpp
// Can take 0, 1, 2, or any number of types
template<typename... Args>  // Args is a template parameter pack
class Tuple { };

Tuple<> t0;                    // 0 arguments
Tuple<int> t1;                 // 1 argument
Tuple<int, double, char> t2;   // 3 arguments
```

### 1.2 Basic Syntax

```cpp
template<typename... Types>     // Types is a template parameter pack
void func(Types... args) {      // args is a function parameter pack
    // ...
}

// Usage
func(1, 2.5, "hello");  // Types = {int, double, const char*}
                        // args = {1, 2.5, "hello"}
```

**Key Operators:**
- `typename...` or `class...` - declares a template parameter pack
- `...` after name - declares a function parameter pack
- `...` after expression - expands the pack

### 1.3 Getting Pack Size

```cpp
template<typename... Args>
void func(Args... args) {
    std::cout << "Number of arguments: " << sizeof...(Args) << '\n';
    std::cout << "Number of parameters: " << sizeof...(args) << '\n';
}

func(1, 2, 3);        // Prints: 3, 3
func();               // Prints: 0, 0
func("a", "b", "c", "d"); // Prints: 4, 4
```

### 1.4 Pack Expansion

**Basic Pattern:** `pattern...`

```cpp
template<typename... Args>
void printAll(Args... args) {
    // Expand pack in initializer list
    int dummy[] = { (std::cout << args << ' ', 0)... };
    
    // Expands to:
    // { (cout << arg1 << ' ', 0), (cout << arg2 << ' ', 0), ... }
}

printAll(1, 2.5, "hello");  // Output: 1 2.5 hello
```

### 1.5 Recursive Template Approach

Classic way to process parameter packs:

```cpp
#include <iostream>

// Base case: no arguments
void print() {
    std::cout << '\n';
}

// Recursive case: process first, recurse on rest
template<typename T, typename... Args>
void print(T first, Args... rest) {
    std::cout << first << ' ';
    print(rest...);  // Recursive call with remaining arguments
}

// Usage
print(1, 2.5, "hello", 'x');  // Output: 1 2.5 hello x
```

**How it works:**
```cpp
print(1, 2.5, "hello")
→ print(1) + print(2.5, "hello")
→ print(1) + print(2.5) + print("hello")
→ print(1) + print(2.5) + print("hello") + print()
```

### 1.6 Fold Expressions (C++17+)

**Much simpler** than recursion for many cases!

```cpp
// Sum all arguments
template<typename... Args>
auto sum(Args... args) {
    return (args + ...);  // Fold expression!
}

auto result = sum(1, 2, 3, 4, 5);  // 15
```

**Four Forms:**

```cpp
template<typename... Args>
void example(Args... args) {
    // 1. Unary right fold: (args op ...)
    auto r1 = (args + ...);  // a1 + (a2 + (a3 + a4))
    
    // 2. Unary left fold: (... op args)
    auto r2 = (... + args);  // ((a1 + a2) + a3) + a4
    
    // 3. Binary right fold: (args op ... op init)
    auto r3 = (args + ... + 0);  // a1 + (a2 + (a3 + (a4 + 0)))
    
    // 4. Binary left fold: (init op ... op args)
    auto r4 = (0 + ... + args);  // (((0 + a1) + a2) + a3) + a4
}
```

**Practical Examples:**

```cpp
// Print all arguments
template<typename... Args>
void print(Args... args) {
    ((std::cout << args << ' '), ...);  // Left fold with comma operator
    std::cout << '\n';
}

// Check if all arguments are positive
template<typename... Args>
bool all_positive(Args... args) {
    return ((args > 0) && ...);  // Fold with &&
}

// Push all arguments to vector
template<typename T, typename... Args>
void push_all(std::vector<T>& vec, Args... args) {
    (vec.push_back(args), ...);
}
```

### 1.7 Pack Expansion in Different Contexts

```cpp
template<typename... Args>
class MultipleInheritance : public Args... {
    // Inherits from all types in Args
};

template<typename... Args>
void func() {
    // Call function for each type
    (processType<Args>(), ...);
    
    // Create tuple
    std::tuple<Args...> tup;
    
    // Create array of sizes
    size_t sizes[] = { sizeof(Args)... };
    
    // Function call with expanded pack
    someFunction(Args{}...);  // Calls with default-constructed Args
}
```

### 1.8 Variadic Class Template

```cpp
template<typename... Types>
class Tuple;  // Forward declaration

// Base case: empty tuple
template<>
class Tuple<> {
};

// Recursive case
template<typename Head, typename... Tail>
class Tuple<Head, Tail...> : private Tuple<Tail...> {
    Head value;
public:
    Tuple(Head h, Tail... t) 
        : Tuple<Tail...>(t...), value(h) {}
    
    Head& head() { return value; }
    Tuple<Tail...>& tail() { return *this; }
};

// Usage
Tuple<int, double, std::string> t(42, 3.14, "hello");
```

### 1.9 Perfect Forwarding with Variadic Templates

```cpp
// Forward arbitrary number of arguments
template<typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

// Usage
auto ptr = make_unique<Widget>(arg1, arg2, arg3);
```

---

## 2. SFINAE (Substitution Failure Is Not An Error)

### 2.1 What is SFINAE?

**SFINAE** is a C++ rule: when template substitution fails, it's not an error—just remove that template from the overload set.

```cpp
// Template that only works for types with a .size() method
template<typename T>
auto getSize(T& container) -> decltype(container.size()) {
    return container.size();
}

// Fallback for types without .size()
int getSize(...) {
    return -1;
}

std::vector<int> vec{1, 2, 3};
int x = 42;

getSize(vec);  // Calls first overload: returns 3
getSize(x);    // First fails (int has no .size()), calls second: returns -1
```

### 2.2 How SFINAE Works

**Process:**
1. Compiler attempts template substitution
2. If substitution would create invalid code → remove from overload set
3. Try next candidate
4. If no valid candidate → error

**Important:** Only substitution failures in the **immediate context** are SFINAE errors.

```cpp
template<typename T>
typename T::type func(T);  // SFINAE: T::type in immediate context

template<typename T>
void func(T) {
    typename T::type x;  // NOT SFINAE: inside function body
}
```

### 2.3 std::enable_if

The standard way to use SFINAE:

```cpp
// Only enable for integral types
template<typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
increment(T value) {
    return value + 1;
}

// Only enable for floating-point types
template<typename T>
typename std::enable_if<std::is_floating_point<T>::value, T>::type
increment(T value) {
    return value + 0.1;
}

increment(5);      // Calls first: returns 6
increment(5.0);    // Calls second: returns 5.1
// increment("hi");  // ERROR: No matching overload
```

**C++14 Shorthand:**
```cpp
template<typename T>
std::enable_if_t<std::is_integral_v<T>, T>
increment(T value) {
    return value + 1;
}
```

### 2.4 SFINAE with return type (C++11+)

```cpp
// Using trailing return type
template<typename T>
auto getSize(T& container) -> decltype(container.size()) {
    return container.size();
}

// C++14: Can use auto directly
template<typename T>
auto getSize(T& container) {
    return container.size();  // SFINAE if .size() doesn't exist
}
```

### 2.5 Detecting Member Functions/Types

```cpp
// Check if type has a member typedef
template<typename T>
struct has_value_type {
private:
    template<typename U>
    static std::true_type test(typename U::value_type*);
    
    template<typename>
    static std::false_type test(...);
    
public:
    static constexpr bool value = decltype(test<T>(nullptr))::value;
};

// Usage
static_assert(has_value_type<std::vector<int>>::value);  // true
static_assert(!has_value_type<int>::value);              // true
```

### 2.6 Expression SFINAE

Check if an expression is valid:

```cpp
// Check if two types can be added
template<typename T, typename U>
struct is_addable {
private:
    template<typename T2, typename U2>
    static auto test(int) 
        -> decltype(std::declval<T2>() + std::declval<U2>(), std::true_type{});
    
    template<typename, typename>
    static std::false_type test(...);
    
public:
    static constexpr bool value = decltype(test<T, U>(0))::value;
};

static_assert(is_addable<int, int>::value);          // true
static_assert(!is_addable<int, std::string>::value); // true (can't add)
```

### 2.7 Common SFINAE Patterns

```cpp
// 1. Return type SFINAE
template<typename T>
auto func(T x) -> decltype(x.foo(), void()) {
    x.foo();
}

// 2. Template parameter SFINAE
template<typename T, 
         typename = std::enable_if_t<condition>>
void func(T x);

// 3. Non-type parameter SFINAE
template<typename T, 
         std::enable_if_t<condition, int> = 0>
void func(T x);

// 4. Default argument SFINAE
template<typename T>
void func(T x, std::enable_if_t<condition>* = nullptr);
```

---

## 3. Type Traits & Metaprogramming

### 3.1 Standard Type Traits

```cpp
#include <type_traits>

// Primary categories
std::is_integral<int>::value        // true
std::is_floating_point<double>::value // true
std::is_pointer<int*>::value        // true
std::is_array<int[10]>::value       // true
std::is_class<std::string>::value   // true

// Composite categories
std::is_arithmetic<int>::value      // true (integral or floating)
std::is_fundamental<int>::value     // true (arithmetic or void)
std::is_object<int>::value          // true (not function/ref/void)

// C++17: _v suffix
std::is_integral_v<int>             // true (no need for ::value)
```

### 3.2 Type Modifications

```cpp
// Remove qualifiers
std::remove_const<const int>::type           // int
std::remove_reference<int&>::type            // int
std::remove_pointer<int*>::type              // int
std::remove_cv<const volatile int>::type     // int

// Add qualifiers
std::add_const<int>::type                    // const int
std::add_lvalue_reference<int>::type         // int&
std::add_pointer<int>::type                  // int*

// C++14: _t suffix
std::remove_const_t<const int>               // int
```

### 3.3 Type Relationships

```cpp
std::is_same<int, int>::value                // true
std::is_same<int, int32_t>::value            // true
std::is_same<int, long>::value               // false

std::is_base_of<Base, Derived>::value        // true
std::is_convertible<Derived*, Base*>::value  // true
```

### 3.4 Compile-Time Conditionals

```cpp
// std::conditional: compile-time if-else
template<typename T>
using MyType = std::conditional_t<
    std::is_integral_v<T>,
    long long,          // Use if integral
    double              // Use otherwise
>;

MyType<int> x;     // x is long long
MyType<float> y;   // y is double
```

### 3.5 Custom Type Traits

```cpp
// Check if type is container
template<typename T, typename = void>
struct is_container : std::false_type {};

template<typename T>
struct is_container<T, std::void_t<
    typename T::value_type,
    typename T::iterator,
    decltype(std::declval<T>().begin()),
    decltype(std::declval<T>().end())
>> : std::true_type {};

// Usage
static_assert(is_container<std::vector<int>>::value);  // true
static_assert(!is_container<int>::value);              // true
```

### 3.6 if constexpr (C++17)

Compile-time if statement:

```cpp
template<typename T>
void process(T value) {
    if constexpr (std::is_integral_v<T>) {
        // Only compiled if T is integral
        std::cout << "Integer: " << value << '\n';
    } else if constexpr (std::is_floating_point_v<T>) {
        // Only compiled if T is floating-point
        std::cout << "Float: " << value << '\n';
    } else {
        // Otherwise
        std::cout << "Other: " << value << '\n';
    }
}

process(42);      // Compiles only first branch
process(3.14);    // Compiles only second branch
process("hi");    // Compiles only third branch
```

**Replaces complex SFINAE:**
```cpp
// Before C++17: Need two functions
template<typename T>
std::enable_if_t<std::is_integral_v<T>>
func(T x) { /* ... */ }

template<typename T>
std::enable_if_t<!std::is_integral_v<T>>
func(T x) { /* ... */ }

// C++17: One function
template<typename T>
void func(T x) {
    if constexpr (std::is_integral_v<T>) {
        // ...
    } else {
        // ...
    }
}
```

---

## 4. Concepts (C++20)

### 4.1 What Are Concepts?

**Concepts** are named compile-time predicates on template parameters.

```cpp
// Define a concept
template<typename T>
concept Integral = std::is_integral_v<T>;

// Use in template
template<Integral T>
T increment(T value) {
    return value + 1;
}

increment(5);      // ✅ OK: int satisfies Integral
// increment(5.0);  // ❌ Error: double doesn't satisfy Integral
```

### 4.2 Basic Concept Syntax

```cpp
// Four ways to use concepts:

// 1. Requires clause
template<typename T> requires Integral<T>
T func(T x);

// 2. Constrained template parameter
template<Integral T>
T func(T x);

// 3. Abbreviated function template (C++20)
auto func(Integral auto x);

// 4. Trailing requires clause
template<typename T>
T func(T x) requires Integral<T>;
```

### 4.3 Standard Concepts

```cpp
#include <concepts>

// Type categories
std::integral<int>              // true
std::floating_point<double>     // true
std::signed_integral<int>       // true

// Comparisons
std::equality_comparable<int>
std::totally_ordered<double>

// Conversions
std::convertible_to<int, double>
std::same_as<int, int>

// Callables
std::invocable<Func, Args...>
std::predicate<Func, Args...>
```

### 4.4 Defining Custom Concepts

```cpp
// Simple concept
template<typename T>
concept Numeric = std::is_arithmetic_v<T>;

// Compound concept
template<typename T>
concept Addable = requires(T a, T b) {
    a + b;              // Must support +
};

// Complex concept
template<typename T>
concept Container = requires(T c) {
    typename T::value_type;     // Must have value_type
    typename T::iterator;       // Must have iterator
    { c.begin() } -> std::same_as<typename T::iterator>;
    { c.end() } -> std::same_as<typename T::iterator>;
    { c.size() } -> std::convertible_to<size_t>;
};
```

### 4.5 Requires Expression

```cpp
template<typename T>
concept Incrementable = requires(T x) {
    ++x;                // Simple requirement
    x++;                // Simple requirement
    { ++x } -> std::same_as<T&>;  // Type requirement
};

template<typename T>
concept HasSize = requires(T container) {
    { container.size() } noexcept -> std::convertible_to<size_t>;
    // Must be noexcept and return something convertible to size_t
};
```

### 4.6 Concept Composition

```cpp
template<typename T>
concept SignedIntegral = std::integral<T> && std::signed_integral<T>;

template<typename T>
concept NumericOrString = Numeric<T> || std::same_as<T, std::string>;

// Use in constraints
template<typename T>
requires SignedIntegral<T>
T absolute(T value) {
    return value < 0 ? -value : value;
}
```

### 4.7 Concepts vs SFINAE

**Before C++20 (SFINAE):**
```cpp
template<typename T>
std::enable_if_t<std::is_integral_v<T>, T>
increment(T value) {
    return value + 1;
}
```

**C++20 (Concepts):**
```cpp
template<std::integral T>
T increment(T value) {
    return value + 1;
}

// Or even simpler:
auto increment(std::integral auto value) {
    return value + 1;
}
```

**Benefits:**
- ✅ Clearer syntax
- ✅ Better error messages
- ✅ Can be composed easily
- ✅ Documented in function signature

---

## 5. Advanced Topics

### 5.1 Template Aliases (C++11)

```cpp
// Type alias
template<typename T>
using Vec = std::vector<T>;

Vec<int> v{1, 2, 3};  // Same as std::vector<int>

// Partial specialization via alias
template<typename T>
using StringMap = std::map<std::string, T>;

StringMap<int> ages;  // std::map<std::string, int>
```

### 5.2 Variable Templates (C++14)

```cpp
template<typename T>
constexpr T pi = T(3.1415926535897932385);

float f = pi<float>;
double d = pi<double>;

// With type deduction
template<typename T>
T circleArea(T radius) {
    return pi<T> * radius * radius;
}
```

### 5.3 Template Template Parameters

```cpp
// Accept any template that takes one type parameter
template<typename T, template<typename> class Container>
class Stack {
    Container<T> data;
public:
    void push(T value) { data.push_back(value); }
};

// Works with any single-type container template
Stack<int, std::vector> s1;
Stack<int, std::deque> s2;
Stack<int, std::list> s3;
```

### 5.4 Member Templates

```cpp
template<typename T>
class MyClass {
public:
    // Member function template
    template<typename U>
    void convert(const MyClass<U>& other) {
        // ...
    }
    
    // Member class template
    template<typename U>
    struct Inner {
        U value;
    };
};

MyClass<int> obj1;
MyClass<double> obj2;
obj1.convert(obj2);  // U deduced as double
```

### 5.5 Dependent Names

```cpp
template<typename T>
class MyClass {
    typename T::value_type x;  // 'typename' needed for dependent type
    
    template<typename U>
    void func() {
        T::template innerTemplate<U>();  // 'template' needed
    }
};
```

---

## Summary: When to Use What

| Feature | C++ Version | Use When |
|---------|-------------|----------|
| Basic Templates | C++98 | Always - foundation |
| Variadic Templates | C++11 | Variable number of arguments |
| Alias Templates | C++11 | Simplify complex types |
| Variable Templates | C++14 | Compile-time constants |
| Fold Expressions | C++17 | Processing parameter packs |
| if constexpr | C++17 | Compile-time conditionals |
| CTAD | C++17 | Avoid repeating type names |
| Concepts | C++20 | Constrain templates clearly |
| SFINAE | Any | When concepts not available |

---

# C++ Templates: Complete Guide - Part 4
## Best Practices, Design Patterns & Real-World Applications

---

## Table of Contents (Part 4)
1. Template Design Patterns
2. Compilation Model & Organization
3. Performance Considerations
4. Error Messages & Debugging
5. Best Practices & Common Pitfalls
6. Real-World Examples

---

## 1. Template Design Patterns

### 1.1 Curiously Recurring Template Pattern (CRTP)

**Pattern:** Class inherits from a template instantiated with itself.

```cpp
// Base template
template<typename Derived>
class Base {
public:
    void interface() {
        // Static polymorphism - no virtual functions!
        static_cast<Derived*>(this)->implementation();
    }
    
    void commonFunction() {
        std::cout << "Common behavior\n";
    }
};

// Derived classes
class Derived1 : public Base<Derived1> {
public:
    void implementation() {
        std::cout << "Derived1 implementation\n";
    }
};

class Derived2 : public Base<Derived2> {
public:
    void implementation() {
        std::cout << "Derived2 implementation\n";
    }
};

// Usage
template<typename T>
void process(Base<T>& obj) {
    obj.interface();  // Calls correct implementation at compile-time!
}

Derived1 d1;
Derived2 d2;
process(d1);  // "Derived1 implementation"
process(d2);  // "Derived2 implementation"
```

**Benefits:**
- ✅ Static polymorphism (no virtual function overhead)
- ✅ Compile-time optimization
- ✅ Type-safe

**Real-World Use:** `std::enable_shared_from_this<T>`

### 1.2 Policy-Based Design

**Pattern:** Configure behavior through template parameters.

```cpp
// Storage policies
template<typename T>
class HeapStorage {
    T* data;
public:
    HeapStorage(T value) : data(new T(value)) {}
    ~HeapStorage() { delete data; }
    T& get() { return *data; }
};

template<typename T>
class StackStorage {
    T data;
public:
    StackStorage(T value) : data(value) {}
    T& get() { return data; }
};

// Smart pointer with configurable storage policy
template<typename T, template<typename> class StoragePolicy>
class SmartPtr : private StoragePolicy<T> {
public:
    SmartPtr(T value) : StoragePolicy<T>(value) {}
    
    T& operator*() { return this->get(); }
    T* operator->() { return &this->get(); }
};

// Usage
SmartPtr<int, HeapStorage> heapPtr(42);
SmartPtr<int, StackStorage> stackPtr(42);
```

**Benefits:**
- ✅ Highly configurable
- ✅ Mix and match policies
- ✅ Zero runtime overhead

### 1.3 Type Erasure

**Pattern:** Hide template parameters behind a non-template interface.

```cpp
// Non-template interface
class Any {
    struct HolderBase {
        virtual ~HolderBase() = default;
        virtual HolderBase* clone() const = 0;
    };
    
    template<typename T>
    struct Holder : HolderBase {
        T value;
        Holder(T v) : value(std::move(v)) {}
        HolderBase* clone() const override {
            return new Holder(value);
        }
    };
    
    HolderBase* ptr = nullptr;
    
public:
    template<typename T>
    Any(T value) : ptr(new Holder<T>(std::move(value))) {}
    
    Any(const Any& other) : ptr(other.ptr ? other.ptr->clone() : nullptr) {}
    
    ~Any() { delete ptr; }
    
    template<typename T>
    T& get() {
        return static_cast<Holder<T>*>(ptr)->value;
    }
};

// Usage
std::vector<Any> vec;
vec.push_back(42);           // Stores int
vec.push_back(std::string("hello"));  // Stores string
vec.push_back(3.14);         // Stores double
```

**Real-World Use:** `std::any`, `std::function`

### 1.4 Mixin Pattern

**Pattern:** Add functionality through multiple inheritance.

```cpp
// Mixin classes
template<typename Derived>
struct Printable {
    void print() const {
        std::cout << "Printing: " 
                  << static_cast<const Derived*>(this)->getValue() 
                  << '\n';
    }
};

template<typename Derived>
struct Serializable {
    std::string serialize() const {
        return std::to_string(static_cast<const Derived*>(this)->getValue());
    }
};

// Combine mixins
class Value : public Printable<Value>, 
              public Serializable<Value> {
    int value;
public:
    Value(int v) : value(v) {}
    int getValue() const { return value; }
};

// Usage
Value v(42);
v.print();                      // From Printable
std::string s = v.serialize();  // From Serializable
```

### 1.5 Tag Dispatching

**Pattern:** Select implementation based on type traits.

```cpp
// Tags
struct random_access_iterator_tag {};
struct bidirectional_iterator_tag {};

// Implementation for random access iterators
template<typename Iterator>
void advanceImpl(Iterator& it, int n, random_access_iterator_tag) {
    it += n;  // O(1) - direct jump
}

// Implementation for bidirectional iterators
template<typename Iterator>
void advanceImpl(Iterator& it, int n, bidirectional_iterator_tag) {
    while (n--) ++it;  // O(n) - incremental
}

// Public interface with dispatch
template<typename Iterator>
void advance(Iterator& it, int n) {
    advanceImpl(it, n, 
                typename std::iterator_traits<Iterator>::iterator_category{});
}

// Usage
std::vector<int>::iterator vit;
advance(vit, 10);  // Uses random_access version

std::list<int>::iterator lit;
advance(lit, 10);  // Uses bidirectional version
```

---

## 2. Compilation Model & Organization

### 2.1 The Inclusion Model (Most Common)

**Rule:** Template definitions must be in headers.

```cpp
// ===== MyTemplate.h =====
#ifndef MYTEMPLATE_H
#define MYTEMPLATE_H

template<typename T>
class MyTemplate {
public:
    void method();
    T compute(T value);
};

// Definitions in SAME header file
template<typename T>
void MyTemplate<T>::method() {
    // implementation
}

template<typename T>
T MyTemplate<T>::compute(T value) {
    return value * 2;
}

#endif
```

**Why?**
- Compiler needs full definition to instantiate
- Each translation unit instantiates independently
- Linker merges duplicate instantiations

### 2.2 Explicit Instantiation (For Known Types)

**Reduce compilation time** by instantiating in .cpp file:

```cpp
// ===== MyTemplate.h =====
template<typename T>
class MyTemplate {
public:
    void method();
};

// Declaration only in header
template<typename T>
void MyTemplate<T>::method();

// ===== MyTemplate.cpp =====
#include "MyTemplate.h"

// Full definition in .cpp
template<typename T>
void MyTemplate<T>::method() {
    // implementation
}

// Explicit instantiations for known types
template class MyTemplate<int>;
template class MyTemplate<double>;
template class MyTemplate<std::string>;

// ===== main.cpp =====
#include "MyTemplate.h"

// Prevent instantiation here
extern template class MyTemplate<int>;
extern template class MyTemplate<double>;

int main() {
    MyTemplate<int> obj;  // Uses instantiation from MyTemplate.cpp
    obj.method();
}
```

**Benefits:**
- ✅ Faster compilation (no repeated instantiation)
- ✅ Smaller object files
- ❌ Only works for predetermined types

### 2.3 Separating Interface and Implementation

```cpp
// ===== MyTemplate.h =====
template<typename T>
class MyTemplate {
public:
    void method();
};

// Declare that implementation is elsewhere
#include "MyTemplate.impl.h"

// ===== MyTemplate.impl.h =====
template<typename T>
void MyTemplate<T>::method() {
    // implementation
}
```

### 2.4 Module System (C++20)

```cpp
// ===== mymodule.ixx (module interface) =====
export module mymodule;

export template<typename T>
class MyTemplate {
public:
    void method() {
        // Can define here or in module implementation
    }
};

// ===== main.cpp =====
import mymodule;

int main() {
    MyTemplate<int> obj;
}
```

**Benefits:**
- ✅ Faster compilation
- ✅ Better encapsulation
- ✅ No header guards needed

---

## 3. Performance Considerations

### 3.1 Code Bloat

**Problem:** Each template instantiation creates new code.

```cpp
template<typename T>
void sort(T* array, size_t size) {
    // 100 lines of sorting code
}

// Each call instantiates new code!
sort(intArray, 10);        // ~100 lines for int
sort(doubleArray, 20);     // ~100 lines for double
sort(stringArray, 5);      // ~100 lines for string
// ... potentially massive binary size
```

**Solution 1: Factor out common code**
```cpp
// Non-template implementation
void sortImpl(void* array, size_t size, size_t elemSize,
              bool (*compare)(const void*, const void*)) {
    // Sorting logic using void pointers
}

// Thin template wrapper
template<typename T>
void sort(T* array, size_t size) {
    sortImpl(array, size, sizeof(T),
             [](const void* a, const void* b) {
                 return *static_cast<const T*>(a) < *static_cast<const T*>(b);
             });
}
```

**Solution 2: Use inheritance**
```cpp
// Non-template base with common functionality
class VectorBase {
protected:
    void* data;
    size_t size, capacity;
    void resize(size_t elemSize);
    void reserve(size_t elemSize);
    // Common algorithms
};

// Template derived class
template<typename T>
class Vector : private VectorBase {
public:
    T& operator[](size_t i) { return static_cast<T*>(data)[i]; }
    void push_back(const T& value) {
        if (size == capacity) reserve(sizeof(T));
        new (static_cast<T*>(data) + size++) T(value);
    }
};
```

### 3.2 Inline and Optimization

```cpp
// Templates are implicitly inline
template<typename T>
T square(T x) {  // Automatically inline candidate
    return x * x;
}

// Compiler can fully optimize:
int result = square(5);  // Often optimized to: int result = 25;
```

### 3.3 Compile-Time vs Runtime

```cpp
// ❌ BAD: Runtime check
template<typename T>
T compute(T value) {
    if (std::is_floating_point<T>::value) {  // Runtime check!
        return value * 1.1;
    }
    return value + 1;
}

// ✅ GOOD: Compile-time selection
template<typename T>
T compute(T value) {
    if constexpr (std::is_floating_point_v<T>) {  // Compile-time!
        return value * 1.1;
    }
    return value + 1;
}

// Even better: Overloading
template<typename T>
std::enable_if_t<std::is_floating_point_v<T>, T>
compute(T value) { return value * 1.1; }

template<typename T>
std::enable_if_t<std::is_integral_v<T>, T>
compute(T value) { return value + 1; }
```

---

## 4. Error Messages & Debugging

### 4.1 Understanding Template Error Messages

**Typical error:**
```
error: no matching function for call to 'std::vector<int>::push_back(std::string)'
  in instantiation of function template specialization 'func<int>' requested here
    note: candidate function not viable: no known conversion from 'std::string' to 'const int &'
```

**Reading strategy:**
1. Find the **root cause** (usually at the bottom)
2. Trace back through **instantiation stack**
3. Look for **constraint violations**

### 4.2 Static Assertions

```cpp
template<typename T>
void process(T value) {
    static_assert(std::is_arithmetic_v<T>, 
                  "process() requires arithmetic type");
    // ...
}

process(42);      // ✅ OK
process("hi");    // ❌ Clear error message
```

### 4.3 Concepts for Better Errors (C++20)

```cpp
// ❌ Without concepts: cryptic errors
template<typename T>
void sort(T& container) {
    std::sort(container.begin(), container.end());
}

sort(42);  // Huge error about T::begin() not found

// ✅ With concepts: clear errors
template<typename T>
concept Container = requires(T c) {
    c.begin();
    c.end();
};

template<Container T>
void sort(T& container) {
    std::sort(container.begin(), container.end());
}

sort(42);  // Error: 'int' does not satisfy 'Container'
```

### 4.4 Debugging Techniques

**1. Type Printing:**
```cpp
// Show deduced type as compile error
template<typename T>
struct ShowType;

int x;
ShowType<decltype(x)> dummy;  // Error shows: ShowType<int>
```

**2. Compiler Type Display:**
```cpp
#include <boost/type_index.hpp>

template<typename T>
void debug(T value) {
    std::cout << boost::typeindex::type_id_with_cvr<T>().pretty_name() 
              << '\n';
}
```

**3. Explicit Instantiation:**
```cpp
// Force instantiation to see errors early
template class MyTemplate<int>;
```

---

## 5. Best Practices & Common Pitfalls

### 5.1 Best Practices

#### ✅ DO: Use Concepts (C++20+)
```cpp
template<std::integral T>  // Clear intent
T increment(T value);
```

#### ✅ DO: Constrain Templates
```cpp
// Don't accept everything
template<typename T>
void sort(T& container) requires Container<T>;
```

#### ✅ DO: Use Perfect Forwarding for Universal References
```cpp
template<typename T>
void wrapper(T&& arg) {
    func(std::forward<T>(arg));  // Always forward!
}
```

#### ✅ DO: Prefer alias over typedef
```cpp
template<typename T>
using Vec = std::vector<T>;  // ✅ Can be templated

template<typename T>
typedef std::vector<T> Vec;  // ❌ Can't do this
```

#### ✅ DO: Put Definitions in Headers
```cpp
// Unless using explicit instantiation
```

#### ✅ DO: Use if constexpr for Compile-Time Branches
```cpp
if constexpr (condition) { }  // Discards branch at compile-time
```

### 5.2 Common Pitfalls

#### ❌ DON'T: Forget typename for Dependent Types
```cpp
template<typename T>
void func() {
    T::value_type x;  // ❌ ERROR!
    typename T::value_type x;  // ✅ OK
}
```

#### ❌ DON'T: Use std::move on Universal References
```cpp
template<typename T>
void bad(T&& arg) {
    func(std::move(arg));  // ❌ WRONG! Loses lvalue-ness
}

template<typename T>
void good(T&& arg) {
    func(std::forward<T>(arg));  // ✅ CORRECT
}
```

#### ❌ DON'T: Specialize Function Templates (Usually)
```cpp
// ❌ Prefer overloading
template<typename T> void func(T);
template<> void func<int>(int);  // Tricky interactions

// ✅ Use overloading
template<typename T> void func(T);
void func(int);  // Clearer
```

#### ❌ DON'T: Return References to Local Variables
```cpp
template<typename T>
T& bad() {
    T local;
    return local;  // ❌ Dangling reference!
}

template<typename T>
T good() {
    T local;
    return local;  // ✅ Value return
}
```

#### ❌ DON'T: Assume Template Works for All Types
```cpp
template<typename T>
T divide(T a, T b) {
    return a / b;  // ❌ What if T is std::string?
}

// ✅ Constrain it
template<typename T> requires std::is_arithmetic_v<T>
T divide(T a, T b) {
    return a / b;
}
```

---

## 6. Real-World Examples

### 6.1 Generic Smart Pointer

```cpp
template<typename T>
class UniquePtr {
    T* ptr;
    
public:
    // Constructor
    explicit UniquePtr(T* p = nullptr) : ptr(p) {}
    
    // Destructor
    ~UniquePtr() { delete ptr; }
    
    // Move semantics (no copy!)
    UniquePtr(UniquePtr&& other) noexcept 
        : ptr(other.ptr) {
        other.ptr = nullptr;
    }
    
    UniquePtr& operator=(UniquePtr&& other) noexcept {
        if (this != &other) {
            delete ptr;
            ptr = other.ptr;
            other.ptr = nullptr;
        }
        return *this;
    }
    
    // Deleted copy operations
    UniquePtr(const UniquePtr&) = delete;
    UniquePtr& operator=(const UniquePtr&) = delete;
    
    // Dereference
    T& operator*() const { return *ptr; }
    T* operator->() const { return ptr; }
    T* get() const { return ptr; }
    
    // Release ownership
    T* release() {
        T* old = ptr;
        ptr = nullptr;
        return old;
    }
};

// Usage
UniquePtr<int> ptr(new int(42));
std::cout << *ptr << '\n';  // 42
```

### 6.2 Generic Observer Pattern

```cpp
template<typename EventData>
class Subject {
    using Observer = std::function<void(const EventData&)>;
    std::vector<Observer> observers;
    
public:
    void attach(Observer obs) {
        observers.push_back(std::move(obs));
    }
    
    void notify(const EventData& data) {
        for (auto& obs : observers) {
            obs(data);
        }
    }
};

// Usage
struct TemperatureData {
    double celsius;
};

Subject<TemperatureData> thermometer;

// Attach observers
thermometer.attach([](const auto& data) {
    std::cout << "Temperature: " << data.celsius << "°C\n";
});

thermometer.attach([](const auto& data) {
    std::cout << "Temperature: " << (data.celsius * 9/5 + 32) << "°F\n";
});

// Notify
thermometer.notify({25.0});  // Both observers called
```

### 6.3 Compile-Time Expression Evaluator

```cpp
// Compile-time factorial
template<int N>
struct Factorial {
    static constexpr int value = N * Factorial<N - 1>::value;
};

template<>
struct Factorial<0> {
    static constexpr int value = 1;
};

constexpr int result = Factorial<5>::value;  // 120 at compile-time

// C++17: constexpr function version
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}

constexpr int result2 = factorial(5);  // Also 120 at compile-time
```

---

## Summary: Template Mastery Checklist

### Fundamentals
- [ ] Understand template instantiation
- [ ] Know difference between template and type
- [ ] Can write function and class templates
- [ ] Understand template parameters (type, non-type, template)

### Intermediate
- [ ] Master template specialization
- [ ] Understand template argument deduction
- [ ] Use CTAD effectively (C++17+)
- [ ] Implement perfect forwarding
- [ ] Work with variadic templates

### Advanced
- [ ] Apply SFINAE for type constraints
- [ ] Use type traits for metaprogramming
- [ ] Leverage if constexpr (C++17+)
- [ ] Write and use concepts (C++20+)
- [ ] Apply template design patterns

### Professional
- [ ] Organize template code effectively
- [ ] Minimize code bloat
- [ ] Debug template errors efficiently
- [ ] Write template-friendly APIs
- [ ] Balance compile-time vs runtime

---

## Further Learning Resources

1. **Books:**
   - "C++ Templates: The Complete Guide" by Vandevoorde & Josuttis
   - "Modern C++ Design" by Andrei Alexandrescu
   - "Effective Modern C++" by Scott Meyers

2. **Online:**
   - cppreference.com - Comprehensive reference
   - CppCon talks on YouTube
   - Jason Turner's C++ Weekly

3. **Practice:**
   - Implement STL containers from scratch
   - Write template-heavy libraries
   - Contribute to open-source C++ projects

---

**You now have comprehensive knowledge of C++ templates! Practice and experiment to master them.**
