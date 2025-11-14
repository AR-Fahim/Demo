# C++ String Deep Dive: Complete Reference

## 1. String Types in C++: Foundations and Comparison

### 1.1 The Four Primary String Types

**C-Style Strings (`char*`, `const char*`)**
```cpp
const char* str = "hello";  // 6 bytes: 'h','e','l','l','o','\0'
char buffer[10] = "test";   // mutable, fixed capacity
```
- **NUL-terminated**: Contiguous arrays ending with `'\0'`
- **No metadata**: Size requires O(n) scanning via `strlen()`
- **No ownership**: Raw pointers don't manage memory
- **Manual lifetime**: Caller handles allocation/deallocation
- **Unsafe by default**: Buffer overruns, missing terminators cause UB
- **Use when**: C API interop, string literals, performance-critical code with controlled lifetimes

**`std::string` (Workhorse Type)**
```cpp
std::string s1 = "hello";         // owns copy of data
std::string s2 = s1;              // deep copy
std::string s3 = std::move(s1);   // transfer ownership, s1 becomes empty
```
- **RAII ownership**: Automatically manages dynamic memory
- **Size tracking**: O(1) `.size()` via stored member
- **Dynamic growth**: Automatic reallocation when capacity exceeded
- **Value semantics**: Copies are independent
- **NUL-terminated**: `.c_str()` always returns `'\0'`-terminated data
- **Alias for**: `std::basic_string<char, std::char_traits<char>, std::allocator<char>>`

**`std::string_view` (Non-Owning Reference, C++17)**
```cpp
std::string s = "hello world";
std::string_view sv = s;           // no copy, just reference {ptr, size}
std::string_view sub = sv.substr(0, 5);  // still no copy
```
- **Structure**: Essentially `{const char* data, size_t size}`
- **No ownership**: Does NOT extend source lifetime
- **Not NUL-terminated**: Cannot safely pass to C APIs without verification
- **Lightweight**: 16 bytes (pointer + size)
- **Critical danger**: Dangling references are UB
```cpp
std::string_view danger() {
    std::string local = "temporary";
    return local;  // UB! Returns view to destroyed object
}
```

**Wide and Unicode String Types**
```cpp
std::wstring    // wchar_t (UTF-16 on Windows, UTF-32 on Unix)
std::u8string   // char8_t (C++20), UTF-8
std::u16string  // char16_t, UTF-16
std::u32string  // char32_t, UTF-32
```

### 1.2 Character Encoding: Critical Concepts

**Three-level hierarchy:**
- **Code unit**: Single element of encoding (1 byte UTF-8, 2 bytes UTF-16, 4 bytes UTF-32)
- **Code point**: Unicode scalar value (U+0000 to U+10FFFF)
- **Grapheme cluster**: User-perceived character (may span multiple code points)

```cpp
// UTF-8: "é" can be 1 code point (U+00E9 precomposed) 
// or 2 code points (U+0065 'e' + U+0301 combining acute)
std::string s = "café";  // 5 bytes if precomposed, 6 if decomposed
s.size();                // returns BYTES, not characters
```

**Reality check**: `std::string` performs byte operations, not Unicode-aware operations. For proper Unicode handling, use **ICU** or **Boost.Text**.

### 1.3 Type Selection Decision Matrix

| Scenario | Use | Reason |
|----------|-----|--------|
| Own and store data | `std::string` | RAII, safe, self-managing |
| Read-only function parameter | `std::string_view` | Avoids copies, accepts any string type |
| Return value | `std::string` | Safe ownership transfer |
| Return substring without copy | `std::string_view` | Only if source lifetime guaranteed |
| Store in class/struct | `std::string` | Avoids lifetime hazards |
| Temporary computation | `std::string_view` | Zero-copy efficiency |
| C API interop (read) | `s.c_str()` or `s.data()` | NUL-terminated guarantee |
| C API interop (write) | `s.data()` (C++17) or `&s[0]` | Mutable access |
| Unicode handling | UTF-8 in `std::string` + ICU | Industry standard |

---

## 2. Memory Model and Internal Mechanisms

### 2.1 Memory Layout

**Typical layout** (simplified, varies by implementation):
```cpp
class string {
    char*  ptr;       // pointer to data (heap or internal buffer)
    size_t size;      // current number of characters
    size_t capacity;  // allocated space (excluding NUL terminator)
};
```

### 2.2 Small-String Optimization (SSO)

**Concept**: Store short strings inline within the object to avoid heap allocation.

```cpp
std::string short_str = "tiny";        // stored inline, no allocation
std::string long_str = "this is a very long string";  // heap allocation
```

**Implementation strategy** (common approach):
```cpp
union {
    struct {
        char*  ptr;
        size_t size;
        size_t capacity;
    } heap_data;
    
    struct {
        char   buffer[23];  // size varies by implementation
        char   size_or_flag; // high bit indicates heap mode
    } sso_data;
};
```

**SSO buffer sizes by implementation:**
- **libstdc++ (GCC)**: 15 bytes
- **libc++ (Clang)**: 22 bytes
- **MSVC STL**: 15 bytes

**Performance implications:**
- **Short strings**: No allocations, cache-friendly, fast construction
- **Copy cost**: 24-32 bytes stack copy vs single pointer copy
- **Move cost**: Short strings still copy bytes (can't "steal" inline buffer)
- **ABI locked**: Changing SSO size breaks binary compatibility

```cpp
std::string s = "short";
std::string t = std::move(s);  // Still copies ~24 bytes for SSO strings
```

### 2.3 Capacity Management and Growth Strategy

```cpp
std::string s;
s.reserve(100);      // allocate at least 100, size stays 0
s.capacity();        // >= 100
s.size();            // 0
s += "text";         // size = 4, capacity unchanged
```

**Growth strategy**: Geometric (typically 1.5x or 2x) for amortized O(1) append
```cpp
for (int i = 0; i < 1000; ++i) {
    s += 'x';  // ~10 reallocations with 2x growth
}
```

**Key insight**: Each reallocation copies all existing data. Use `.reserve()` when final size is known.

### 2.4 Copy Semantics (Deep Copy Always)

```cpp
std::string s1 = "data";
std::string s2 = s1;     // Allocates new buffer, copies content
                         // s1 and s2 are independent
```

**Copy-on-Write (COW) is deprecated**: Pre-C++11 implementations used COW, but it's incompatible with C++11 thread-safety requirements. All modern implementations use deep copy.

### 2.5 Move Semantics (Ownership Transfer)

```cpp
std::string s1 = "data";
std::string s2 = std::move(s1);  
// s2 takes s1's buffer
// s1 becomes empty (valid but unspecified state)
```

**SSO caveat**: If string fits in SSO buffer, move still copies bytes (cannot "steal" inline storage).

**Copy elision (C++17 guaranteed)**:
```cpp
std::string make() { return std::string("result"); }
std::string s = make();  // No copy, no move—direct construction (RVO)
```

### 2.6 Iterator/Pointer/Reference Invalidation Rules

**Invalidation triggers:**
- **Capacity-changing operations**: `reserve()`, `shrink_to_fit()`, operations causing reallocation
  - **Effect**: All iterators, pointers, and references invalidated
- **Non-capacity modifications**: `insert()`, `erase()`, `replace()` at positions before `.end()`
  - **Effect**: Iterators/pointers at or after modification point invalidated
  - **References**: May remain valid (implementation-dependent)

```cpp
std::string s = "hello";
const char* ptr = s.c_str();
s += " world";               // May reallocate, ptr potentially dangling
```

**Safe pattern**: Re-acquire pointers after modifications
```cpp
s.reserve(100);              // Ensure capacity
const char* ptr = s.c_str(); // Safe until next capacity change
```

### 2.7 `c_str()` vs `data()` Evolution

**Before C++11:**
- `c_str()`: Returns NUL-terminated `const char*`
- `data()`: May not be NUL-terminated, read-only

**C++11 onwards:**
- Both return NUL-terminated `const char*`
- Accessing `s[s.size()]` is legal and returns `'\0'`

**C++17 addition:**
- `data()` also has non-const overload returning `char*`

```cpp
std::string s = "test";
assert(s.data()[s.size()] == '\0');  // Guaranteed since C++11
s.data()[0] = 'T';                   // OK in C++17, modifies string
```

### 2.8 Character Traits and Allocators

**Full template signature:**
```cpp
template<
    class CharT,
    class Traits = std::char_traits<CharT>,
    class Allocator = std::allocator<CharT>
>
class basic_string;
```

**Traits**: Define character operations (comparison, copying, finding, EOF)
**Allocators**: Control memory allocation strategy

**Custom allocator example:**
```cpp
using PmrString = std::basic_string<char, 
                                     std::char_traits<char>,
                                     std::pmr::polymorphic_allocator<char>>;

std::pmr::monotonic_buffer_resource pool;
PmrString s("data", &pool);  // Allocated from pool
```

**Important**: Strings with different allocators are incompatible—assignment requires reallocation.

### 2.9 Thread Safety Model

**Const operations**: Thread-safe (multiple readers)
```cpp
const std::string s = "shared";
// Safe: Multiple threads can call s.size(), s[0], s.data() simultaneously
```

**Non-const operations**: Not thread-safe (data races)
```cpp
std::string s2 = "mutable";
// NOT safe: One thread modifies while another reads
```

**Safe patterns:**
1. **Read-only sharing**: Pass by `const&` or `string_view`
2. **Exclusive ownership**: One thread owns, others don't touch
3. **Synchronized access**: Use `std::mutex` for shared mutable strings

---

## 3. Complete API Reference

### 3.1 Construction

**All constructor overloads (19 in C++20):**
```cpp
// Default and allocator variants
std::string s;                              // Empty
std::string s(alloc);                       // Empty with allocator
std::string s(str, alloc);                  // Copy with allocator
std::string s(std::move(str), alloc);       // Move with allocator

// From existing string (substring)
std::string s(str, pos);                    // From position to end
std::string s(str, pos, len);               // Substring

// From C-string
std::string s("text");                      // NUL-terminated
std::string s(cstr, len);                   // First len characters

// Fill constructor
std::string s(10, 'x');                     // 10 copies of 'x'

// From range
std::string s(begin, end);                  // Iterator range

// Copy and move
std::string s(other);                       // Deep copy
std::string s(std::move(other));            // Transfer ownership

// From initializer list
std::string s{'h', 'i'};                    // {'h', 'i'}

// From string_view (C++17)
std::string s(sv);                          // Full view
std::string s(sv, pos, len);                // Substring of view

// From string_view-like types (C++17)
std::string s(t);                           // Any type convertible to string_view
```

**Critical**: Constructing with `nullptr` is UB
```cpp
const char* p = nullptr;
std::string s(p);  // UB!
```

**String literals with embedded nulls:**
```cpp
std::string s("ab\0cd", 5);      // Size = 5
std::string s = "ab\0cd"s;       // C++14 literal, size = 5
```

### 3.2 Assignment

**`operator=` variants:**
```cpp
s = other;              // Copy assignment
s = std::move(other);   // Move assignment
s = "text";             // C-string
s = 'c';                // Single character
s = {'a', 'b'};         // Initializer list
s = sv;                 // string_view (C++17)
```

**`assign()` overloads (9 total):**
```cpp
s.assign(str);                         // Assign string
s.assign(str, subpos, sublen);         // Assign substring
s.assign(std::move(str));              // Move assign
s.assign(cstr);                        // Assign C-string
s.assign(cstr, n);                     // Assign first n chars
s.assign(n, c);                        // Assign n copies of char
s.assign(first, last);                 // Assign iterator range
s.assign({init, list});                // Assign initializer list
s.assign(sv);                          // Assign string_view (C++17)
```

### 3.3 Element Access

| Method | Description | Behavior |
|--------|-------------|----------|
| `operator[i]` | Unchecked access | UB if `i > size()` (except `i == size()` returns `'\0'`) |
| `at(i)` | Checked access | Throws `std::out_of_range` if invalid |
| `front()` | First character | UB on empty string |
| `back()` | Last character | UB on empty string |
| `data()` | Pointer to buffer | Returns `char*` (C++17) or `const char*` |
| `c_str()` | NUL-terminated C-string | Always returns `const char*` |

```cpp
std::string s = "hello";
char c1 = s[0];           // 'h', no bounds check
char c2 = s.at(10);       // Throws std::out_of_range
char null = s[s.size()];  // '\0', guaranteed since C++11
```

### 3.4 Capacity Management

| Method | Description | Notes |
|--------|-------------|-------|
| `size()` / `length()` | Number of characters | O(1), stored member |
| `capacity()` | Allocated space | O(1), may exceed size |
| `empty()` | True if `size() == 0` | O(1) |
| `max_size()` | Theoretical maximum | Implementation limit |
| `reserve(n)` | Ensure `capacity >= n` | May allocate, never reduces |
| `shrink_to_fit()` | Request capacity reduction | Non-binding request |
| `resize(n)` | Change size | Pads with `'\0'` if growing |
| `resize(n, c)` | Change size, pad with `c` | Truncates or extends |
| `clear()` | Set `size = 0` | Does NOT free memory |
| `get_allocator()` | Returns allocator copy | For allocator-aware code |

```cpp
std::string s;
s.reserve(100);         // Allocate 100 bytes
s.resize(50, 'x');      // Size = 50, filled with 'x'
s.clear();              // Size = 0, capacity still 100
s.shrink_to_fit();      // Request to reduce capacity (non-binding)
```

### 3.5 Modifiers

**Appending:**
```cpp
s += "text";            // operator+=, accepts string/C-string/char/string_view
s.append(str);          // Multiple overloads (8 total)
s.push_back('c');       // Append single character
```

**`append()` overloads:**
```cpp
s.append(str);                         // Append string
s.append(str, subpos, sublen);         // Append substring
s.append(cstr);                        // Append C-string
s.append(cstr, n);                     // Append first n chars
s.append(n, c);                        // Append n copies of char
s.append(first, last);                 // Append iterator range
s.append({init, list});                // Append initializer list
s.append(sv);                          // Append string_view (C++17)
```

**Inserting:**
```cpp
s.insert(pos, str);     // Insert at position (9 overloads total)
```

**`insert()` overloads:**
```cpp
s.insert(pos, str);                    // Insert string at position
s.insert(pos, str, subpos, sublen);    // Insert substring
s.insert(pos, cstr);                   // Insert C-string
s.insert(pos, cstr, n);                // Insert first n chars
s.insert(pos, n, c);                   // Insert n copies of char
iterator insert(const_iterator p, c);  // Insert char before iterator
iterator insert(const_iterator p, n, c); // Insert n chars before iterator
iterator insert(const_iterator p, first, last); // Insert range
iterator insert(const_iterator p, {init, list}); // Insert initializer list
```

**Erasing:**
```cpp
s.erase(pos, len);      // Erase substring (3 overloads total)
s.pop_back();           // Remove last character
```

**`erase()` overloads:**
```cpp
s.erase(pos, len);                           // Erase substring
iterator erase(const_iterator position);     // Erase single char
iterator erase(const_iterator first, last);  // Erase range
```

**Replacing:**
```cpp
s.replace(pos, len, str);  // Replace substring (14 overloads)
```

**`replace()` overloads (position-based):**
```cpp
s.replace(pos, len, str);
s.replace(pos, len, str, subpos, sublen);
s.replace(pos, len, cstr);
s.replace(pos, len, cstr, n);
s.replace(pos, len, n, c);
s.replace(pos, len, sv);                     // C++17
s.replace(pos, len, t, subpos, sublen);      // C++17, t → string_view
```

**`replace()` overloads (iterator-based):**
```cpp
s.replace(i1, i2, str);
s.replace(i1, i2, cstr);
s.replace(i1, i2, cstr, n);
s.replace(i1, i2, n, c);
s.replace(i1, i2, first, last);
s.replace(i1, i2, {init, list});
s.replace(i1, i2, sv);                       // C++17
```

**Other modifiers:**
```cpp
s.swap(other);          // O(1) exchange (pointer swap)
s.copy(buf, n, pos);    // Copy substring to C-array (does NOT NUL-terminate!)
```

**Critical `copy()` note:**
```cpp
char buffer[6];
s.copy(buffer, 5, 0);   // Copies 5 chars
buffer[5] = '\0';       // Must manually NUL-terminate!
```

### 3.6 String Operations and Searching

**Substring extraction:**
```cpp
std::string sub = s.substr(pos, len);  // Creates new string (copy)
```

**Comparison:**
```cpp
int result = s.compare(str);  // Returns <0, 0, or >0 (6 overloads)
```

**`compare()` overloads:**
```cpp
s.compare(str);
s.compare(pos, len, str);
s.compare(pos, len, str, subpos, sublen);
s.compare(cstr);
s.compare(pos, len, cstr);
s.compare(pos, len, cstr, n);
```

**Searching (returns `std::string::npos` if not found):**

| Method | Description | Example |
|--------|-------------|---------|
| `find(str, pos=0)` | Find first occurrence | `s.find("sub")` |
| `rfind(str, pos=npos)` | Find last occurrence | `s.rfind("sub")` |
| `find_first_of(chars, pos=0)` | First of any char | `s.find_first_of("aeiou")` |
| `find_last_of(chars, pos=npos)` | Last of any char | `s.find_last_of("aeiou")` |
| `find_first_not_of(chars, pos=0)` | First not matching | `s.find_first_not_of(" \t")` |
| `find_last_not_of(chars, pos=npos)` | Last not matching | `s.find_last_not_of(" \t")` |

**C++20/23 additions:**
```cpp
bool hasPrefix = s.starts_with("prefix");  // C++20
bool hasSuffix = s.ends_with(".txt");      // C++20
bool hasSubstr = s.contains("middle");     // C++23
```

**Search pattern:**
```cpp
size_t pos = s.find("needle");
if (pos != std::string::npos) {
    // Found at position pos
}
```

### 3.7 Advanced Methods

**`resize_and_overwrite()` (C++23)** - High-performance resize with direct buffer access:
```cpp
std::string s;
s.resize_and_overwrite(100, [](char* buf, size_t n) {
    int written = snprintf(buf, n, "formatted %d", 42);
    return written;  // Return actual size used
});
```
**Use case**: Efficiently work with C APIs that write to buffers, avoiding unnecessary zero-initialization.

### 3.8 Iterators

| Method | Description |
|--------|-------------|
| `begin()` / `end()` | Forward iterator range |
| `rbegin()` / `rend()` | Reverse iterator range |
| `cbegin()` / `cend()` | Const forward iterators |
| `crbegin()` / `crend()` | Const reverse iterators |

**Range-based for:**
```cpp
for (char c : s) { /* process c */ }
for (char& c : s) { c = std::toupper(c); }  // Modify in place
```

### 3.9 Non-Member Functions

**Concatenation:**
```cpp
std::string result = s1 + s2;       // Creates new string
std::string result = s1 + "text";   // Many operator+ overloads
std::string result = "text" + s1;   // All combinations supported
```

**Comparison operators:**
```cpp
bool eq = (s1 == s2);               // ==, !=, <, <=, >, >=
auto cmp = s1 <=> s2;               // Three-way comparison (C++20)
```

**Stream I/O:**
```cpp
std::cout << s;                     // Output
std::cin >> s;                      // Input (reads until whitespace)
std::getline(std::cin, s);          // Read line (until '\n')
std::getline(std::cin, s, delim);   // Read until delimiter
```

**Conversions:**

| Function | Description | Example |
|----------|-------------|---------|
| `std::to_string(val)` | Number → string | `std::to_string(42)` |
| `std::stoi(str)` | String → int | `std::stoi("123")` |
| `std::stol(str)` | String → long | `std::stol("1234567")` |
| `std::stoll(str)` | String → long long | `std::stoll("999999999999")` |
| `std::stoul(str)` | String → unsigned long | `std::stoul("123")` |
| `std::stoull(str)` | String → unsigned long long | `std::stoull("999999999999")` |
| `std::stof(str)` | String → float | `std::stof("3.14")` |
| `std::stod(str)` | String → double | `std::stod("3.14159")` |
| `std::stold(str)` | String → long double | `std::stold("3.14159")` |

**Conversion error handling:**
```cpp
try {
    int n = std::stoi("not a number");
} catch (const std::invalid_argument& e) {
    // Conversion failed
} catch (const std::out_of_range& e) {
    // Number too large for type
}
```

**Swap:**
```cpp
std::swap(s1, s2);  // Equivalent to s1.swap(s2)
```

**String literals (C++14):**
```cpp
using namespace std::string_literals;
auto s = "hello"s;              // Type: std::string
auto ws = L"hello"s;            // Type: std::wstring
auto u8s = u8"hello"s;          // Type: std::string (C++17) / std::u8string (C++20)
auto u16s = u"hello"s;          // Type: std::u16string
auto u32s = U"hello"s;          // Type: std::u32string
```

---

## 4. `std::string_view` Complete API

### 4.1 Core Characteristics

**Internal structure** (conceptual):
```cpp
class string_view {
    const char* data_;
    size_t size_;
};
```

**Size**: 16 bytes (2 pointers) on 64-bit systems
**Ownership**: None—just a view into existing data
**NUL-termination**: NOT guaranteed

### 4.2 Construction

```cpp
std::string s = "hello";
std::string_view sv1 = s;              // From std::string
std::string_view sv2 = "literal";      // From string literal
std::string_view sv3(ptr, len);        // From pointer + length
std::string_view sv4(ptr);             // From NUL-terminated C-string
std::string_view sv5;                  // Empty view
```

### 4.3 Lifetime Safety Rules

**DANGER**: View does NOT extend source lifetime

**Unsafe patterns:**
```cpp
// 1. Returning view to local
std::string_view bad1() {
    std::string temp = "local";
    return temp;  // UB! temp destroyed
}

// 2. Returning view to temporary
std::string_view bad2() {
    return std::string("temp");  // UB! Temporary destroyed immediately
}

// 3. Storing view to temporary
void bad3() {
    std::string_view sv = std::string("temp");  // UB! Dangling after statement
}

// 4. View outlives source
std::string_view bad4() {
    std::string s = "data";
    std::string_view sv = s;
    s.clear();
    return sv;  // UB! Views cleared string
}
```

**Safe patterns:**
```cpp
// 1. Parameter: source outlives call
void process(std::string_view sv);
std::string s = "data";
process(s);                    // Safe: s outlives call
process("literal");            // Safe: literal has static storage

// 2. Local usage only
void compute() {
    std::string s = "data";
    std::string_view sv = s;
    // Use sv within function
}  // Both destroyed together—safe

// 3. Copy to owned string when needed
std::string store(std::string_view sv) {
    return std::string(sv);    // Safe: creates owned copy
}
```

### 4.4 Operations

**Element access:**
```cpp
sv[i]           // Unchecked access
sv.at(i)        // Throws on out of bounds
sv.front()      // First character
sv.back()       // Last character
sv.data()       // Returns const char*
```

**Capacity:**
```cpp
sv.size()       // Number of characters
sv.length()     // Same as size()
sv.empty()      // True if size() == 0
sv.max_size()   // Maximum possible size
```

**Modifiers (modify view, not underlying data):**
```cpp
sv.remove_prefix(n);   // Advance start by n
sv.remove_suffix(n);   // Reduce end by n
sv.swap(other);        // Exchange views
```

```cpp
std::string_view sv = "hello world";
sv.remove_prefix(6);   // Now views "world"
sv.remove_suffix(2);   // Now views "wor"
```

**Operations (same as std::string):**
```cpp
sv.substr(pos, len)     // Returns new string_view (no allocation!)
sv.compare(...)         // Lexicographic comparison
sv.find(...)            // Search operations
sv.rfind(...)
sv.find_first_of(...)
sv.find_last_of(...)
sv.find_first_not_of(...)
sv.find_last_not_of(...)
sv.starts_with(...)     // C++20
sv.ends_with(...)       // C++20
sv.contains(...)        // C++23
```

**Key difference**: `substr()` returns another view, not a copy
```cpp
std::string s = "hello world";
std::string_view sv = s;
std::string_view sub = sv.substr(0, 5);  // Views "hello", no allocation
```

### 4.5 Conversion to std::string

**Explicit copy required:**
```cpp
std::string_view sv = "view";
std::string s1(sv);                // Constructor
std::string s2 = std::string(sv);  // Explicit construction
std::string s3;
s3 = sv;                           // Assignment (C++17)
```

### 4.6 Comparison with C-Strings

**Problem**: `string_view` is NOT guaranteed NUL-terminated
```cpp
std::string s = "hello world";
std::string_view sv = s;
sv.remove_suffix(6);         // Now views "hello"
// sv.data() points into s, but sv.data()[sv.size()] might be ' ', not '\0'

printf("%s", sv.data());     // DANGER! May print "hello world" or worse
```

**Safe C-API usage:**
```cpp
// Option 1: Copy to string first
void safe_printf(std::string_view sv) {
    printf("%s", std::string(sv).c_str());
}

// Option 2: Use size explicitly
void safe_printf(std::string_view sv) {
    printf("%.*s", static_cast<int>(sv.size()), sv.data());
}
```

### 4.7 When to Use string_view

**✓ Good uses:**
- Function parameters for read-only strings
- Temporary substring operations without allocation
- Parsing and tokenization (within function scope)
- Avoiding unnecessary copies from various string types

**✗ Bad uses:**
- Storing in class members (lifetime hazard)
- Returning from functions (unless source lifetime is guaranteed)
- Storing in containers (`std::vector<string_view>` is dangerous)
- Passing to C APIs expecting NUL-termination (without verification)

**Example of good usage:**
```cpp
// Accepts string, string_view, C-string with no copies
void log(std::string_view message) {
    std::cout << "[LOG] " << message << '\n';
}

log("literal");           // No copy
log(my_string);           // No copy
log(my_string.substr(0, 10));  // No copy
```

---

## 5. Common String Operations and Efficient Idioms

### 5.1 Concatenation Strategies

**Approach 1: `operator+` (simple but creates temporaries)**
```cpp
std::string s = s1 + s2 + s3;  // Multiple temporary strings
```

**Approach 2: `operator+=` (better)**
```cpp
std::string s = s1;
s += s2;
s += s3;
```

**Approach 3: Pre-reserve + append (best for known sizes)**
```cpp
std::string s;
s.reserve(s1.size() + s2.size() + s3.size());
s += s1;
s += s2;
s += s3;
```

**Approach 4: `ostringstream` (flexible, slightly slower)**
```cpp
std::ostringstream oss;
oss << s1 << s2 << s3;
std::string s = oss.str();
```

**Performance ranking**: Pre-reserve > operator+= > ostringstream > operator+

### 5.2 Formatting Options

**C++20 `std::format` (Modern, type-safe, fast):**
```cpp
#include <format>
std::string s = std::format("Value: {}, Hex: {:x}", 42, 255);
// "Value: 42, Hex: ff"

std::string s = std::format("{0} {1} {0}", "A", "B");  // "A B A"
std::string s = std::format("{:.2f}", 3.14159);        // "3.14"
```

**C++17 `std::to_chars` (Fastest, no allocation, no locale):**
```cpp
#include <charconv>
char buffer[20];
auto [ptr, ec] = std::to_chars(buffer, buffer + sizeof(buffer), 42);
if (ec == std::errc{}) {
    std::string s(buffer, ptr);  // Success
}
```

**Traditional `sprintf` family (unsafe):**
```cpp
char buf[100];
sprintf(buf, "Value: %d", 42);   // Buffer overflow risk!
snprintf(buf, sizeof(buf), "Value: %d", 42);  // Safer
```

**`ostringstream` (pre-C++20 flexible option):**
```cpp
#include <sstream>
std::ostringstream oss;
oss << "Value: " << 42 << ", Hex: " << std::hex << 255;
std::string s = oss.str();
```

**Third-party `{fmt}` library (basis for std::format):**
```cpp
#include <fmt/core.h>
std::string s = fmt::format("The answer is {}", 42);
```

**Performance comparison**: `to_chars` > `std::format` > `ostringstream` > `sprintf`

### 5.3 String Splitting

**Manual split with string_view:**
```cpp
std::vector<std::string> split(std::string_view sv, char delim) {
    std::vector<std::string> result;
    size_t start = 0, end;
    while ((end = sv.find(delim, start)) != std::string_view::npos) {
        result.emplace_back(sv.substr(start, end - start));
        start = end + 1;
    }
    result.emplace_back(sv.substr(start));
    return result;
}
```

**Using `getline`:**
```cpp
std::istringstream iss(str);
std::string token;
std::vector<std::string> tokens;
while (std::getline(iss, token, ',')) {
    tokens.push_back(token);
}
```

**Boost.StringAlgo:**
```cpp
#include <boost/algorithm/string.hpp>
std::vector<std::string> tokens;
boost::split(tokens, str, boost::is_any_of(","));
```

### 5.4 Trimming Whitespace

**Manual trim with string_view:**
```cpp
std::string_view trim(std::string_view sv) {
    size_t start = sv.find_first_not_of(" \t\n\r");
    if (start == std::string_view::npos) return {};
    size_t end = sv.find_last_not_of(" \t\n\r");
    return sv.substr(start, end - start + 1);
}
```

**In-place trim:**
```cpp
void trim_inplace(std::string& s) {
    // Trim left
    s.erase(s.begin(), std::find_if(s.begin(), s.end(), [](unsigned char c) {
        return !std::isspace(c);
    }));
    // Trim right
    s.erase(std::find_if(s.rbegin(), s.rend(), [](unsigned char c) {
        return !std::isspace(c);
    }).base(), s.end());
}
```

**Boost.StringAlgo:**
```cpp
#include <boost/algorithm/string.hpp>
std::string s = "  text  ";
boost::trim(s);  // In-place: s = "text"
std::string trimmed = boost::trim_copy(s);  // Copy version
```

### 5.5 Case Conversion

**Simple ASCII (locale-independent):**
```cpp
std::string to_upper(std::string s) {
    std::transform(s.begin(), s.end(), s.begin(),
        [](unsigned char c) { return std::toupper(c); });
    return s;
}

std::string to_lower(std::string s) {
    std::transform(s.begin(), s.end(), s.begin(),
        [](unsigned char c) { return std::tolower(c); });
    return s;
}
```

**Locale-aware (use with caution):**
```cpp
std::locale loc("en_US.UTF-8");
std::string s = "text";
std::use_facet<std::ctype<char>>(loc).toupper(s.data(), s.data() + s.size());
```

**Boost.StringAlgo:**
```cpp
#include <boost/algorithm/string.hpp>
std::string s = "Hello";
boost::to_upper(s);        // In-place: s = "HELLO"
std::string upper = boost::to_upper_copy(s);  // Copy version
```

**ICU (Unicode-aware, correct for international text):**
```cpp
#include <unicode/unistr.h>
icu::UnicodeString u = icu::UnicodeString::fromUTF8("straße");
u.toUpper();  // Correctly handles German ß → SS
```

### 5.6 Searching and Replacing

**Find and replace first:**
```cpp
void replace_first(std::string& str, std::string_view from, std::string_view to) {
    size_t pos = str.find(from);
    if (pos != std::string::npos) {
        str.replace(pos, from.size(), to);
    }
}
```

**Find and replace all:**
```cpp
void replace_all(std::string& str, std::string_view from, std::string_view to) {
    size_t pos = 0;
    while ((pos = str.find(from, pos)) != std::string::npos) {
        str.replace(pos, from.size(), to);
        pos += to.size();  // Move past the replacement
    }
}
```

**Boost.StringAlgo:**
```cpp
#include <boost/algorithm/string.hpp>
boost::replace_all(str, "old", "new");  // In-place
std::string result = boost::replace_all_copy(str, "old", "new");  // Copy
```

### 5.7 Case-Insensitive Operations

**Case-insensitive comparison:**
```cpp
bool iequals(std::string_view a, std::string_view b) {
    return std::equal(a.begin(), a.end(), b.begin(), b.end(),
        [](char c1, char c2) {
            return std::tolower(static_cast<unsigned char>(c1)) ==
                   std::tolower(static_cast<unsigned char>(c2));
        });
}
```

**Case-insensitive search:**
```cpp
size_t ifind(std::string_view haystack, std::string_view needle) {
    auto it = std::search(haystack.begin(), haystack.end(),
        needle.begin(), needle.end(),
        [](char c1, char c2) {
            return std::tolower(static_cast<unsigned char>(c1)) ==
                   std::tolower(static_cast<unsigned char>(c2));
        });
    return it == haystack.end() ? std::string::npos : std::distance(haystack.begin(), it);
}
```

**Boost.StringAlgo:**
```cpp
#include <boost/algorithm/string.hpp>
bool same = boost::iequals(s1, s2);
bool found = boost::ifind_first(text, pattern);
```

### 5.8 Numeric Conversions

**String to number (exceptions on error):**
```cpp
int n = std::stoi("123");
long l = std::stol("123456");
double d = std::stod("3.14");
```

**String to number with error handling:**
```cpp
std::optional<int> safe_stoi(std::string_view sv) {
    try {
        return std::stoi(std::string(sv));
    } catch (...) {
        return std::nullopt;
    }
}
```

**`from_chars` (C++17, fastest, no exceptions, no locale):**
```cpp
#include <charconv>
int value;
auto [ptr, ec] = std::from_chars(str.data(), str.data() + str.size(), value);
if (ec == std::errc{}) {
    // Success: value contains result
} else if (ec == std::errc::invalid_argument) {
    // Invalid format
} else if (ec == std::errc::result_out_of_range) {
    // Number too large
}
```

**Number to string:**
```cpp
std::string s = std::to_string(42);     // Simple
std::string s = std::format("{}", 42);  // C++20, flexible
```

**`to_chars` (C++17, fastest):**
```cpp
char buffer[20];
auto [ptr, ec] = std::to_chars(buffer, buffer + sizeof(buffer), 42);
std::string s(buffer, ptr);
```

### 5.9 Working with Binary Data

**std::string can hold arbitrary bytes:**
```cpp
std::string binary("\x00\x01\x02\xFF", 4);  // Size = 4
assert(binary.size() == 4);
assert(binary[0] == '\0');
```

**Reading binary data:**
```cpp
std::ifstream file("data.bin", std::ios::binary);
std::string data((std::istreambuf_iterator<char>(file)),
                  std::istreambuf_iterator<char>());
```

**Writing binary data:**
```cpp
std::ofstream file("data.bin", std::ios::binary);
file.write(data.data(), data.size());
```

**Warning**: C-string functions will fail on embedded nulls
```cpp
std::string binary("ab\0cd", 5);
strlen(binary.c_str());  // Returns 2, not 5!
binary.size();           // Correct: returns 5
```

### 5.10 C API Interoperability

**Passing to C functions (read-only):**
```cpp
void c_function(const char* str);

std::string s = "data";
c_function(s.c_str());  // Safe: NUL-terminated
```

**Receiving from C functions:**
```cpp
char* c_str = get_c_string();  // C allocates
std::string s = c_str;         // Copy into std::string
free(c_str);                   // Free C memory
```

**Providing mutable buffer to C API:**
```cpp
std::string buffer(100, '\0');  // Allocate 100 bytes
some_c_function(buffer.data(), buffer.size());
buffer.resize(strlen(buffer.c_str()));  // Adjust to actual size
```

**Modern approach (C++17):**
```cpp
std::string buffer(100, '\0');
int written = snprintf(buffer.data(), buffer.size(), "format %d", 42);
buffer.resize(written);
```

**Using `resize_and_overwrite` (C++23):**
```cpp
std::string buffer;
buffer.resize_and_overwrite(100, [](char* buf, size_t n) {
    return snprintf(buf, n, "format %d", 42);
});
```

---

## 6. Performance Optimization and Best Practices

### 6.1 Avoiding Unnecessary Copies

**Use move semantics:**
```cpp
std::string create() {
    std::string result = "data";
    return result;  // Automatically moved or elided
}

std::string s = create();  // No copy
```

**Pass by `const&` or `string_view`:**
```cpp
// Bad: unnecessary copy
void process(std::string s) { /* ... */ }

// Good: no copy for read-only
void process(const std::string& s) { /* ... */ }

// Better: accepts any string type
void process(std::string_view sv) { /* ... */ }
```

**Use `reserve()` when size is known:**
```cpp
// Bad: multiple reallocations
std::string s;
for (int i = 0; i < 1000; ++i) {
    s += std::to_string(i);
}

// Good: single allocation
std::string s;
s.reserve(5000);  // Estimate total size
for (int i = 0; i < 1000; ++i) {
    s += std::to_string(i);
}
```

**Prefer `append` over `operator+`:**
```cpp
// Less efficient: creates temporaries
std::string s = a + b + c;

// More efficient: in-place append
std::string s;
s.reserve(a.size() + b.size() + c.size());
s.append(a).append(b).append(c);
```

**Use `swap` instead of assignment for large strings:**
```cpp
std::string a = "large data...";
std::string b = "other large data...";
a.swap(b);  // O(1) pointer swap
```

### 6.2 Small-String Optimization Impact

**SSO benefits:**
- No heap allocation for short strings (≤15-23 chars)
- Cache-friendly data access
- Fast construction and destruction

**SSO downsides:**
- Move operations still copy bytes
- Larger object size (24-32 bytes)

**When SSO helps:**
```cpp
std::vector<std::string> vec;
for (int i = 0; i < 10000; ++i) {
    vec.push_back("short");  // No allocations, all SSO
}
```

**When SSO doesn't help:**
```cpp
std::string s = "this is definitely too long for SSO";
std::string t = std::move(s);  // Still heap-allocated, pointer stolen
```

**For guaranteed cheap moves:**
```cpp
std::vector<std::unique_ptr<std::string>> vec;  // Move is always O(1)
```

### 6.3 Allocator Strategies

**Default allocator:** Uses `new`/`delete`, general-purpose

**Custom allocators for specific scenarios:**

**Pool allocator (reduce fragmentation):**
```cpp
#include <memory_resource>
std::pmr::monotonic_buffer_resource pool(10000);  // 10KB pool
std::pmr::string s("data", &pool);
```

**Use case:** Many small strings with predictable lifetime
```cpp
void process_records() {
    std::pmr::monotonic_buffer_resource pool;
    std::vector<std::pmr::string> records(&pool);
    
    // Parse many records...
    
    // All deallocated at once when pool destroyed
}
```

### 6.4 Threading Patterns

**Read-only sharing (safe):**
```cpp
const std::string shared = "immutable data";

void worker() {
    size_t len = shared.size();  // Thread-safe
    char first = shared[0];      // Thread-safe
}
```

**Exclusive ownership (safe):**
```cpp
void worker(std::string data) {  // Copy/move by value
    data += " modified";         // Safe: private copy
}
```

**Shared mutable (requires synchronization):**
```cpp
std::mutex mtx;
std::string shared;

void append(std::string_view text) {
    std::lock_guard lock(mtx);
    shared += text;
}
```

**Thread-local storage:**
```cpp
thread_local std::string buffer;  // Each thread has own copy

void worker() {
    buffer.clear();
    buffer += "thread-specific data";
}
```

### 6.5 Common Anti-Patterns

**❌ Returning local by reference:**
```cpp
const std::string& bad() {
    std::string local = "temp";
    return local;  // Dangling reference!
}
```

**❌ Unnecessary string copies:**
```cpp
// Bad
std::string get_name() const { return name_; }

// Good for most cases
const std::string& get_name() const { return name_; }

// Best for flexibility
std::string_view get_name() const { return name_; }
```

**❌ Inefficient concatenation:**
```cpp
// Quadratic complexity
std::string result;
for (const auto& s : strings) {
    result = result + s;  // Creates temporary each iteration
}

// Linear complexity
std::string result;
for (const auto& s : strings) {
    result += s;  // Amortized O(1) append
}
```

**❌ Ignoring return values:**
```cpp
s.find("pattern");  // Result discarded!

// Correct
if (s.find("pattern") != std::string::npos) {
    // Found
}
```

**❌ Using wrong conversion:**
```cpp
// Inefficient: exception overhead
int n;
try { n = std::stoi(str); } catch (...) {}

// Efficient: no exceptions
auto [ptr, ec] = std::from_chars(str.data(), str.data() + str.size(), n);
```

---

## 7. Security, Correctness, and Safety

### 7.1 Buffer Safety

**std::string is safe by design:**
- Automatic bounds checking with `.at()`
- No buffer overruns with member functions
- Exception safety guarantees

**C-string dangers:**
```cpp
char buf[10];
strcpy(buf, "this is too long");  // Buffer overrun! UB!
gets(buf);                        // Extremely dangerous, deprecated
```

**Safe C-string practices:**
```cpp
strncpy(buf, src, sizeof(buf) - 1);
buf[sizeof(buf) - 1] = '\0';  // Ensure termination

// Better: use std::string
std::string s = src;
```

### 7.2 Embedded Null Handling

**std::string handles embedded nulls correctly:**
```cpp
std::string s("data\0more", 9);
assert(s.size() == 9);           // Correct
assert(strlen(s.c_str()) == 4);  // C functions stop at first null
```

**Safe practices:**
- Use `.size()` for length, never `strlen()` on `.c_str()`
- Use binary-safe methods: `.compare()`, `.find()`, etc.
- Be aware when interfacing with C APIs

### 7.3 Input Validation

**Always validate untrusted input:**
```cpp
std::string validate_input(std::string_view input) {
    constexpr size_t MAX_SIZE = 1000;
    if (input.size() > MAX_SIZE) {
        throw std::invalid_argument("Input too large");
    }
    
    // Check for valid characters
    for (char c : input) {
        if (!std::isprint(static_cast<unsigned char>(c))) {
            throw std::invalid_argument("Invalid character");
        }
    }
    
    return std::string(input);
}
```

**Length limits:**
```cpp
bool is_valid_username(std::string_view name) {
    return name.size() >= 3 && 
           name.size() <= 20 &&
           std::all_of(name.begin(), name.end(), [](char c) {
               return std::isalnum(static_cast<unsigned char>(c)) || c == '_';
           });
}
```

### 7.4 Injection Prevention

**SQL injection (use parameterized queries):**
```cpp
// NEVER do this
std::string query = "SELECT * FROM users WHERE name = '" + user_input + "'";

// Use library-specific parameterized queries instead
// (Example syntax varies by database library)
```

**Path traversal:**
```cpp
bool is_safe_filename(std::string_view name) {
    return name.find("..") == std::string_view::npos &&
           name.find('/') == std::string_view::npos &&
           name.find('\\') == std::string_view::npos &&
           !name.empty() &&
           name[0] != '.';
}
```

**Command injection:**
```cpp
// NEVER pass unsanitized input to system()
// system(("command " + user_input).c_str());  // DANGER!

// Use proper APIs that don't invoke shell
// Or carefully validate and escape
```

### 7.5 Encoding Validation

**UTF-8 validation:**
```cpp
bool is_valid_utf8(std::string_view sv) {
    size_t i = 0;
    while (i < sv.size()) {
        unsigned char c = sv[i];
        int len = 0;
        
        if (c < 0x80) len = 1;
        else if ((c & 0xE0) == 0xC0) len = 2;
        else if ((c & 0xF0) == 0xE0) len = 3;
        else if ((c & 0xF8) == 0xF0) len = 4;
        else return false;  // Invalid start byte
        
        if (i + len > sv.size()) return false;  // Incomplete sequence
        
        // Validate continuation bytes
        for (int j = 1; j < len; ++j) {
            if ((sv[i + j] & 0xC0) != 0x80) return false;
        }
        
        i += len;
    }
    return true;
}
```

**Use ICU for production:**
```cpp
#include <unicode/unistr.h>
bool is_valid_utf8(std::string_view sv) {
    UErrorCode status = U_ZERO_ERROR;
    icu::UnicodeString::fromUTF8(icu::StringPiece(sv.data(), sv.size()))
        .toUTF8String(std::string());  // Will set error if invalid
    return U_SUCCESS(status);
}
```

### 7.6 Locale Issues

**Locale-dependent behavior can cause bugs:**
```cpp
std::string s = "TITLE";
std::transform(s.begin(), s.end(), s.begin(), ::tolower);
// Result depends on global locale!
```

**Locale-independent (ASCII-only):**
```cpp
char to_lower_ascii(char c) {
    return (c >= 'A' && c <= 'Z') ? c + 32 : c;
}
```

**For Unicode, use ICU:**
```cpp
icu::UnicodeString s = icu::UnicodeString::fromUTF8("STRASSE");
s.toLower();  // Correctly handles German ß
```

---

## 8. Unicode and Internationalization

### 8.1 The Unicode Problem

**std::string is byte-oriented:**
```cpp
std::string s = u8"Hello 世界";
s.size();     // Returns 11 (bytes), not 7 (characters)
s[6];         // Might be middle of multi-byte character!
```

**Key concepts:**
- **Code unit**: Basic storage unit (1 byte UTF-8, 2 bytes UTF-16, 4 bytes UTF-32)
- **Code point**: Unicode character value (U+0000 to U+10FFFF)
- **Grapheme cluster**: User-perceived character (e.g., "é" can be 1 or 2 code points)

### 8.2 UTF-8 Best Practices

**Recommendation: Store as UTF-8 in `std::string`**

**Why UTF-8:**
- ASCII-compatible
- Variable-length (efficient for Western text)
- Self-synchronizing
- Industry standard for interchange

**Manual UTF-8 iteration:**
```cpp
void iterate_utf8_code_points(std::string_view utf8) {
    for (size_t i = 0; i < utf8.size(); ) {
        unsigned char c = utf8[i];
        size_t len = 1;
        
        if (c < 0x80) len = 1;  // ASCII
        else if ((c & 0xE0) == 0xC0) len = 2;
        else if ((c & 0xF0) == 0xE0) len = 3;
        else if ((c & 0xF8) == 0xF0) len = 4;
        
        // Process code point: utf8.substr(i, len)
        i += len;
    }
}
```

**Never split on arbitrary byte boundaries:**
```cpp
// WRONG
std::string utf8 = u8"世界";
std::string half = utf8.substr(0, 2);  // Might split multi-byte character!

// Correct: use library for substring
```

### 8.3 Unicode Libraries

**ICU (International Components for Unicode)**
- Industry-standard, comprehensive
- Normalization, collation, case mapping, breaking
- Large dependency (~25 MB)

```cpp
#include <unicode/unistr.h>
#include <unicode/brkiter.h>

// Case conversion
icu::UnicodeString s = icu::UnicodeString::fromUTF8("straße");
s.toUpper();  // "STRASSE" (correctly handles German ß)

// Grapheme cluster iteration
UErrorCode status = U_ZERO_ERROR;
icu::BreakIterator* it = icu::BreakIterator::createCharacterInstance(
    icu::Locale::getUS(), status);
it->setText(s);
int32_t p = it->first();
while (p != icu::BreakIterator::DONE) {
    // Process grapheme cluster
    p = it->next();
}
delete it;
```

**Boost.Text**
- Modern C++ API
- Normalization, case mapping, segmentation
- Smaller than ICU

```cpp
#include <boost/text/case_mapping.hpp>
std::string s = "straße";
std::string upper = boost::text::to_upper(s);
```

**When to use:**
- **Simple ASCII**: std::string is sufficient
- **UTF-8 storage, ASCII operations**: std::string
- **Counting characters**: ICU or Boost.Text
- **Case conversion**: ICU or Boost.Text
- **Sorting/collation**: ICU (locale-aware)
- **Text segmentation**: ICU or Boost.Text
- **Regular expressions**: ICU (Unicode-aware)

### 8.4 Normalization

**Unicode allows multiple representations:**
```cpp
std::string s1 = "café";  // U+00E9 (precomposed é)
std::string s2 = "café";  // U+0065 U+0301 (e + combining acute)
// s1 == s2 is false! Different byte sequences
```

**Solution: Normalize before comparing:**
```cpp
#include <unicode/normalizer2.h>

std::string normalize_nfc(std::string_view utf8) {
    UErrorCode status = U_ZERO_ERROR;
    const icu::Normalizer2* nfc = icu::Normalizer2::getNFCInstance(status);
    
    icu::UnicodeString src = icu::UnicodeString::fromUTF8(
        icu::StringPiece(utf8.data(), utf8.size()));
    icu::UnicodeString result = nfc->normalize(src, status);
    
    std::string output;
    result.toUTF8String(output);
    return output;
}
```

**Normal forms:**
- **NFC**: Canonical composition (most common)
- **NFD**: Canonical decomposition
- **NFKC**: Compatibility composition
- **NFKD**: Compatibility decomposition

### 8.5 Collation (Locale-Aware Sorting)

**Simple byte comparison fails for many languages:**
```cpp
std::vector<std::string> words = {"ä", "z", "a"};
std::sort(words.begin(), words.end());
// Result: {"a", "z", "ä"} - wrong for German!
```

**ICU collation:**
```cpp
#include <unicode/coll.h>

UErrorCode status = U_ZERO_ERROR;
icu::Collator* coll = icu::Collator::createInstance(
    icu::Locale::getGerman(), status);

std::sort(words.begin(), words.end(), [&](const std::string& a, const std::string& b) {
    UErrorCode s = U_ZERO_ERROR;
    return coll->compareUTF8(a, b, s) < 0;
});
// Result: {"a", "ä", "z"} - correct for German

delete coll;
```

---

## 9. Regular Expressions

### 9.1 `<regex>` Standard Library

**Basic usage:**
```cpp
#include <regex>

std::string text = "Email: test@example.com";
std::regex pattern(R"([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})");

// Search
std::smatch matches;
if (std::regex_search(text, matches, pattern)) {
    std::string email = matches[0];  // "test@example.com"
}

// Match entire string
if (std::regex_match(text, pattern)) {
    // Entire text matches pattern
}

// Replace
std::string result = std::regex_replace(text, pattern, "[REDACTED]");
```

**Capture groups:**
```cpp
std::regex pattern(R"((\w+)@(\w+\.\w+))");
std::smatch matches;
if (std::regex_search(text, matches, pattern)) {
    std::string full = matches[0];   // "test@example.com"
    std::string user = matches[1];   // "test"
    std::string domain = matches[2]; // "example.com"
}
```

**Iterator usage:**
```cpp
std::string text = "one@a.com, two@b.com, three@c.com";
std::regex pattern(R"([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})");

std::sregex_iterator it(text.begin(), text.end(), pattern);
std::sregex_iterator end;

while (it != end) {
    std::cout << it->str() << '\n';
    ++it;
}
```

**Performance warning:** `std::regex` is notoriously slow and can suffer from catastrophic backtracking. For performance-critical code, consider alternatives.

### 9.2 Alternative Regex Libraries

**RE2 (Google) - Linear time, safe from backtracking:**
```cpp
#include <re2/re2.h>

std::string text = "test@example.com";
std::string email;
if (RE2::PartialMatch(text, R"((\S+@\S+))", &email)) {
    // email = "test@example.com"
}
```

**PCRE2 - Perl-compatible, feature-rich:**
```cpp
#include <pcre2.h>
// More complex API, but very powerful and fast
```

**Performance comparison:** RE2 > PCRE2 > std::regex

**When to use:**
- **Simple patterns, not performance-critical**: `std::regex`
- **Untrusted patterns**: RE2 (safe from DoS)
- **Complex patterns, high performance**: PCRE2
- **Perl compatibility**: PCRE2

---

## 10. String Streams and I/O

### 10.1 String Streams

**`std::ostringstream` (output):**
```cpp
#include <sstream>
std::ostringstream oss;
oss << "Value: " << 42 << ", hex: " << std::hex << 255;
std::string result = oss.str();  // "Value: 42, hex: ff"
```

**`std::istringstream` (input):**
```cpp
std::istringstream iss("42 3.14 hello");
int n;
double d;
std::string word;
iss >> n >> d >> word;  // n=42, d=3.14, word="hello"

if (iss.fail()) {
    // Parsing error
}
```

**`std::stringstream` (bidirectional):**
```cpp
std::stringstream ss;
ss << "data";
std::string s = ss.str();
ss.str("");  // Clear contents
ss.clear();  // Clear error flags
ss << "new data";
```

**Reusing stream:**
```cpp
std::ostringstream oss;
for (int i = 0; i < 10; ++i) {
    oss.str("");           // Clear buffer
    oss.clear();           // Clear state
    oss << "Item " << i;
    std::string item = oss.str();
}
```

### 10.2 File I/O

**Reading entire file:**
```cpp
#include <fstream>
#include <sstream>

// Method 1: Using stringstream
std::ifstream file("data.txt");
std::stringstream buffer;
buffer << file.rdbuf();
std::string content = buffer.str();

// Method 2: Using iterators
std::ifstream file("data.txt");
std::string content((std::istreambuf_iterator<char>(file)),
                     std::istreambuf_iterator<char>());

// Method 3: Using getline (line by line)
std::ifstream file("data.txt");
std::string line, content;
while (std::getline(file, line)) {
    content += line + '\n';
}
```

**Binary file handling:**
```cpp
// Read binary
std::ifstream file("data.bin", std::ios::binary);
std::string data((std::istreambuf_iterator<char>(file)),
                  std::istreambuf_iterator<char>());

// Write binary
std::ofstream file("data.bin", std::ios::binary);
file.write(data.data(), data.size());
```

**Writing to file:**
```cpp
std::ofstream file("output.txt");
file << "Line 1\n";
file << "Line 2\n";
```

### 10.3 Console I/O

**Reading strings:**
```cpp
std::string word;
std::cin >> word;  // Reads until whitespace

std::string line;
std::getline(std::cin, line);  // Reads entire line
```

**Formatted output:**
```cpp
#include <iomanip>
std::cout << std::setw(10) << std::setfill('0') << 42;  // "0000000042"
std::cout << std::fixed << std::setprecision(2) << 3.14159;  // "3.14"
```

---

## 11. Third-Party String Libraries

### 11.1 `{fmt}` Library (basis for std::format)

**Installation:** Header-only or compiled library

**Basic formatting:**
```cpp
#include <fmt/core.h>

std::string s = fmt::format("The answer is {}", 42);
std::string s = fmt::format("{0} {1} {0}", "A", "B");  // "A B A"
std::string s = fmt::format("{:.2f}", 3.14159);        // "3.14"
```

**Advanced features:**
```cpp
// Named arguments
std::string s = fmt::format("Hello, {name}!", fmt::arg("name", "World"));

// Custom types
struct Point { int x, y; };
template<> struct fmt::formatter<Point> {
    constexpr auto parse(format_parse_context& ctx) { return ctx.begin(); }
    auto format(const Point& p, format_context& ctx) {
        return fmt::format_to(ctx.out(), "({}, {})", p.x, p.y);
    }
};
```

**Performance:** Much faster than iostreams, comparable to printf

### 11.2 Boost.StringAlgo

**Comprehensive string algorithms:**
```cpp
#include <boost/algorithm/string.hpp>

std::string s = "  Hello World  ";

// Trimming
boost::trim(s);                    // In-place
std::string t = boost::trim_copy(s);  // Copy version

// Case conversion
boost::to_upper(s);
boost::to_lower(s);

// Predicates
bool b = boost::starts_with(s, "Hello");
bool b = boost::ends_with(s, "World");
bool b = boost::contains(s, "lo Wo");

// Searching
boost::find_first(s, "World");     // Returns iterator range
boost::ifind_first(s, "world");    // Case-insensitive

// Replacing
boost::replace_all(s, "World", "Universe");
boost::ireplace_all(s, "hello", "Hi");  // Case-insensitive

// Splitting
std::vector<std::string> tokens;
boost::split(tokens, s, boost::is_any_of(" ,"));

// Joining
std::string joined = boost::join(tokens, "-");

// Erasing
boost::erase_all(s, " ");          // Remove all spaces
boost::erase_first(s, "Hello");    // Remove first occurrence
```

**Classification predicates:**
```cpp
boost::is_space(), boost::is_alpha(), boost::is_digit()
boost::is_alnum(), boost::is_lower(), boost::is_upper()
```

### 11.3 Boost.Text

**Modern Unicode library:**
```cpp
#include <boost/text/case_mapping.hpp>
#include <boost/text/normalize.hpp>
#include <boost/text/segmentation.hpp>

// Case mapping
std::string s = "straße";
std::string upper = boost::text::to_upper(s);

// Normalization
std::string normalized = boost::text::normalize_to_nfc(s);

// Grapheme segmentation
boost::text::grapheme_view gv(s);
for (auto grapheme : gv) {
    // Iterate over grapheme clusters
}
```

### 11.4 ICU (International Components for Unicode)

**Most comprehensive Unicode library:**
```cpp
#include <unicode/unistr.h>
#include <unicode/coll.h>
#include <unicode/brkiter.h>

// String creation
icu::UnicodeString s = icu::UnicodeString::fromUTF8("Hello 世界");

// Case conversion
s.toUpper();
s.toLower();
s.toTitle();

// Normalization
UErrorCode status = U_ZERO_ERROR;
const icu::Normalizer2* nfc = icu::Normalizer2::getNFCInstance(status);
icu::UnicodeString normalized = nfc->normalize(s, status);

// Collation (locale-aware sorting)
icu::Collator* coll = icu::Collator::createInstance(
    icu::Locale::getGerman(), status);
if (coll->compare(s1, s2) < 0) {
    // s1 < s2 in German locale
}

// Breaking (segmentation)
icu::BreakIterator* bi = icu::BreakIterator::createCharacterInstance(
    icu::Locale::getDefault(), status);
bi->setText(s);
int32_t pos = bi->first();
while (pos != icu::BreakIterator::DONE) {
    pos = bi->next();
}

// Conversion to UTF-8
std::string utf8;
s.toUTF8String(utf8);
```

**When to use ICU:**
- Full internationalization support needed
- Locale-aware operations (sorting, formatting)
- Complex text rendering
- Production-grade Unicode handling

### 11.5 abseil (Google)

**String utilities:**
```cpp
#include <absl/strings/string_view.h>
#include <absl/strings/str_cat.h>
#include <absl/strings/str_split.h>
#include <absl/strings/str_join.h>

// Efficient concatenation
std::string s = absl::StrCat("Hello", " ", "World", " ", 42);

// Splitting
std::vector<absl::string_view> parts = absl::StrSplit("a,b,c", ',');

// Joining
std::string joined = absl::StrJoin(parts, "-");

// String formatting (absl::StrFormat)
std::string s = absl::StrFormat("Value: %d", 42);
```

### 11.6 Library Selection Guide

| Need | Recommended Library | Alternative |
|------|-------------------|-------------|
| Modern formatting | `std::format` (C++20) | `{fmt}` |
| String algorithms | Boost.StringAlgo | Manual implementations |
| Unicode support | ICU | Boost.Text |
| Fast regex | RE2 | PCRE2 |
| JSON/XML parsing | nlohmann/json, pugixml | RapidJSON, tinyxml2 |
| High-performance strings | abseil | folly (Facebook) |

---

## 12. Implementation Differences

### 12.1 Standard Library Implementations

**libstdc++ (GCC):**
- SSO buffer: 15 bytes
- Growth factor: 2x
- ABI: Stable since GCC 5 (dual ABI support)
- COW removed in C++11 mode

**libc++ (Clang/LLVM):**
- SSO buffer: 22-23 bytes (platform-dependent)
- Growth factor: 1.5x
- ABI: Versioned, more aggressive optimization
- Always deep copy (never COW)

**MSVC STL:**
- SSO buffer: 15 bytes
- Growth factor: 1.5x
- ABI: Stable within major versions
- Debug iterators in debug builds

### 12.2 ABI Compatibility

**Important:** Different standard libraries are NOT binary compatible

**Implications:**
- Can't pass `std::string` across library boundaries with different STL implementations
- Must rebuild all code when switching compilers/STL versions
- Dynamic libraries must use same STL implementation

**Safe practices:**
- Use C-compatible interfaces at library boundaries (`const char*`, lengths)
- Use `string_view` for read-only interfaces (still requires same ABI)
- Document STL implementation requirements

### 12.3 Debug vs Release Builds

**Debug builds often include:**
- Iterator debugging (detects invalid iterators)
- Bounds checking
- Extra assertions
- Slower performance

**Release builds:**
- Optimizations enabled
- Minimal checks
- Smaller binary size

**Important:** Debug and release builds are ABI-incompatible in MSVC

---

## 13. Practical Examples and Patterns

### 13.1 String Builder Pattern

```cpp
class StringBuilder {
    std::string buffer_;
    
public:
    StringBuilder& append(std::string_view sv) {
        buffer_ += sv;
        return *this;
    }
    
    StringBuilder& append(char c) {
        buffer_ += c;
        return *this;
    }
    
    StringBuilder& append(int value) {
        buffer_ += std::to_string(value);
        return *this;
    }
    
    void reserve(size_t capacity) {
        buffer_.reserve(capacity);
    }
    
    std::string build() {
        return std::move(buffer_);
    }
    
    void clear() {
        buffer_.clear();
    }
};

// Usage
StringBuilder sb;
sb.reserve(100);
sb.append("Hello").append(' ').append("World").append('!');
std::string result = sb.build();
```

### 13.2 String Interning (Deduplication)

```cpp
class StringInterner {
    std::unordered_set<std::string> pool_;
    
public:
    std::string_view intern(std::string_view sv) {
        auto [it, inserted] = pool_.emplace(sv);
        return *it;
    }
    
    void clear() {
        pool_.clear();
    }
};

// Usage: Reduce memory for repeated strings
StringInterner interner;
auto s1 = interner.intern("common");
auto s2 = interner.intern("common");
assert(s1.data() == s2.data());  // Same underlying storage
```

### 13.3 Case-Insensitive String Comparator

```cpp
struct CaseInsensitiveCompare {
    bool operator()(std::string_view a, std::string_view b) const {
        return std::lexicographical_compare(
            a.begin(), a.end(), b.begin(), b.end(),
            [](unsigned char c1, unsigned char c2) {
                return std::tolower(c1) < std::tolower(c2);
            }
        );
    }
};

// Usage with map
std::map<std::string, int, CaseInsensitiveCompare> map;
map["Hello"] = 1;
map["hello"] = 2;  // Overwrites because case-insensitive
assert(map.size() == 1);
```

### 13.4 URL Encoding/Decoding

```cpp
std::string url_encode(std::string_view sv) {
    std::ostringstream oss;
    oss << std::hex << std::uppercase << std::setfill('0');
    
    for (unsigned char c : sv) {
        if (std::isalnum(c) || c == '-' || c == '_' || c == '.' || c == '~') {
            oss << c;
        } else {
            oss << '%' << std::setw(2) << int(c);
        }
    }
    return oss.str();
}

std::string url_decode(std::string_view sv) {
    std::string result;
    result.reserve(sv.size());
    
    for (size_t i = 0; i < sv.size(); ++i) {
        if (sv[i] == '%' && i + 2 < sv.size()) {
            int value;
            std::istringstream iss(std::string(sv.substr(i + 1, 2)));
            if (iss >> std::hex >> value) {
                result += static_cast<char>(value);
                i += 2;
            }
        } else if (sv[i] == '+') {
            result += ' ';
        } else {
            result += sv[i];
        }
    }
    return result;
}
```

### 13.5 Token Parser with Error Handling

```cpp
struct Token {
    enum Type { Number, Identifier, Symbol } type;
    std::string value;
    size_t position;
};

class Tokenizer {
    std::string_view input_;
    size_t pos_ = 0;
    
public:
    Tokenizer(std::string_view input) : input_(input) {}
    
    std::optional<Token> next() {
        // Skip whitespace
        while (pos_ < input_.size() && std::isspace(input_[pos_])) {
            ++pos_;
        }
        
        if (pos_ >= input_.size()) {
            return std::nullopt;
        }
        
        Token token;
        token.position = pos_;
        
        if (std::isdigit(input_[pos_])) {
            // Parse number
            size_t start = pos_;
            while (pos_ < input_.size() && std::isdigit(input_[pos_])) {
                ++pos_;
            }
            token.type = Token::Number;
            token.value = std::string(input_.substr(start, pos_ - start));
        } else if (std::isalpha(input_[pos_])) {
            // Parse identifier
            size_t start = pos_;
            while (pos_ < input_.size() && std::isalnum(input_[pos_])) {
                ++pos_;
            }
            token.type = Token::Identifier;
            token.value = std::string(input_.substr(start, pos_ - start));
        } else {
            // Symbol
            token.type = Token::Symbol;
            token.value = std::string(1, input_[pos_++]);
        }
        
        return token;
    }
};
```

### 13.6 Safe Path Handling

```cpp
#include <filesystem>

class SafePath {
    std::filesystem::path path_;
    
public:
    explicit SafePath(std::string_view base) : path_(base) {}
    
    bool append(std::string_view component) {
        // Validate: no path traversal, no absolute paths
        if (component.find("..") != std::string_view::npos ||
            component.find('/') != std::string_view::npos ||
            component.find('\\') != std::string_view::npos ||
            component.empty() ||
            component[0] == '.') {
            return false;
        }
        
        path_ /= component;
        return true;
    }
    
    std::string string() const {
        return path_.string();
    }
    
    bool exists() const {
        return std::filesystem::exists(path_);
    }
};

// Usage
SafePath path("/var/data");
if (path.append(user_input)) {
    // Safe to use path.string()
}
```

### 13.7 Command-Line Argument Parser

```cpp
class ArgParser {
    std::map<std::string, std::string> args_;
    std::vector<std::string> positional_;
    
public:
    void parse(int argc, char* argv[]) {
        for (int i = 1; i < argc; ++i) {
            std::string_view arg = argv[i];
            
            if (arg.starts_with("--")) {
                // Long option
                auto eq_pos = arg.find('=');
                if (eq_pos != std::string_view::npos) {
                    std::string key(arg.substr(2, eq_pos - 2));
                    std::string value(arg.substr(eq_pos + 1));
                    args_[key] = value;
                } else {
                    args_[std::string(arg.substr(2))] = "true";
                }
            } else if (arg.starts_with("-")) {
                // Short option
                if (i + 1 < argc && argv[i + 1][0] != '-') {
                    args_[std::string(arg.substr(1))] = argv[++i];
                } else {
                    args_[std::string(arg.substr(1))] = "true";
                }
            } else {
                // Positional argument
                positional_.emplace_back(arg);
            }
        }
    }
    
    std::optional<std::string> get(const std::string& key) const {
        auto it = args_.find(key);
        return it != args_.end() ? std::optional{it->second} : std::nullopt;
    }
    
    const std::vector<std::string>& positional() const {
        return positional_;
    }
};
```

---

## 14. Testing and Validation

### 14.1 Unit Test Examples

```cpp
#include <cassert>

void test_string_operations() {
    // Basic construction
    std::string s1;
    assert(s1.empty());
    assert(s1.size() == 0);
    
    std::string s2 = "hello";
    assert(s2.size() == 5);
    assert(s2 == "hello");
    
    // Embedded nulls
    std::string s3("ab\0cd", 5);
    assert(s3.size() == 5);
    assert(s3[2] == '\0');
    
    // Substring
    std::string s4 = s2.substr(1, 3);
    assert(s4 == "ell");
    
    // Search
    assert(s2.find("ll") == 2);
    assert(s2.find("x") == std::string::npos);
    
    // Modification
    s2 += " world";
    assert(s2 == "hello world");
    
    // Move semantics
    std::string s5 = std::move(s2);
    assert(s5 == "hello world");
    assert(s2.empty());  // s2 moved-from
}

void test_string_view_safety() {
    std::string s = "data";
    std::string_view sv = s;
    assert(sv == "data");
    
    s.clear();
    // sv is now dangling - don't use it!
}
```

### 14.2 Edge Cases to Test

**Empty strings:**
```cpp
std::string s;
assert(s.empty());
assert(s.size() == 0);
assert(s.c_str()[0] == '\0');
assert(s.data() == s.c_str());
```

**Single character:**
```cpp
std::string s = "x";
assert(s.size() == 1);
assert(s.front() == 'x');
assert(s.back() == 'x');
```

**Embedded nulls:**
```cpp
std::string s("a\0b", 3);
assert(s.size() == 3);
assert(strlen(s.c_str()) == 1);  // C-string functions fail
```

**Large strings (beyond SSO):**
```cpp
std::string s(100, 'x');
assert(s.size() == 100);
assert(s.capacity() >= 100);
```

**Boundary conditions:**
```cpp
std::string s = "test";
assert(s[s.size()] == '\0');     // Valid access
assert(s.substr(4, 0).empty());  // Empty substring
```

---

## 15. Quick Reference Summary

### 15.1 When to Use Each Type

| Type | Use Case |
|------|----------|
| `std::string` | Ownership, storage, modification |
| `const std::string&` | Read-only function parameter (same type) |
| `std::string_view` | Read-only parameter (any string type) |
| `const char*` | C API interop, string literals |
| `std::u8string` | Explicit UTF-8 (C++20+) |
| `std::wstring` | Windows APIs, UTF-16/32 |

### 15.2 Header Dependencies

| Header | Contents |
|--------|----------|
| `<string>` | `std::string`, `std::wstring`, `std::u8/u16/u32string` |
| `<string_view>` | `std::string_view` |
| `<cstring>` | C string functions (`strlen`, `strcpy`, etc.) |
| `<sstream>` | String streams |
| `<format>` | `std::format` (C++20) |
| `<charconv>` | `std::to_chars`, `std::from_chars` (C++17) |
| `<regex>` | Regular expressions |
| `<locale>` | Locale support |
| `<cctype>` | Character classification |

### 15.3 Best Practices Checklist

✓ **Use `std::string` for ownership**, `string_view` for read-only parameters  
✓ **Reserve capacity** when final size is known  
✓ **Use `std::format`** (C++20) or `{fmt}` for formatting  
✓ **Use `std::to_chars`/`from_chars`** for fast numeric conversions  
✓ **Never store `string_view`** in data structures  
✓ **Use `std::move`** when transferring ownership  
✓ **Validate external input** to prevent injection and encoding issues  
✓ **Use UTF-8** for Unicode storage  
✓ **Use ICU/Boost.Text** for proper Unicode handling  
✓ **Avoid C-string functions** on `std::string` data  
✓ **Profile before optimizing**: Modern implementations are fast  
✓ **Test edge cases**: empty strings, embedded nulls, large strings  

### 15.4 Common Pitfalls to Avoid

✗ Returning `string_view` to temporaries  
✗ Storing `string_view` in class members  
✗ Unnecessary string copies in parameters  
✗ Inefficient concatenation (use `reserve` + `append`)  
✗ Ignoring search function return values  
✗ Assuming 1 byte = 1 character (UTF-8)  
✗ Using `strlen()` on binary data  
✗ Direct user input in SQL queries  
✗ Mixing different string encodings  
✗ Locale-dependent operations in performance-critical code  

---

## 16. Conclusion

This comprehensive guide covered the complete C++ string ecosystem:

**Foundations**: C-strings, `std::string`, `std::string_view`, Unicode types, and character encoding fundamentals

**Internals**: Memory model, SSO, copy/move semantics, iterator invalidation, traits, allocators, and thread safety

**Complete API**: All constructors, modifiers, operations, conversions, and utility functions with detailed explanations

**Practical Operations**: Concatenation, formatting, splitting, trimming, searching, replacing, and conversions with multiple approaches

**Performance**: Avoiding copies, SSO impact, allocator strategies, threading patterns, and optimization techniques

**Security**: Buffer safety, input validation, injection prevention, encoding validation, and safe practices

**Unicode**: UTF-8 handling, normalization, collation, and when to use ICU or Boost.Text

**Advanced Topics**: Regular expressions, string streams, implementation differences, and ABI considerations

**Libraries**: Comprehensive coverage of `{fmt}`, Boost.StringAlgo, Boost.Text, ICU, abseil, and when to use each

**Patterns**: Practical examples including builders, interners, parsers, and safe path handling

**Key Takeaway**: `std::string` is powerful, safe, and efficient when used correctly. Combine it with `std::string_view` for flexibility, understand the underlying mechanisms, leverage modern features (`std::format`, `std::to_chars`, move semantics), and use specialized libraries (ICU, Boost) for advanced Unicode requirements. Always validate input, test edge cases, and profile before optimizing.
