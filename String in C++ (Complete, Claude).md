# C++ String Deep Dive: Complete Reference

## 1. Foundations: String Types in C++

### 1.1 C-Style Strings (`char*`, `const char*`)

C-style strings are contiguous arrays of `char` terminated by a NUL byte (`'\0'`). They have no inherent size information—functions must scan for the terminator.

```cpp
const char* str = "hello";  // 6 bytes: 'h','e','l','l','o','\0'
char buffer[10] = "test";   // mutable, fixed capacity
```

**Key characteristics:**
- **No ownership semantics**: raw pointers don't manage memory
- **Manual lifetime management**: caller handles allocation/deallocation
- **Unsafe by default**: buffer overruns, missing terminators cause UB
- **O(n) length calculation**: `strlen()` must scan the entire string
- **Memory layout**: contiguous, terminated, no metadata

**When to use:** Interop with C APIs, string literals, performance-critical code where you control lifetimes.

### 1.2 `std::string` — The Standard String Class

`std::string` is an alias for `std::basic_string<char>`. It manages dynamic memory, tracks size/capacity, and provides value semantics.

```cpp
std::string s1 = "hello";         // owns copy of data
std::string s2 = s1;              // deep copy (COW deprecated)
std::string s3 = std::move(s1);   // transfer ownership, s1 empty
```

**Key characteristics:**
- **Ownership**: RAII—automatically manages heap allocation
- **Size tracking**: O(1) `.size()` via stored member
- **Dynamic growth**: automatic reallocation when capacity exceeded
- **Value semantics**: copies are independent
- **NUL-terminated**: `.c_str()` always returns `'\0'`-terminated data

### 1.3 `std::string_view` (C++17)

Non-owning reference to a character sequence. Think of it as `{const char*, size_t}`.

```cpp
std::string s = "hello world";
std::string_view sv = s;           // no copy, just reference
std::string_view sub = sv.substr(0, 5);  // still no copy
```

**Critical warning:** Does NOT extend lifetime. Dangling references are UB.

```cpp
std::string_view danger() {
    std::string local = "temporary";
    return local;  // UB! Returns view to destroyed object
}
```

**When to use:**
- Function parameters that read strings (avoids copies)
- Substring operations without allocation
- **Never** for storage or return values unless lifetime is guaranteed

### 1.4 Wide and Unicode String Types

```cpp
std::wstring    // wchar_t (UTF-16 on Windows, UTF-32 on Unix)
std::u8string   // char8_t (C++20), UTF-8
std::u16string  // char16_t, UTF-16
std::u32string  // char32_t, UTF-32
```

**Important distinction:**
- **Code unit**: single element of encoding (1 byte for UTF-8, 2 for UTF-16)
- **Code point**: Unicode scalar value (U+0000 to U+10FFFF)
- **Grapheme cluster**: user-perceived character (may be multiple code points)

```cpp
// UTF-8: "é" can be 1 code point (U+00E9) or 2 (e + combining acute)
std::string s = "café";  // 5 bytes if precomposed, 6 if decomposed
s.size();                // returns bytes, NOT characters
```

**Reality check:** `std::string` does byte operations, not Unicode-aware operations. For proper Unicode handling, use ICU or Boost.Text.

---

## 2. Memory Model and Internals

### 2.1 Memory Layout

Typical `std::string` layout (simplified):

```cpp
class string {
    char*  ptr;       // pointer to data (heap or internal buffer)
    size_t size;      // current number of characters
    size_t capacity;  // allocated space (excluding NUL terminator)
};
```

Actual layout varies by implementation and SSO strategy.

### 2.2 Small-String Optimization (SSO)

**Concept:** Store short strings directly in the object, avoiding heap allocation.

```cpp
// Typical SSO threshold: 15-23 characters (implementation-dependent)
std::string short_str = "tiny";        // stored inline, no allocation
std::string long_str = "this is a very long string";  // heap allocation
```

**Implementation details:**
- **libstdc++ (GCC)**: 15 bytes SSO buffer
- **libc++ (Clang)**: 22 bytes SSO buffer  
- **MSVC STL**: 15 bytes SSO buffer

**How it works (common approach):**
```cpp
union {
    struct {
        char*  ptr;
        size_t size;
        size_t capacity;
    } heap_data;
    
    struct {
        char   buffer[23];
        char   size;  // high bit indicates heap mode
    } sso_data;
};
```

**Performance impact:**
- Short strings: no allocations, cache-friendly
- Copy cost: 24-32 bytes (stack copy) vs pointer copy
- Move cost: entire object copied (can't steal inline buffer)

**ABI implications:** SSO size is ABI-locked. Changing it breaks binary compatibility.

### 2.3 Capacity and Reallocation

```cpp
std::string s;
s.reserve(100);      // allocate at least 100 bytes, size stays 0
s.capacity();        // >= 100
s.size();            // 0

s += "text";         // size = 4, capacity unchanged
```

**Growth strategy:** Typically geometric (1.5x or 2x) to achieve amortized O(1) append.

```cpp
for (int i = 0; i < 1000; ++i) {
    s += 'x';  // ~10 reallocations with 2x growth
}
```

**Key insight:** Each reallocation is O(n) copy. Use `.reserve()` if final size is known.

### 2.4 Copy and Move Semantics

**Copy (deep):**
```cpp
std::string s1 = "data";
std::string s2 = s1;     // allocates new buffer, copies content
```

**Move (transfer):**
```cpp
std::string s1 = "data";
std::string s2 = std::move(s1);  // s2 takes s1's buffer, s1 becomes empty
// s1 is in valid but unspecified state (typically empty)
```

**SSO caveat:** If string fits in SSO buffer, move still copies the bytes (can't "steal" inline storage).

**Copy elision (C++17 guaranteed):**
```cpp
std::string make() { return std::string("result"); }
std::string s = make();  // no copy, no move—direct construction
```

### 2.5 Iterator/Pointer/Reference Invalidation

**Rules:**
- **Any modification that changes capacity**: invalidates all iterators, pointers, references
- **Non-capacity-changing modifications**: iterators/pointers may be invalidated, references to elements remain valid
- **`reserve()`, `shrink_to_fit()`**: may reallocate → invalidate all

```cpp
std::string s = "hello";
const char* ptr = s.c_str();
s += " world";               // if reallocation occurs, ptr is dangling
```

**Safe patterns:**
```cpp
s.reserve(100);
const char* ptr = s.c_str();  // safe until next capacity change
```

### 2.6 `c_str()` vs `data()`

**Before C++11:**
- `c_str()`: NUL-terminated, read-only
- `data()`: may not be NUL-terminated

**C++11 and later:**
- Both return NUL-terminated `char*`/`const char*`
- `data()` is non-const in C++17
- Accessing `s[s.size()]` is legal and returns `'\0'`

```cpp
std::string s = "test";
assert(s.data()[s.size()] == '\0');  // guaranteed since C++11
```

### 2.7 Character Traits and Allocators

**Traits:** Define character operations (comparison, copying, etc.)

```cpp
template<class CharT, 
         class Traits = std::char_traits<CharT>,
         class Allocator = std::allocator<CharT>>
class basic_string;
```

**Custom allocator example:**
```cpp
using MyString = std::basic_string<char, std::char_traits<char>, 
                                    std::pmr::polymorphic_allocator<char>>;
```

**Impact:** Different allocators can't be compared/assigned without reallocation.

### 2.8 Thread Safety

**Const operations:** Thread-safe (multiple readers)
**Non-const operations:** Not thread-safe (data races)

```cpp
const std::string s = "shared";
// Safe: multiple threads can call s.size(), s[0], s.data()

std::string s2 = "mutable";
// NOT safe: one thread calls s2 += "x" while another reads s2
```

**Safe patterns:**
- Read-only sharing: pass by `const&` or `string_view`
- Exclusive ownership: one thread owns, others don't touch
- Synchronization: mutex/atomic for shared mutable strings

---

## 3. Complete API Reference

### 3.1 Construction and Assignment

| Method | Complexity | Description | Example |
|--------|-----------|-------------|---------|
| `string()` | O(1) | Empty string | `std::string s;` |
| `string(const char*)` | O(n) | From C-string | `std::string s("text");` |
| `string(const char*, size_t)` | O(n) | From buffer + length | `std::string s(data, 10);` |
| `string(size_t, char)` | O(n) | Repeat char n times | `std::string s(5, 'x');` |
| `string(iterator, iterator)` | O(n) | From range | `std::string s(v.begin(), v.end());` |
| `string(const string&)` | O(n) | Copy constructor | `std::string s2(s1);` |
| `string(string&&)` | O(1)* | Move constructor | `std::string s2(std::move(s1));` |
| `string(initializer_list)` | O(n) | From init list | `std::string s{'a','b','c'};` |
| `string(string_view)` (C++17) | O(n) | From string_view | `std::string s(sv);` |
| `operator=` | O(n) | Assignment | `s1 = s2;` |
| `assign(...)` | O(n) | Various overloads | `s.assign(10, 'a');` |

**Notes:**
- *Move constructor is O(1) for heap-allocated strings, but O(n) for SSO strings (must copy inline buffer)
- Constructing with `nullptr` is UB
- String literals with embedded nulls: Use `std::string s("ab\0cd", 5)` or `"ab\0cd"s` literal

**Complete constructor overloads (19 total in C++20):**
```cpp
// 1. Default
std::string s;

// 2-4. From allocator
std::string s(alloc);
std::string s(str, alloc);
std::string s(std::move(str), alloc);

// 5. Substring
std::string s(str, pos);
std::string s(str, pos, len);

// 6-7. From C-string
std::string s("text");
std::string s(cstr, len);

// 8. Fill
std::string s(10, 'x');

// 9. From range
std::string s(begin, end);

// 10. Copy/move
std::string s(other);
std::string s(std::move(other));

// 11. Initializer list
std::string s{'h','i'};

// 12-13. From string_view
std::string s(sv);
std::string s(sv, pos, len);

// 14-15. From string_view-like types
std::string s(sv_like);  // requires implicit conversion to string_view
```

**Edge case:** Constructing with `nullptr` is UB.

```cpp
const char* p = nullptr;
std::string s(p);  // UB!
```

### 3.2 Capacity Management

| Method | Complexity | Description | Example |
|--------|-----------|-------------|---------|
| `size()` / `length()` | O(1) | Number of chars | `s.size()` |
| `capacity()` | O(1) | Allocated space | `s.capacity()` |
| `empty()` | O(1) | True if size==0 | `if (s.empty())` |
| `reserve(n)` | O(n) | Ensure capacity≥n | `s.reserve(100);` |
| `shrink_to_fit()` | O(n) | Request capacity reduction | `s.shrink_to_fit();` |
| `resize(n)` | O(n) | Change size, pad with `'\0'` | `s.resize(10);` |
| `resize(n, char)` | O(n) | Change size, pad with char | `s.resize(10, 'x');` |
| `clear()` | O(1) | Set size=0 | `s.clear();` |
| `max_size()` | O(1) | Theoretical max size | `s.max_size()` |
| `get_allocator()` | O(1) | Returns copy of allocator | `auto a = s.get_allocator();` |

**Notes:**
- `reserve()` doesn't reduce capacity (use `shrink_to_fit()`)
- `shrink_to_fit()` is non-binding (implementation may ignore)
- `clear()` doesn't free memory, just sets size to 0

### 3.3 Element Access

| Method | Complexity | Description | Example |
|--------|-----------|-------------|---------|
| `operator[i]` | O(1) | Unchecked access | `char c = s[0];` |
| `at(i)` | O(1) | Checked access (throws) | `char c = s.at(0);` |
| `front()` | O(1) | First character | `s.front()` |
| `back()` | O(1) | Last character | `s.back()` |
| `data()` | O(1) | Pointer to buffer | `const char* p = s.data();` |
| `c_str()` | O(1) | NUL-terminated C-string | `printf("%s", s.c_str());` |

**Edge cases:**
- `s[s.size()]` is valid and returns `'\0'` (since C++11)
- `s[i]` with `i >= size()` (excluding `i == size()`) is UB
- `at()` throws `std::out_of_range` for invalid index
- `front()/back()` on empty string is UB

### 3.4 Modifiers

| Method | Complexity | Description | Example |
|--------|-----------|-------------|---------|
| `operator+=` | Amortized O(n) | Append | `s += "text";` |
| `append(...)` | Amortized O(n) | Append (many overloads) | `s.append(5, 'x');` |
| `push_back(char)` | Amortized O(1) | Append single char | `s.push_back('!');` |
| `insert(pos, ...)` | O(n) | Insert at position | `s.insert(0, "pre");` |
| `erase(pos, len)` | O(n) | Remove substring | `s.erase(0, 3);` |
| `replace(pos, len, ...)` | O(n) | Replace substring | `s.replace(0, 2, "ab");` |
| `pop_back()` | O(1) | Remove last char | `s.pop_back();` |
| `swap(other)` | O(1) | Exchange contents | `s1.swap(s2);` |
| `copy(dest, count, pos)` | O(n) | Copy substring to buffer | `char buf[10]; s.copy(buf, 5, 0);` |

**Performance tips:**
```cpp
// Inefficient: multiple reallocations
std::string s;
for (int i = 0; i < 1000; ++i) s += std::to_string(i);

// Efficient: pre-allocate
std::string s;
s.reserve(5000);
for (int i = 0; i < 1000; ++i) s += std::to_string(i);
```

### 3.5 String Operations

| Method | Complexity | Description | Example |
|--------|-----------|-------------|---------|
| `substr(pos, len)` | O(n) | Extract substring | `s.substr(0, 5)` |
| `compare(...)` | O(n) | Lexicographic compare | `s.compare("other")` |
| `find(str, pos)` | O(n*m) | Find first occurrence | `s.find("sub")` |
| `rfind(str, pos)` | O(n*m) | Find last occurrence | `s.rfind("sub")` |
| `find_first_of(chars)` | O(n*m) | Find first of any char | `s.find_first_of("aeiou")` |
| `find_last_of(chars)` | O(n*m) | Find last of any char | `s.find_last_of("aeiou")` |
| `find_first_not_of(chars)` | O(n*m) | First not matching | `s.find_first_not_of(" ")` |
| `find_last_not_of(chars)` | O(n*m) | Last not matching | `s.find_last_not_of(" ")` |
| `starts_with(...)` | O(n) | Prefix check (C++20) | `s.starts_with("pre")` |
| `ends_with(...)` | O(n) | Suffix check (C++20) | `s.ends_with(".txt")` |
| `contains(...)` | O(n*m) | Substring check (C++23) | `s.contains("sub")` |
| `resize_and_overwrite(n, op)` | O(n) | Resize with callback (C++23) | `s.resize_and_overwrite(10, operation);` |

**Return value:** `std::string::npos` (typically -1 as `size_t`) indicates not found.

```cpp
size_t pos = s.find("needle");
if (pos != std::string::npos) {
    // found at pos
}
```

### 3.6 Iterators

| Method | Description | Example |
|--------|-------------|---------|
| `begin() / end()` | Iterator range | `for (auto it = s.begin(); it != s.end(); ++it)` |
| `rbegin() / rend()` | Reverse iterators | `for (auto it = s.rbegin(); it != s.rend(); ++it)` |
| `cbegin() / cend()` | Const iterators | `auto it = s.cbegin();` |

**Range-based for:**
```cpp
for (char c : s) { /* process c */ }
```

### 3.8 Less Common but Important Methods

**`get_allocator()`** - Returns allocator copy:
```cpp
std::string s;
auto alloc = s.get_allocator();
char* p = alloc.allocate(10);  // allocate 10 chars
alloc.deallocate(p, 10);
```

**`copy(dest, count, pos = 0)`** - Copy substring to C-array:
```cpp
std::string s = "hello world";
char buffer[6];
size_t copied = s.copy(buffer, 5, 0);  // copy "hello", returns 5
buffer[copied] = '\0';  // must manually null-terminate!
```

**Critical:** `copy()` does NOT null-terminate. You must add `'\0'` yourself.

**`resize_and_overwrite(n, operation)` (C++23)** - High-performance resize with direct buffer access:
```cpp
std::string s;
s.resize_and_overwrite(100, [](char* buf, size_t n) {
    // Fill buf[0..n-1] with data
    int written = snprintf(buf, n, "formatted %d", 42);
    return written;  // return actual size used
});
```

**Use case:** Efficiently work with C APIs that write to buffers, avoiding unnecessary initialization and copy operations.

### 3.9 Non-Member Functions

| Function | Description | Example |
|----------|-------------|---------|
| `operator+` | Concatenation | `std::string r = s1 + s2;` |
| `operator==, !=, <, <=, >, >=` | Comparison | `if (s1 == s2)` |
| `operator<=>` (C++20) | Three-way comparison | `auto cmp = s1 <=> s2;` |
| `std::swap(s1, s2)` | Swap strings | `std::swap(s1, s2);` |
| `std::getline(stream, str, delim)` | Read line | `std::getline(std::cin, s);` |
| `std::stoi, stol, stoll, stof, stod` | String to number | `int n = std::stoi("123");` |
| `std::to_string(value)` | Number to string | `std::string s = std::to_string(42);` |
| `operator<<, operator>>` | Stream I/O | `std::cout << s;` |

**Conversion error handling:**
```cpp
try {
    int n = std::stoi("not a number");
} catch (const std::invalid_argument& e) {
    // conversion failed
} catch (const std::out_of_range& e) {
    // number too large
}
```

### 3.10 Complete Overload Variants (Reference)

Many `std::string` methods have multiple overloads. Here's a complete breakdown:

**`insert()` overloads (9 total):**
```cpp
s.insert(pos, str);                    // insert string at position
s.insert(pos, str, subpos, sublen);    // insert substring
s.insert(pos, cstr);                   // insert C-string
s.insert(pos, cstr, n);                // insert first n chars of C-string
s.insert(pos, n, c);                   // insert n copies of char c
iterator insert(const_iterator p, c);  // insert char before iterator
iterator insert(const_iterator p, n, c); // insert n chars before iterator
iterator insert(const_iterator p, first, last); // insert range
iterator insert(const_iterator p, ilist); // insert initializer list
```

**`erase()` overloads (3 total):**
```cpp
s.erase(pos, len);                     // erase substring
iterator erase(const_iterator position); // erase single char
iterator erase(const_iterator first, const_iterator last); // erase range
```

**`append()` overloads (8 total):**
```cpp
s.append(str);                         // append string
s.append(str, subpos, sublen);         // append substring
s.append(cstr);                        // append C-string
s.append(cstr, n);                     // append first n chars
s.append(n, c);                        // append n copies of char
s.append(first, last);                 // append iterator range
s.append(ilist);                       // append initializer list
s.append(sv);                          // append string_view (C++17)
```

**`replace()` overloads (14 total):**
```cpp
// Position-based replacements
s.replace(pos, len, str);
s.replace(pos, len, str, subpos, sublen);
s.replace(pos, len, cstr);
s.replace(pos, len, cstr, n);
s.replace(pos, len, n, c);

// Iterator-based replacements
s.replace(i1, i2, str);
s.replace(i1, i2, cstr);
s.replace(i1, i2, cstr, n);
s.replace(i1, i2, n, c);
s.replace(i1, i2, first, last);
s.replace(i1, i2, ilist);

// String view variants (C++17)
s.replace(pos, len, sv);
s.replace(const_iterator i1, i2, sv);
s.replace(pos, len, t, subpos, sublen); // where t converts to string_view
```

**`assign()` overloads (9 total):**
```cpp
s.assign(str);                         // assign string
s.assign(str, subpos, sublen);         // assign substring
s.assign(std::move(str));              // move assign
s.assign(cstr);                        // assign C-string
s.assign(cstr, n);                     // assign first n chars
s.assign(n, c);                        // assign n copies of char
s.assign(first, last);                 // assign iterator range
s.assign(ilist);                       // assign initializer list
s.assign(sv);                          // assign string_view
```

**`compare()` overloads (6 total):**
```cpp
s.compare(str);
s.compare(pos, len, str);
s.compare(pos, len, str, subpos, sublen);
s.compare(cstr);
s.compare(pos, len, cstr);
s.compare(pos, len, cstr, n);
```

**Note:** Most methods that accept `const std::string&` also have overloads accepting:
- `std::string_view` (C++17)
- C-strings (`const char*`)
- Character ranges (iterators)
- Initializer lists

This design provides flexibility while maintaining efficiency through perfect forwarding and move semantics.

---

## 4. `std::string_view` Deep Dive

### 4.1 What It Is

Non-owning view into a character sequence. Size: 16 bytes (pointer + size).

```cpp
struct string_view {
    const char* data_;
    size_t size_;
};
```

### 4.2 Construction

```cpp
std::string s = "hello";
std::string_view sv1 = s;              // from std::string
std::string_view sv2 = "literal";      // from string literal
std::string_view sv3(ptr, len);        // from pointer + length
```

### 4.3 Lifetime Pitfalls

**Danger:** View doesn't extend source lifetime.

```cpp
std::string_view get_view() {
    std::string temp = "temporary";
    return temp;  // UB! temp destroyed, view dangles
}

void process(std::string_view sv) {
    std::string local = std::string(sv);  // safe: copy the data
    // use local, not sv, after this point if source might be destroyed
}
```

**Safe pattern:**
```cpp
void process(std::string_view sv);  // read-only, doesn't store

std::string s = "data";
process(s);  // safe: s outlives call
process("literal");  // safe: literal has static storage
```

### 4.4 Operations

Similar to `std::string` but no modifiers:
- `size()`, `empty()`, `data()`
- `operator[]`, `at()`, `front()`, `back()`
- `substr()` — returns another view (no allocation!)
- `find()`, `rfind()`, `find_first_of()`, etc.
- `remove_prefix(n)`, `remove_suffix(n)` — adjust view in-place

```cpp
std::string_view sv = "hello world";
sv.remove_prefix(6);  // now views "world"
```

### 4.5 Conversion

`string_view` → `string`: explicit copy
```cpp
std::string_view sv = "view";
std::string s(sv);  // or std::string s = std::string(sv);
```

### 4.6 When to Use

**Good:**
- Function parameters for read-only strings (avoids copy)
- Substring operations
- Parsing/tokenization

**Bad:**
- Storage in data structures (lifetime risk)
- Return values (unless source lifetime is clear)
- Passing to C APIs (not NUL-terminated unless you verify)

**NUL-termination:** `string_view` is **not** guaranteed to be NUL-terminated.

```cpp
std::string s = "hello world";
std::string_view sv = s;
sv.remove_suffix(6);  // views "hello"
// sv.data() points into s, but sv.data()[sv.size()] might be ' ', not '\0'
```

---

## 5. Common Operations and Idioms

### 5.1 Concatenation

**Approaches:**

```cpp
// 1. operator+ (simple but can be inefficient)
std::string s = s1 + s2 + s3;  // multiple temporaries

// 2. operator+= (better)
std::string s = s1;
s += s2;
s += s3;

// 3. reserve + append (best for known size)
std::string s;
s.reserve(s1.size() + s2.size() + s3.size());
s += s1;
s += s2;
s += s3;

// 4. ostringstream (flexible, slightly slower)
std::ostringstream oss;
oss << s1 << s2 << s3;
std::string s = oss.str();
```

### 5.2 Formatting

**C++20 `std::format`:**
```cpp
std::string s = std::format("Value: {}, Hex: {:x}", 42, 255);
// "Value: 42, Hex: ff"
```

**Before C++20:**
```cpp
// sprintf (unsafe)
char buf[100];
sprintf(buf, "Value: %d", 42);

// snprintf (safer)
snprintf(buf, sizeof(buf), "Value: %d", 42);

// ostringstream
std::ostringstream oss;
oss << "Value: " << 42;
std::string s = oss.str();

// fmt library (third-party, basis for std::format)
std::string s = fmt::format("Value: {}", 42);
```

### 5.3 Splitting

**Manual:**
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
while (std::getline(iss, token, ',')) {
    // process token
}
```

### 5.4 Trimming

```cpp
std::string_view trim(std::string_view sv) {
    size_t start = sv.find_first_not_of(" \t\n\r");
    if (start == std::string_view::npos) return {};
    size_t end = sv.find_last_not_of(" \t\n\r");
    return sv.substr(start, end - start + 1);
}
```

### 5.5 Searching and Replacing

**Find and replace all:**
```cpp
void replace_all(std::string& str, std::string_view from, std::string_view to) {
    size_t pos = 0;
    while ((pos = str.find(from, pos)) != std::string::npos) {
        str.replace(pos, from.size(), to);
        pos += to.size();
    }
}
```

**Case-insensitive search:**
```cpp
#include <algorithm>
#include <cctype>

bool iequals(std::string_view a, std::string_view b) {
    return std::equal(a.begin(), a.end(), b.begin(), b.end(),
        [](char c1, char c2) {
            return std::tolower(static_cast<unsigned char>(c1)) ==
                   std::tolower(static_cast<unsigned char>(c2));
        });
}
```

### 5.6 Conversions

**String to number:**
```cpp
// stoi family (throws on error)
int n = std::stoi("123");
double d = std::stod("3.14");

// from_chars (C++17, no exceptions, no locale)
int value;
auto [ptr, ec] = std::from_chars(str.data(), str.data() + str.size(), value);
if (ec == std::errc{}) {
    // success
}
```

**Number to string:**
```cpp
// to_string (simple, locale-dependent)
std::string s = std::to_string(42);

// to_chars (C++17, fast, no allocation)
char buf[20];
auto [ptr, ec] = std::to_chars(buf, buf + sizeof(buf), 42);
std::string s(buf, ptr);  // construct from range
```

**Performance:** `to_chars`/`from_chars` are fastest, no locale overhead, no exceptions.

### 5.7 Binary Data

`std::string` can hold arbitrary bytes, including embedded nulls.

```cpp
std::string binary("\x00\x01\x02", 3);  // size = 3, not 1
assert(binary.size() == 3);
assert(binary[0] == '\0');
```

**Caveat:** C-string functions (`strlen`, `strcpy`) will misinterpret embedded nulls.

### 5.8 C API Interop

**Passing to C:**
```cpp
void c_function(const char* str);

std::string s = "data";
c_function(s.c_str());  // safe: NUL-terminated
```

**Receiving from C:**
```cpp
char* c_str = get_c_string();  // C allocates
std::string s = c_str;         // copy into std::string
free(c_str);                   // free C memory
```

**Mutable buffer for C API:**
```cpp
std::string buffer(100, '\0');  // allocate 100 bytes
some_c_function(buffer.data(), buffer.size());
buffer.resize(strlen(buffer.c_str()));  // adjust size
```

---

## 6. Performance and Best Practices

### 6.1 When to Use `string_view`

**Use:**
- Function parameters for read-only strings
- Avoid unnecessary copies
- Substring operations without allocation

**Don't use:**
- Storage (member variables, containers)
- Return values (unless lifetime is guaranteed)
- With C APIs requiring NUL-termination (verify first)

**Example:**
```cpp
// Good
void process(std::string_view sv) {
    if (sv.starts_with("prefix")) { /* ... */ }
}

// Bad
class Widget {
    std::string_view name_;  // DANGER: what if source is destroyed?
public:
    Widget(std::string_view n) : name_(n) {}  // lifetime risk
};
```

**Fix:**
```cpp
class Widget {
    std::string name_;  // own the data
public:
    Widget(std::string_view n) : name_(n) {}  // copy into owned string
};
```

### 6.2 Avoiding Copies

**Move when possible:**
```cpp
std::string create() {
    std::string result = "data";
    return result;  // automatically moved (or elided)
}

std::string s = create();  // no copy
```

**Reserve capacity:**
```cpp
std::string s;
s.reserve(expected_size);
// append operations won't reallocate until capacity exceeded
```

**Use `emplace` instead of `push_back` (for containers of strings):**
```cpp
std::vector<std::string> vec;
vec.emplace_back("text");  // constructs in-place
// vs
vec.push_back("text");     // constructs temporary, then moves
```

**Swap instead of assign:**
```cpp
std::string a = "large data";
std::string b = "other data";
a.swap(b);  // O(1) pointer swap, no copying
```

### 6.3 Hot-Path Optimizations

**Avoid repeated allocation:**
```cpp
// Bad
std::string build_string(const std::vector<std::string>& parts) {
    std::string result;
    for (const auto& p : parts) result += p;  // many reallocations
    return result;
}

// Good
std::string build_string(const std::vector<std::string>& parts) {
    size_t total = 0;
    for (const auto& p : parts) total += p.size();
    std::string result;
    result.reserve(total);
    for (const auto& p : parts) result += p;  // no reallocations
    return result;
}
```

**Prefer `append` over `operator+`:**
```cpp
// Less efficient
std::string s = a + b + c;  // multiple temporary strings

// More efficient
std::string s;
s.reserve(a.size() + b.size() + c.size());
s.append(a).append(b).append(c);
```

**Use `string_view` for substring comparisons:**
```cpp
bool starts_with(std::string_view str, std::string_view prefix) {
    return str.size() >= prefix.size() &&
           str.substr(0, prefix.size()) == prefix;  // no allocation
}
```

### 6.4 SSO Impact

**Benefit:** Short strings (≤15-23 bytes) avoid heap allocation.

**Downside:** Moving short strings still copies bytes.

```cpp
std::string s = "short";
std::string t = std::move(s);  // copies 5 bytes + metadata, doesn't steal
```

**Implication:** For truly cheap moves, use `std::unique_ptr<std::string>` or ensure strings are long enough to avoid SSO.

### 6.5 Allocator Strategies

**Default allocator:** `std::allocator<char>` uses `new`/`delete`.

**Custom allocators:**
- **Pool allocators:** Reduce fragmentation for many small strings
- **Monotonic allocators:** Fast allocation, bulk deallocation
- **PMR allocators (C++17):** Runtime polymorphic allocators

```cpp
#include <memory_resource>

std::pmr::monotonic_buffer_resource pool;
std::pmr::string s("data", &pool);  // allocated from pool
```

**Use case:** High-frequency string creation in a bounded scope.

### 6.6 Threading and Concurrency

**Read-only sharing:**
```cpp
const std::string shared = "data";
// Multiple threads can safely read shared
```

**Exclusive ownership:**
```cpp
// Each thread has its own string
void worker(std::string data) {  // copy passed by value
    data += " modified";
    // safe: no sharing
}
```

**Shared mutable (requires synchronization):**
```cpp
std::mutex mtx;
std::string shared;

void append(const std::string& text) {
    std::lock_guard<std::mutex> lock(mtx);
    shared += text;
}
```

**Copy-on-write (deprecated in C++11):** Old implementations used COW, but it's incompatible with thread-safety requirements of C++11. Modern implementations use deep copy.

---

## 7. Security and Correctness

### 7.1 Buffer Overruns

**std::string is safe:** Automatic bounds checking with `.at()`, safe by default with `.operator[]` (UB on out-of-bounds, but no buffer overrun).

```cpp
std::string s = "test";
char c = s.at(10);  // throws std::out_of_range
```

**C-string danger:**
```cpp
char buf[10];
strcpy(buf, "this is too long");  // buffer overrun!
```

**Safe C-string handling:**
```cpp
strncpy(buf, src, sizeof(buf) - 1);
buf[sizeof(buf) - 1] = '\0';  // ensure termination
```

### 7.2 Embedded Nulls

`std::string` handles embedded nulls correctly; C-string functions don't.

```cpp
std::string s("data\0more", 9);
assert(s.size() == 9);           // correct
assert(strlen(s.c_str()) == 4);  // stops at first null
```

**Safe practice:** Use `.size()` for length, not `strlen()`.

### 7.3 Injection and Validation

**SQL injection prevention:**
```cpp
// BAD: direct concatenation
std::string query = "SELECT * FROM users WHERE name = '" + user_input + "'";

// GOOD: use parameterized queries (not shown, depends on DB library)
```

**Path traversal prevention:**
```cpp
bool is_safe_filename(std::string_view name) {
    return name.find("..") == std::string_view::npos &&
           name.find('/') == std::string_view::npos &&
           name.find('\\') == std::string_view::npos;
}
```

**Input validation:**
```cpp
bool is_valid_email(std::string_view email) {
    // Simple check (use regex for real validation)
    auto at_pos = email.find('@');
    return at_pos != std::string_view::npos &&
           at_pos > 0 &&
           at_pos < email.size() - 1;
}
```

### 7.4 Encoding Issues

**Mixing encodings causes corruption:**
```cpp
std::string utf8 = u8"Hello 世界";     // UTF-8
std::wstring wide = L"Hello 世界";     // UTF-16 or UTF-32

// Naive conversion is WRONG
std::string bad(wide.begin(), wide.end());  // corrupts data
```

**Proper conversion requires iconv, ICU, or platform APIs.**

### 7.5 Locale Mismatches

```cpp
std::string s = "straße";  // German sharp s
std::transform(s.begin(), s.end(), s.begin(), ::toupper);
// Result depends on locale, may not be "STRASSE"
```

**Safe practice:** Use locale-aware functions or libraries (ICU) for case conversion.

### 7.6 Untrusted Input

**Always validate:**
```cpp
std::string process_user_input(std::string_view input) {
    if (input.size() > MAX_SIZE) {
        throw std::invalid_argument("Input too large");
    }
    // sanitize, validate, then process
    return std::string(input);
}
```

**Length limits:** Protect against memory exhaustion.
**Character whitelist:** Reject unexpected characters.
**Escape special characters:** For shell commands, HTML, SQL, etc.

---

## 8. Advanced Topics

### 8.1 Unicode and Internationalization

**Reality:** `std::string` is byte-oriented, not character-oriented.

```cpp
std::string s = u8"Hello 世界";
s.size();  // returns byte count (11), not character count (8)
```

**UTF-8 best practices:**
- Store as `std::string` (or `std::u8string` in C++20)
- Never split on arbitrary byte boundaries
- Use libraries for character-level operations

**Code point iteration (manual):**
```cpp
void iterate_utf8(std::string_view utf8) {
    for (size_t i = 0; i < utf8.size(); ) {
        unsigned char c = utf8[i];
        size_t len = 1;
        
        if (c < 0x80) len = 1;
        else if ((c & 0xE0) == 0xC0) len = 2;
        else if ((c & 0xF0) == 0xE0) len = 3;
        else if ((c & 0xF8) == 0xF0) len = 4;
        
        // Process code point: utf8.substr(i, len)
        i += len;
    }
}
```

**Grapheme cluster iteration:** Requires Unicode libraries (ICU, Boost.Text).

**Normalization:** Unicode text can be in different normal forms (NFC, NFD, NFKC, NFKD). Always normalize when comparing.

```cpp
// Requires ICU
icu::UnicodeString s1 = icu::UnicodeString::fromUTF8("café");  // precomposed
icu::UnicodeString s2 = icu::UnicodeString::fromUTF8("café");  // decomposed
// Direct byte comparison fails; normalize first
```

**When to use ICU or Boost.Text:**
- Character counting
- Case conversion
- Collation (locale-aware sorting)
- Line breaking
- Regular expressions on Unicode text
- Bidirectional text

### 8.2 Locales and Facets

**Locale basics:**
```cpp
std::locale loc("en_US.UTF-8");
std::string s = "text";
std::use_facet<std::ctype<char>>(loc).toupper(s.data(), s.data() + s.size());
```

**Issues:**
- Locale-dependent operations are slow
- Results vary by platform and installed locales
- `std::tolower/toupper` from `<cctype>` use global locale

**Recommendation:** Avoid locale-dependent operations in performance-critical code. Use ICU for serious internationalization.

### 8.3 Regular Expressions

```cpp
#include <regex>

std::string text = "Email: test@example.com";
std::regex pattern(R"([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})");
std::smatch matches;

if (std::regex_search(text, matches, pattern)) {
    std::string email = matches[0];  // "test@example.com"
}
```

**Performance note:** `std::regex` is notoriously slow. Consider alternatives (PCRE2, RE2, hyperscan) for high-performance needs.

**Safety:** Regex can cause catastrophic backtracking. Validate patterns.

### 8.4 String Stream Operations

**`std::ostringstream` (output):**
```cpp
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
```

**`std::stringstream` (bidirectional):**
```cpp
std::stringstream ss;
ss << "data";
std::string s = ss.str();
ss.str("");  // clear contents
ss << "new data";
```

**Performance:** String streams have overhead. For simple conversions, prefer `std::to_string`, `std::to_chars`, or `std::format`.

### 8.5 Implementation Differences

**libstdc++ (GCC):**
- SSO buffer: 15 bytes
- Growth factor: 2x
- ABI stability: yes (frozen since GCC 5)

**libc++ (Clang):**
- SSO buffer: 22 bytes (23 on some platforms)
- Growth factor: 1.5x
- ABI stability: versioned

**MSVC STL:**
- SSO buffer: 15 bytes
- Growth factor: 1.5x
- ABI stability: yes

**Portable code:** Don't rely on exact SSO threshold or growth factor.

**ABI compatibility:** Different standard library implementations are not binary compatible. Link consistently.

### 8.6 Third-Party Libraries

**`{fmt}` / `std::format`:**
- Modern, type-safe formatting
- Faster than iostreams
- `std::format` in C++20 is based on `{fmt}`

```cpp
#include <fmt/core.h>
std::string s = fmt::format("The answer is {}", 42);
```

**Boost.StringAlgo:**
- Comprehensive string algorithms
- Case conversion, trimming, splitting, replacing, finding

```cpp
#include <boost/algorithm/string.hpp>
std::string s = " text ";
boost::trim(s);  // "text"
```

**Boost.Text:**
- Unicode text handling
- Segmentation (grapheme clusters, words, sentences, lines)
- Normalization, case mapping

**ICU (International Components for Unicode):**
- Industry-standard Unicode library
- Comprehensive: collation, formatting, calendars, time zones
- Large dependency (~25 MB), but feature-complete

**PCRE2 / RE2:**
- High-performance regex engines
- RE2: linear time, no backtracking (safe)
- PCRE2: Perl-compatible, powerful

**When to use:**
- Simple projects: stick with standard library
- Complex formatting: `{fmt}` or `std::format`
- String algorithms: Boost.StringAlgo
- Unicode: ICU or Boost.Text
- Regex performance: RE2 or PCRE2

---

## 9. Common Pitfalls and Anti-Patterns

### 9.1 Unnecessary Copies

```cpp
// BAD: returns by value from member
std::string Widget::get_name() const {
    return name_;  // copies every time
}

// GOOD: return by const reference
const std::string& Widget::get_name() const {
    return name_;
}

// BETTER: accept both string and string_view users
std::string_view Widget::get_name() const {
    return name_;  // implicit conversion, no copy
}
```

### 9.2 Temporary Lifetime Issues

```cpp
// DANGER: dangling reference
const char* get_cstr() {
    std::string temp = "data";
    return temp.c_str();  // temp destroyed, pointer invalid
}

// DANGER: dangling string_view
std::string_view get_view() {
    return std::string("temp");  // temporary destroyed immediately
}
```

### 9.3 Inefficient Concatenation

```cpp
// BAD: quadratic complexity
std::string s;
for (int i = 0; i < n; ++i) {
    s = s + std::to_string(i);  // creates temporary each iteration
}

// GOOD: linear complexity
std::string s;
s.reserve(estimated_size);
for (int i = 0; i < n; ++i) {
    s += std::to_string(i);  // amortized O(1) append
}
```

### 9.4 Misusing `string_view`

```cpp
// BAD: storing string_view
class Bad {
    std::string_view view_;  // lifetime hazard
public:
    Bad(const std::string& s) : view_(s) {}
};

std::string temp = "data";
Bad b(temp);
temp.clear();
// b.view_ now dangles

// GOOD: store std::string
class Good {
    std::string data_;
public:
    Good(std::string_view sv) : data_(sv) {}  // copy into owned storage
};
```

### 9.5 Ignoring Return Values

```cpp
std::string s = "hello world";
s.find("foo");  // forgot to check return value
// Should be:
if (s.find("foo") != std::string::npos) {
    // found
}
```

### 9.6 Character Encoding Assumptions

```cpp
// WRONG: assumes 1 byte = 1 character
std::string utf8 = u8"Hello 世界";
for (size_t i = 0; i < utf8.size(); ++i) {
    char c = utf8[i];  // may get mid-character byte
    // process c as character  // WRONG
}

// RIGHT: iterate code points or use library
```

### 9.7 Const Correctness Violations

```cpp
void modify(std::string& s) {
    s += " modified";
}

const std::string cs = "const";
// modify(cs);  // won't compile (correct)

// Casting away const is UB if you modify
std::string& bad = const_cast<std::string&>(cs);
bad += "!";  // UB!
```

---

## 10. Practical Examples

### 10.1 Efficient String Builder

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
    
    void reserve(size_t capacity) {
        buffer_.reserve(capacity);
    }
    
    std::string build() {
        return std::move(buffer_);
    }
};

// Usage
StringBuilder sb;
sb.reserve(100);
sb.append("Hello").append(' ').append("World");
std::string result = sb.build();
```

### 10.2 URL Encoding

```cpp
std::string url_encode(std::string_view sv) {
    std::ostringstream oss;
    oss << std::hex << std::uppercase;
    
    for (unsigned char c : sv) {
        if (std::isalnum(c) || c == '-' || c == '_' || c == '.' || c == '~') {
            oss << c;
        } else {
            oss << '%' << std::setw(2) << std::setfill('0') << int(c);
        }
    }
    return oss.str();
}
```

### 10.3 Case-Insensitive String Comparison

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
map["hello"] = 2;  // overwrites because case-insensitive
```

### 10.4 String Interning (Deduplication)

```cpp
class StringInterner {
    std::unordered_set<std::string> pool_;
public:
    std::string_view intern(std::string_view sv) {
        auto [it, inserted] = pool_.emplace(sv);
        return *it;
    }
};

// Usage: reduce memory for repeated strings
StringInterner interner;
auto s1 = interner.intern("common");
auto s2 = interner.intern("common");
assert(s1.data() == s2.data());  // same underlying storage
```

### 10.5 Token Parsing with Error Handling

```cpp
struct Token {
    enum Type { Number, String, Symbol } type;
    std::string value;
};

std::vector<Token> tokenize(std::string_view input) {
    std::vector<Token> tokens;
    size_t pos = 0;
    
    while (pos < input.size()) {
        // Skip whitespace
        while (pos < input.size() && std::isspace(input[pos])) ++pos;
        if (pos >= input.size()) break;
        
        if (std::isdigit(input[pos])) {
            size_t end = pos;
            while (end < input.size() && std::isdigit(input[end])) ++end;
            tokens.push_back({Token::Number, std::string(input.substr(pos, end - pos))});
            pos = end;
        } else if (std::isalpha(input[pos])) {
            size_t end = pos;
            while (end < input.size() && std::isalnum(input[end])) ++end;
            tokens.push_back({Token::String, std::string(input.substr(pos, end - pos))});
            pos = end;
        } else {
            tokens.push_back({Token::Symbol, std::string(1, input[pos])});
            ++pos;
        }
    }
    return tokens;
}
```

### 10.6 Safe File Path Handling

```cpp
#include <filesystem>

std::string safe_join_path(std::string_view base, std::string_view filename) {
    namespace fs = std::filesystem;
    
    // Validate filename doesn't contain path traversal
    if (filename.find("..") != std::string_view::npos ||
        filename.find('/') != std::string_view::npos ||
        filename.find('\\') != std::string_view::npos) {
        throw std::invalid_argument("Invalid filename");
    }
    
    fs::path result = fs::path(base) / fs::path(filename);
    return result.string();
}
```

---

## 11. Quick Reference Tables

### 11.1 Complexity Summary

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| Construction (empty) | O(1) | SSO buffer initialized |
| Construction (from C-string) | O(n) | Copy n characters |
| Copy construction | O(n) | Deep copy always |
| Move construction | O(1) | O(n) if SSO and short |
| Assignment | O(n) | May reuse capacity |
| `.size()`, `.capacity()` | O(1) | Stored members |
| `.operator[]`, `.at()` | O(1) | Direct access |
| `.push_back()` | Amortized O(1) | May reallocate |
| `.append()`, `+=` | Amortized O(n) | n = size of appended |
| `.insert()` | O(n) | May reallocate + shift |
| `.erase()` | O(n) | Shift elements |
| `.find()` | O(n*m) | Naive search |
| `.substr()` | O(n) | Copy substring |
| `.compare()` | O(n) | Lexicographic |
| `.reserve()` | O(n) | If reallocation needed |
| `.clear()` | O(1) | Doesn't free memory |

### 11.2 When to Use Each String Type

| Type | Use Case | Avoid When |
|------|----------|------------|
| `std::string` | Ownership, storage, modification | Read-only parameters (prefer `string_view`) |
| `std::string_view` | Read-only parameters, substrings | Storage, return values (lifetime risk) |
| `const char*` | C API interop, literals | General use (prefer `string_view`) |
| `std::wstring` | Windows APIs requiring wide strings | Cross-platform Unicode (prefer UTF-8) |
| `std::u8string` | Explicit UTF-8 (C++20) | Before C++20 (use `std::string`) |

### 11.3 Header Dependencies

| Header | Key Types/Functions |
|--------|---------------------|
| `<string>` | `std::string`, `std::wstring`, `std::u8/u16/u32string` |
| `<string_view>` | `std::string_view`, `std::wstring_view` |
| `<cstring>` | `strlen`, `strcpy`, `strcmp`, `memcpy`, etc. |
| `<sstream>` | `std::ostringstream`, `std::istringstream`, `std::stringstream` |
| `<format>` (C++20) | `std::format`, `std::format_to` |
| `<charconv>` (C++17) | `std::to_chars`, `std::from_chars` |
| `<regex>` | `std::regex`, `std::regex_search`, `std::regex_replace` |
| `<locale>` | `std::locale`, facets |
| `<cctype>` | `std::isalpha`, `std::isdigit`, `std::tolower`, etc. |

---

## 12. Final Best Practices Summary

1. **Default to `std::string` for ownership**, `std::string_view` for read-only parameters.

2. **Reserve capacity** when final size is known or estimable.

3. **Use `std::format` (C++20) or `{fmt}` library** for formatting instead of iostreams or `sprintf`.

4. **Prefer `std::to_chars`/`from_chars`** for numeric conversions (fast, no locale, no exceptions).

5. **Never store `string_view`** in data structures or return it unless lifetime is guaranteed.

6. **Use `std::move`** when transferring ownership to avoid copies.

7. **Validate and sanitize external input** to prevent injection, buffer issues, and encoding problems.

8. **For Unicode, use UTF-8** as storage format and ICU/Boost.Text for proper handling.

9. **Avoid C-string functions** (`strlen`, `strcpy`) on `std::string` data—use member functions.

10. **Profile before optimizing**: SSO, move semantics, and modern compilers make most naive code fast enough.

11. **Thread safety**: `const` operations are safe; non-`const` require synchronization.

12. **Test edge cases**: empty strings, embedded nulls, very large strings, UTF-8 boundaries.

---

## 13. Conclusion

This document covered:
- **Foundations**: C-strings, `std::string`, `std::string_view`, Unicode types
- **Internals**: Memory model, SSO, copy/move semantics, iterator invalidation
- **Complete API**: Constructors, modifiers, operations, conversions
- **Idioms**: Concatenation, formatting, splitting, trimming, searching
- **Performance**: Avoiding copies, hot-path optimization, allocator strategies
- **Security**: Validation, encoding, injection prevention
- **Advanced**: Unicode, locales, regex, implementation differences, third-party libraries
- **Pitfalls**: Common mistakes and how to avoid them

Mastering C++ strings requires understanding not just the API, but the underlying mechanisms, performance characteristics, and safety considerations. Use this document as a reference for both learning and daily development.

**Key takeaway:** `std::string` is a powerful, safe, and efficient type when used correctly. Combine it with `std::string_view` for flexibility, understand the cost model, and leverage modern C++ features (`std::format`, `std::to_chars`, move semantics) for optimal code.
