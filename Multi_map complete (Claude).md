# The Complete Guide to `multimap` in C++ STL

## 🧱 BLOCK 1: The Foundation — What Problem Does Multimap Solve?

### First, Let's Understand the Need

Imagine you're building a phone directory system:

```cpp
#include <iostream>
#include <map>
using namespace std;

int main() {
    // Regular map: ONE key → ONE value
    map<string, string> phoneBook;
    
    phoneBook["John"] = "111-1111";
    phoneBook["John"] = "222-2222";  // This REPLACES the previous value!
    
    cout << "John's number: " << phoneBook["John"] << endl;
    // Output: John's number: 222-2222
    // We LOST the first number!
    
    return 0;
}
```

**The Problem:** Regular `map` allows only ONE value per key. But what if John has TWO phone numbers?

### The Solution: `multimap`

```cpp
#include <iostream>
#include <map>
using namespace std;

int main() {
    // Multimap: ONE key → MULTIPLE values
    multimap<string, string> phoneBook;
    
    phoneBook.insert({"John", "111-1111"});
    phoneBook.insert({"John", "222-2222"});  // BOTH are stored!
    
    // Print all of John's numbers
    auto range = phoneBook.equal_range("John");
    for (auto it = range.first; it != range.second; ++it) {
        cout << it->first << ": " << it->second << endl;
    }
    // Output:
    // John: 111-1111
    // John: 222-2222
    
    return 0;
}
```

---

## 🧱 BLOCK 2: What Exactly IS a Multimap?

### Visual Representation

```
    Regular MAP                         MULTIMAP
    ══════════                         ══════════
    
    Key    →  Value                   Key    →  Values
    ─────────────────                 ─────────────────
    "Apple"  →  5                     "Apple"  →  5
    "Banana" →  3                     "Apple"  →  8    ← Same key, different value!
    "Cherry" →  7                     "Apple"  →  2    ← Same key, another value!
                                      "Banana" →  3
                                      "Cherry" →  7
```

### Definition

```
╔═══════════════════════════════════════════════════════════════════════════╗
║  MULTIMAP = An associative container that stores key-value pairs          ║
║             where MULTIPLE ELEMENTS can have the SAME KEY                 ║
║             Elements are automatically SORTED BY KEY                       ║
╚═══════════════════════════════════════════════════════════════════════════╝
```

---

## 🧱 BLOCK 3: Internal Structure — The Red-Black Tree

### How Multimap Stores Data

```
                    Multimap internally uses a SELF-BALANCING 
                         BINARY SEARCH TREE (Red-Black Tree)
                         
                              ┌─────────────┐
                              │   ("Dog", 3)│
                              │    (BLACK)  │
                              └──────┬──────┘
                           ┌─────────┴─────────┐
                     ┌─────┴─────┐       ┌─────┴─────┐
                     │("Cat", 1) │       │("Fish", 2)│
                     │  (RED)    │       │  (RED)    │
                     └─────┬─────┘       └───────────┘
                           │
                     ┌─────┴─────┐
                     │("Cat", 5) │  ← Duplicate key allowed!
                     │  (BLACK)  │
                     └───────────┘
```

### Why This Matters

```cpp
// Because of the tree structure:
// ✅ Elements are always SORTED by key
// ✅ Search, Insert, Delete: O(log n)
// ✅ Duplicate keys are stored in insertion order (for same keys)
```

---

## 🧱 BLOCK 4: Properties & Characteristics

```
┌────────────────────────────────────────────────────────────────────────────┐
│                     MULTIMAP PROPERTIES CHEAT SHEET                        │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  1. DUPLICATE KEYS         → ✅ Allowed (main difference from map)        │
│                                                                            │
│  2. ORDERING               → Sorted by KEY (ascending by default)         │
│                                                                            │
│  3. KEY MODIFICATION       → ❌ Keys are CONST (cannot be modified)       │
│                                                                            │
│  4. VALUE MODIFICATION     → ✅ Values CAN be modified                    │
│                                                                            │
│  5. OPERATOR []            → ❌ NOT available (unlike map)                │
│                                                                            │
│  6. DIRECT ACCESS          → ❌ No random access (no indexing)            │
│                                                                            │
│  7. ITERATOR TYPE          → Bidirectional Iterator                       │
│                                                                            │
│  8. UNDERLYING STRUCTURE   → Red-Black Tree (self-balancing BST)          │
│                                                                            │
│  9. HEADER FILE            → #include <map>                               │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Time Complexities

```
┌──────────────────────┬─────────────────┐
│     Operation        │  Time Complexity│
├──────────────────────┼─────────────────┤
│  Insert              │    O(log n)     │
│  Delete              │    O(log n)     │
│  Search              │    O(log n)     │
│  Access (iterator)   │    O(1)         │
│  Count               │ O(log n + count)│
└──────────────────────┴─────────────────┘
```

---

## 🧱 BLOCK 5: Declaration & Initialization

### The Syntax Template

```cpp
template <
    class Key,                              // Key type
    class T,                                // Value type
    class Compare = std::less<Key>,         // Comparison function
    class Allocator = std::allocator<pair<const Key, T>>  // Memory allocator
> class multimap;
```

### All Ways to Create a Multimap

```cpp
#include <iostream>
#include <map>
#include <string>
using namespace std;

int main() {
    // ═══════════════════════════════════════════════════════
    // METHOD 1: Default Constructor (Empty multimap)
    // ═══════════════════════════════════════════════════════
    multimap<int, string> mm1;
    cout << "Method 1 - Empty multimap created\n";
    cout << "Size: " << mm1.size() << "\n\n";
    
    
    // ═══════════════════════════════════════════════════════
    // METHOD 2: Initializer List
    // ═══════════════════════════════════════════════════════
    multimap<int, string> mm2 = {
        {1, "One"},
        {2, "Two"},
        {1, "Uno"},      // Duplicate key - ALLOWED!
        {2, "Dos"},      // Duplicate key - ALLOWED!
        {3, "Three"}
    };
    
    cout << "Method 2 - Initializer List:\n";
    for (const auto& pair : mm2) {
        cout << "  " << pair.first << " => " << pair.second << "\n";
    }
    cout << "\n";
    
    
    // ═══════════════════════════════════════════════════════
    // METHOD 3: Copy Constructor
    // ═══════════════════════════════════════════════════════
    multimap<int, string> mm3(mm2);  // Copy of mm2
    
    cout << "Method 3 - Copy Constructor:\n";
    for (const auto& [key, value] : mm3) {  // Structured binding (C++17)
        cout << "  " << key << " => " << value << "\n";
    }
    cout << "\n";
    
    
    // ═══════════════════════════════════════════════════════
    // METHOD 4: Move Constructor
    // ═══════════════════════════════════════════════════════
    multimap<int, string> mmTemp = {{10, "Ten"}, {20, "Twenty"}};
    multimap<int, string> mm4(std::move(mmTemp));  // mmTemp is now empty
    
    cout << "Method 4 - Move Constructor:\n";
    cout << "  mm4 size: " << mm4.size() << "\n";
    cout << "  mmTemp size after move: " << mmTemp.size() << "\n\n";
    
    
    // ═══════════════════════════════════════════════════════
    // METHOD 5: Range Constructor (from iterators)
    // ═══════════════════════════════════════════════════════
    multimap<int, string> mm5(mm2.begin(), mm2.end());
    
    cout << "Method 5 - Range Constructor:\n";
    for (const auto& pair : mm5) {
        cout << "  " << pair.first << " => " << pair.second << "\n";
    }
    cout << "\n";
    
    
    // ═══════════════════════════════════════════════════════
    // METHOD 6: Custom Comparator (Descending Order)
    // ═══════════════════════════════════════════════════════
    multimap<int, string, greater<int>> mm6 = {
        {1, "One"},
        {3, "Three"},
        {2, "Two"}
    };
    
    cout << "Method 6 - Custom Comparator (Descending):\n";
    for (const auto& pair : mm6) {
        cout << "  " << pair.first << " => " << pair.second << "\n";
    }
    cout << "\n";
    
    
    // ═══════════════════════════════════════════════════════
    // METHOD 7: Custom Comparator with Lambda (C++11)
    // ═══════════════════════════════════════════════════════
    auto customCompare = [](const string& a, const string& b) {
        return a.length() < b.length();  // Sort by string length
    };
    
    multimap<string, int, decltype(customCompare)> mm7(customCompare);
    mm7.insert({"Hi", 1});
    mm7.insert({"Hello", 2});
    mm7.insert({"Hey", 3});
    
    cout << "Method 7 - Lambda Comparator (by string length):\n";
    for (const auto& pair : mm7) {
        cout << "  \"" << pair.first << "\" (len=" 
             << pair.first.length() << ") => " << pair.second << "\n";
    }
    
    return 0;
}
```

**Output:**
```
Method 1 - Empty multimap created
Size: 0

Method 2 - Initializer List:
  1 => One
  1 => Uno
  2 => Two
  2 => Dos
  3 => Three

Method 3 - Copy Constructor:
  1 => One
  1 => Uno
  2 => Two
  2 => Dos
  3 => Three

Method 4 - Move Constructor:
  mm4 size: 2
  mmTemp size after move: 0

Method 5 - Range Constructor:
  1 => One
  1 => Uno
  2 => Two
  2 => Dos
  3 => Three

Method 6 - Custom Comparator (Descending):
  3 => Three
  2 => Two
  1 => One

Method 7 - Lambda Comparator (by string length):
  "Hi" (len=2) => 1
  "Hey" (len=3) => 3
  "Hello" (len=5) => 2
```

---

## 🧱 BLOCK 6: All Member Functions — Complete Reference

Let me organize ALL multimap functions by category:

### 📦 Category 1: CAPACITY Functions

```cpp
#include <iostream>
#include <map>
using namespace std;

int main() {
    multimap<int, string> mm = {
        {1, "One"}, {1, "Uno"}, {2, "Two"}, {3, "Three"}
    };
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 1. size() - Returns number of elements
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    cout << "size(): " << mm.size() << endl;  // 4
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 2. empty() - Checks if container is empty
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    cout << "empty(): " << (mm.empty() ? "Yes" : "No") << endl;  // No
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 3. max_size() - Returns maximum possible size
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    cout << "max_size(): " << mm.max_size() << endl;  // Very large number
    
    return 0;
}
```

---

### 📦 Category 2: ITERATOR Functions

```cpp
#include <iostream>
#include <map>
using namespace std;

int main() {
    multimap<int, string> mm = {
        {1, "One"}, {2, "Two"}, {3, "Three"}
    };
    
    cout << "=== Iterator Functions Demo ===\n\n";
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 1. begin() / end() - Forward iteration
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    cout << "Forward iteration (begin/end):\n";
    for (auto it = mm.begin(); it != mm.end(); ++it) {
        cout << "  " << it->first << " => " << it->second << "\n";
    }
    cout << "\n";
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 2. rbegin() / rend() - Reverse iteration
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    cout << "Reverse iteration (rbegin/rend):\n";
    for (auto it = mm.rbegin(); it != mm.rend(); ++it) {
        cout << "  " << it->first << " => " << it->second << "\n";
    }
    cout << "\n";
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 3. cbegin() / cend() - Const forward iteration
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    cout << "Const iteration (cbegin/cend):\n";
    for (auto it = mm.cbegin(); it != mm.cend(); ++it) {
        cout << "  " << it->first << " => " << it->second << "\n";
        // it->second = "Modified";  // ❌ ERROR! Cannot modify through const iterator
    }
    cout << "\n";
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 4. crbegin() / crend() - Const reverse iteration
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    cout << "Const reverse iteration (crbegin/crend):\n";
    for (auto it = mm.crbegin(); it != mm.crend(); ++it) {
        cout << "  " << it->first << " => " << it->second << "\n";
    }
    
    return 0;
}
```

**Visual Representation of Iterators:**

```
    multimap: {1,"One"} {2,"Two"} {3,"Three"}
              
              ↓ begin()                    ↓ end()
              ┌─────────┬─────────┬─────────┬───┐
              │ (1,One) │ (2,Two) │(3,Three)│ X │
              └─────────┴─────────┴─────────┴───┘
                                             ↑
                                     (past-the-end)
              
                               rend() ↓                    ↓ rbegin()
              ┌───┬─────────┬─────────┬─────────┐
              │ X │ (1,One) │ (2,Two) │(3,Three)│
              └───┴─────────┴─────────┴─────────┘
                ↑
          (before-the-start)
```

---

### 📦 Category 3: MODIFIER Functions

```cpp
#include <iostream>
#include <map>
#include <string>
using namespace std;

void printMultimap(const multimap<int, string>& mm, const string& label) {
    cout << label << " (size=" << mm.size() << "):\n";
    for (const auto& [key, value] : mm) {
        cout << "  " << key << " => " << value << "\n";
    }
    cout << "\n";
}

int main() {
    multimap<int, string> mm;
    
    cout << "=== MODIFIER Functions Demo ===\n\n";
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 1. insert() - Multiple variations
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    
    // 1a. Insert using pair
    mm.insert(pair<int, string>(1, "One"));
    
    // 1b. Insert using make_pair
    mm.insert(make_pair(2, "Two"));
    
    // 1c. Insert using initializer list (C++11)
    mm.insert({3, "Three"});
    mm.insert({1, "Uno"});     // Duplicate key - OK!
    
    // 1d. Insert with hint (iterator position) - can improve performance
    auto hint = mm.find(2);
    mm.insert(hint, {2, "Dos"});  // Insert near position 'hint'
    
    // 1e. Insert range
    multimap<int, string> mm2 = {{4, "Four"}, {5, "Five"}};
    mm.insert(mm2.begin(), mm2.end());
    
    // 1f. Insert initializer list
    mm.insert({{6, "Six"}, {7, "Seven"}});
    
    printMultimap(mm, "After various inserts");
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 2. emplace() - Construct element in-place (more efficient)
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    mm.emplace(8, "Eight");
    mm.emplace(8, "Ocho");     // Duplicate key - OK!
    
    printMultimap(mm, "After emplace");
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 3. emplace_hint() - Construct in-place with hint
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    auto pos = mm.find(8);
    mm.emplace_hint(pos, 9, "Nine");
    
    printMultimap(mm, "After emplace_hint");
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 4. erase() - Multiple variations
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    
    // 4a. Erase by iterator
    auto it = mm.find(9);
    if (it != mm.end()) {
        mm.erase(it);
    }
    printMultimap(mm, "After erasing key 9 (by iterator)");
    
    // 4b. Erase by key - removes ALL elements with that key
    size_t count = mm.erase(1);  // Removes both (1,"One") and (1,"Uno")
    cout << "Erased " << count << " elements with key 1\n";
    printMultimap(mm, "After erasing all with key 1");
    
    // 4c. Erase by range
    auto start = mm.find(6);
    auto finish = mm.find(8);
    mm.erase(start, finish);  // Erases [start, finish)
    printMultimap(mm, "After erasing range [6, 8)");
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 5. clear() - Remove all elements
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    multimap<int, string> mmClear = {{1, "A"}, {2, "B"}};
    cout << "Before clear: size = " << mmClear.size() << "\n";
    mmClear.clear();
    cout << "After clear: size = " << mmClear.size() << "\n\n";
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 6. swap() - Exchange contents
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    multimap<int, string> mmA = {{1, "A"}, {2, "B"}};
    multimap<int, string> mmB = {{10, "X"}, {20, "Y"}, {30, "Z"}};
    
    cout << "Before swap:\n";
    cout << "  mmA size: " << mmA.size() << "\n";
    cout << "  mmB size: " << mmB.size() << "\n";
    
    mmA.swap(mmB);  // or std::swap(mmA, mmB);
    
    cout << "After swap:\n";
    cout << "  mmA size: " << mmA.size() << "\n";
    cout << "  mmB size: " << mmB.size() << "\n";
    
    printMultimap(mmA, "mmA after swap");
    printMultimap(mmB, "mmB after swap");
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 7. extract() - C++17 - Extract node from container
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    multimap<int, string> mmExtract = {{1, "One"}, {2, "Two"}, {3, "Three"}};
    
    auto node = mmExtract.extract(2);  // Extract node with key 2
    if (!node.empty()) {
        cout << "Extracted: " << node.key() << " => " << node.mapped() << "\n";
        // Can modify the key now!
        node.key() = 20;
        mmExtract.insert(std::move(node));  // Re-insert with new key
    }
    printMultimap(mmExtract, "After extract and re-insert");
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 8. merge() - C++17 - Merge elements from another multimap
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    multimap<int, string> mmSource = {{100, "Hundred"}, {200, "Two Hundred"}};
    multimap<int, string> mmDest = {{1, "One"}};
    
    mmDest.merge(mmSource);  // Move all elements from source to dest
    
    printMultimap(mmDest, "Destination after merge");
    printMultimap(mmSource, "Source after merge (empty)");
    
    return 0;
}
```

---

### 📦 Category 4: LOOKUP Functions (Very Important!)

```cpp
#include <iostream>
#include <map>
#include <string>
using namespace std;

int main() {
    multimap<int, string> mm = {
        {1, "One"},
        {1, "Uno"},
        {1, "Ein"},
        {2, "Two"},
        {2, "Dos"},
        {3, "Three"},
        {5, "Five"}
    };
    
    cout << "=== LOOKUP Functions Demo ===\n\n";
    cout << "Multimap contents:\n";
    for (const auto& [k, v] : mm) {
        cout << "  " << k << " => " << v << "\n";
    }
    cout << "\n";
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 1. find() - Find first element with given key
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    cout << "1. find():\n";
    
    auto it = mm.find(2);
    if (it != mm.end()) {
        cout << "   Found key 2: " << it->first << " => " << it->second << "\n";
    }
    
    auto it2 = mm.find(99);
    if (it2 == mm.end()) {
        cout << "   Key 99 not found\n";
    }
    cout << "\n";
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 2. count() - Count elements with given key
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    cout << "2. count():\n";
    cout << "   Elements with key 1: " << mm.count(1) << "\n";   // 3
    cout << "   Elements with key 2: " << mm.count(2) << "\n";   // 2
    cout << "   Elements with key 3: " << mm.count(3) << "\n";   // 1
    cout << "   Elements with key 4: " << mm.count(4) << "\n";   // 0
    cout << "\n";
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 3. contains() - C++20 - Check if key exists
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    #if __cplusplus >= 202002L  // C++20
    cout << "3. contains() [C++20]:\n";
    cout << "   Contains key 2: " << (mm.contains(2) ? "Yes" : "No") << "\n";
    cout << "   Contains key 99: " << (mm.contains(99) ? "Yes" : "No") << "\n";
    cout << "\n";
    #endif
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 4. lower_bound() - First element NOT LESS than key
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    cout << "4. lower_bound():\n";
    
    // Visual: Keys are [1, 1, 1, 2, 2, 3, 5]
    //                   ^
    //         lower_bound(1) points here (first element >= 1)
    
    auto lb = mm.lower_bound(1);
    cout << "   lower_bound(1): " << lb->first << " => " << lb->second << "\n";
    
    // Keys: [1, 1, 1, 2, 2, 3, 5]
    //                 ^
    //       lower_bound(2) points here (first element >= 2)
    
    lb = mm.lower_bound(2);
    cout << "   lower_bound(2): " << lb->first << " => " << lb->second << "\n";
    
    // Keys: [1, 1, 1, 2, 2, 3, 5]
    //                       ^
    //       lower_bound(4) points here (first element >= 4, which is 5)
    
    lb = mm.lower_bound(4);
    cout << "   lower_bound(4): " << lb->first << " => " << lb->second << "\n";
    cout << "\n";
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 5. upper_bound() - First element GREATER than key
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    cout << "5. upper_bound():\n";
    
    // Visual: Keys are [1, 1, 1, 2, 2, 3, 5]
    //                           ^
    //         upper_bound(1) points here (first element > 1)
    
    auto ub = mm.upper_bound(1);
    cout << "   upper_bound(1): " << ub->first << " => " << ub->second << "\n";
    
    // Keys: [1, 1, 1, 2, 2, 3, 5]
    //                       ^
    //       upper_bound(2) points here (first element > 2)
    
    ub = mm.upper_bound(2);
    cout << "   upper_bound(2): " << ub->first << " => " << ub->second << "\n";
    cout << "\n";
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 6. equal_range() - Get range of elements with given key
    //    Returns pair<lower_bound, upper_bound>
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    cout << "6. equal_range() - THE MOST IMPORTANT FOR MULTIMAP!\n";
    
    // Visual for equal_range(1):
    // Keys: [1, 1, 1, 2, 2, 3, 5]
    //        ^        ^
    //        |        |
    //      first    second (one past last)
    //        └───┬───┘
    //            └── This range contains all elements with key 1
    
    auto range = mm.equal_range(1);
    cout << "   All elements with key 1:\n";
    for (auto it = range.first; it != range.second; ++it) {
        cout << "      " << it->first << " => " << it->second << "\n";
    }
    
    range = mm.equal_range(2);
    cout << "   All elements with key 2:\n";
    for (auto it = range.first; it != range.second; ++it) {
        cout << "      " << it->first << " => " << it->second << "\n";
    }
    
    // Key that doesn't exist
    range = mm.equal_range(99);
    cout << "   Elements with key 99: ";
    if (range.first == range.second) {
        cout << "None found\n";
    }
    
    return 0;
}
```

**Visual Explanation of Bounds:**

```
    Multimap Keys:    1    1    1    2    2    3    5
                      ↑                   ↑
                      │                   │
    lower_bound(1) ───┘                   │
                                          │
    upper_bound(1) ───────────────────────┘
    
    
    equal_range(1) returns: {lower_bound(1), upper_bound(1)}
                            
    The range [lower_bound, upper_bound) contains ALL elements with key 1
```

---

### 📦 Category 5: OBSERVER Functions

```cpp
#include <iostream>
#include <map>
#include <string>
using namespace std;

int main() {
    multimap<int, string> mm;
    
    cout << "=== OBSERVER Functions Demo ===\n\n";
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 1. key_comp() - Returns the key comparison function
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    auto keyComp = mm.key_comp();
    
    cout << "1. key_comp():\n";
    cout << "   Is 1 < 2? " << (keyComp(1, 2) ? "Yes" : "No") << "\n";  // Yes
    cout << "   Is 2 < 1? " << (keyComp(2, 1) ? "Yes" : "No") << "\n";  // No
    cout << "\n";
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 2. value_comp() - Returns the value comparison function
    //    (compares pair<Key, Value> by key)
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    auto valueComp = mm.value_comp();
    
    pair<int, string> p1 = {1, "One"};
    pair<int, string> p2 = {2, "Two"};
    
    cout << "2. value_comp():\n";
    cout << "   Is (1,'One') < (2,'Two')? " 
         << (valueComp(p1, p2) ? "Yes" : "No") << "\n";  // Yes
    cout << "\n";
    
    
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 3. get_allocator() - Returns the allocator
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    auto alloc = mm.get_allocator();
    cout << "3. get_allocator(): Allocator obtained (used for memory management)\n";
    
    return 0;
}
```

---

## 🧱 BLOCK 7: Complete Function Reference Table

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MULTIMAP COMPLETE FUNCTION REFERENCE                     │
├──────────────────────┬───────────────────────────────────────┬──────────────┤
│     Function         │            Description                │  Complexity  │
├──────────────────────┼───────────────────────────────────────┼──────────────┤
│                      │         ** CAPACITY **                │              │
├──────────────────────┼───────────────────────────────────────┼──────────────┤
│ size()               │ Number of elements                    │    O(1)      │
│ empty()              │ Check if empty                        │    O(1)      │
│ max_size()           │ Maximum possible size                 │    O(1)      │
├──────────────────────┼───────────────────────────────────────┼──────────────┤
│                      │         ** ITERATORS **               │              │
├──────────────────────┼───────────────────────────────────────┼──────────────┤
│ begin() / end()      │ Forward iterators                     │    O(1)      │
│ rbegin() / rend()    │ Reverse iterators                     │    O(1)      │
│ cbegin() / cend()    │ Const forward iterators               │    O(1)      │
│ crbegin() / crend()  │ Const reverse iterators               │    O(1)      │
├──────────────────────┼───────────────────────────────────────┼──────────────┤
│                      │         ** MODIFIERS **               │              │
├──────────────────────┼───────────────────────────────────────┼──────────────┤
│ insert(val)          │ Insert element                        │  O(log n)    │
│ insert(hint, val)    │ Insert with hint                      │  O(log n)*   │
│ insert(first, last)  │ Insert range                          │O(m log(n+m))│
│ emplace(args...)     │ Construct and insert                  │  O(log n)    │
│ emplace_hint(h,args) │ Construct with hint                   │  O(log n)*   │
│ erase(iterator)      │ Erase at position                     │  O(1)**      │
│ erase(key)           │ Erase all with key                    │O(log n + c) │
│ erase(first, last)   │ Erase range                           │  O(dist)     │
│ clear()              │ Remove all elements                   │    O(n)      │
│ swap(other)          │ Exchange contents                     │    O(1)      │
│ extract(iter)        │ Extract node (C++17)                  │  O(log n)    │
│ merge(source)        │ Merge from source (C++17)             │O(m log(n+m))│
├──────────────────────┼───────────────────────────────────────┼──────────────┤
│                      │         ** LOOKUP **                  │              │
├──────────────────────┼───────────────────────────────────────┼──────────────┤
│ find(key)            │ Find element with key                 │  O(log n)    │
│ count(key)           │ Count elements with key               │O(log n + c) │
│ contains(key)        │ Check if key exists (C++20)           │  O(log n)    │
│ lower_bound(key)     │ First element >= key                  │  O(log n)    │
│ upper_bound(key)     │ First element > key                   │  O(log n)    │
│ equal_range(key)     │ Range of elements with key            │  O(log n)    │
├──────────────────────┼───────────────────────────────────────┼──────────────┤
│                      │         ** OBSERVERS **               │              │
├──────────────────────┼───────────────────────────────────────┼──────────────┤
│ key_comp()           │ Returns key comparison function       │    O(1)      │
│ value_comp()         │ Returns value comparison function     │    O(1)      │
│ get_allocator()      │ Returns allocator                     │    O(1)      │
└──────────────────────┴───────────────────────────────────────┴──────────────┘

* Amortized O(1) if hint is correct
** Amortized constant
c = count of elements with that key
n = size of container
m = number of elements being inserted
```

---

## 🧱 BLOCK 8: Practical Examples

### Example 1: Student Grade Management System

```cpp
#include <iostream>
#include <map>
#include <string>
#include <iomanip>
using namespace std;

class GradeBook {
private:
    // Student name -> Grade (one student can have multiple grades)
    multimap<string, double> grades;

public:
    // Add a grade for a student
    void addGrade(const string& student, double grade) {
        grades.insert({student, grade});
        cout << "Added grade " << grade << " for " << student << "\n";
    }
    
    // Get all grades for a student
    void getStudentGrades(const string& student) const {
        auto range = grades.equal_range(student);
        
        if (range.first == range.second) {
            cout << student << " has no grades.\n";
            return;
        }
        
        cout << "\n" << student << "'s grades: ";
        for (auto it = range.first; it != range.second; ++it) {
            cout << it->second << " ";
        }
        cout << "\n";
    }
    
    // Calculate average grade for a student
    double getAverage(const string& student) const {
        auto range = grades.equal_range(student);
        
        double sum = 0;
        int count = 0;
        
        for (auto it = range.first; it != range.second; ++it) {
            sum += it->second;
            count++;
        }
        
        return count > 0 ? sum / count : 0;
    }
    
    // Get number of grades for a student
    size_t getGradeCount(const string& student) const {
        return grades.count(student);
    }
    
    // Remove all grades for a student
    void removeStudent(const string& student) {
        size_t removed = grades.erase(student);
        cout << "Removed " << removed << " grades for " << student << "\n";
    }
    
    // Print all grades
    void printAll() const {
        cout << "\n=== All Grades ===\n";
        string currentStudent = "";
        
        for (const auto& [student, grade] : grades) {
            if (student != currentStudent) {
                if (!currentStudent.empty()) cout << "\n";
                currentStudent = student;
                cout << student << ": ";
            }
            cout << grade << " ";
        }
        cout << "\n";
    }
    
    // Get students with grade above threshold
    void getTopPerformers(double threshold) const {
        cout << "\nStudents with grades above " << threshold << ":\n";
        
        for (const auto& [student, grade] : grades) {
            if (grade >= threshold) {
                cout << "  " << student << ": " << grade << "\n";
            }
        }
    }
};

int main() {
    GradeBook gb;
    
    // Add grades
    gb.addGrade("Alice", 85.5);
    gb.addGrade("Bob", 72.0);
    gb.addGrade("Alice", 90.0);
    gb.addGrade("Charlie", 88.5);
    gb.addGrade("Alice", 92.5);
    gb.addGrade("Bob", 78.5);
    
    gb.printAll();
    
    // Get specific student's grades
    gb.getStudentGrades("Alice");
    
    // Calculate averages
    cout << "\nAverages:\n";
    cout << "  Alice: " << fixed << setprecision(2) << gb.getAverage("Alice") << "\n";
    cout << "  Bob: " << gb.getAverage("Bob") << "\n";
    cout << "  Charlie: " << gb.getAverage("Charlie") << "\n";
    
    // Grade counts
    cout << "\nGrade counts:\n";
    cout << "  Alice: " << gb.getGradeCount("Alice") << " grades\n";
    cout << "  Bob: " << gb.getGradeCount("Bob") << " grades\n";
    
    // Top performers
    gb.getTopPerformers(85.0);
    
    // Remove a student
    gb.removeStudent("Bob");
    gb.printAll();
    
    return 0;
}
```

---

### Example 2: Event Scheduler (Multiple Events at Same Time)

```cpp
#include <iostream>
#include <map>
#include <string>
#include <iomanip>
using namespace std;

struct Time {
    int hour;
    int minute;
    
    bool operator<(const Time& other) const {
        if (hour != other.hour) return hour < other.hour;
        return minute < other.minute;
    }
    
    friend ostream& operator<<(ostream& os, const Time& t) {
        os << setfill('0') << setw(2) << t.hour << ":" 
           << setfill('0') << setw(2) << t.minute;
        return os;
    }
};

class EventScheduler {
private:
    multimap<Time, string> events;

public:
    void addEvent(Time time, const string& event) {
        events.insert({time, event});
        cout << "Scheduled: '" << event << "' at " << time << "\n";
    }
    
    void printSchedule() const {
        cout << "\n===== Daily Schedule =====\n";
        Time currentTime = {-1, -1};
        
        for (const auto& [time, event] : events) {
            if (time.hour != currentTime.hour || time.minute != currentTime.minute) {
                if (currentTime.hour != -1) cout << "\n";
                currentTime = time;
                cout << time << ":\n";
            }
            cout << "    • " << event << "\n";
        }
        cout << "==========================\n\n";
    }
    
    void getEventsAt(Time time) const {
        auto range = events.equal_range(time);
        
        cout << "Events at " << time << ":\n";
        if (range.first == range.second) {
            cout << "  No events scheduled\n";
            return;
        }
        
        for (auto it = range.first; it != range.second; ++it) {
            cout << "  • " << it->second << "\n";
        }
    }
    
    // Get events in time range
    void getEventsBetween(Time start, Time end) const {
        cout << "\nEvents between " << start << " and " << end << ":\n";
        
        auto itStart = events.lower_bound(start);
        auto itEnd = events.upper_bound(end);
        
        for (auto it = itStart; it != itEnd; ++it) {
            cout << "  " << it->first << " - " << it->second << "\n";
        }
    }
    
    void cancelEvent(Time time, const string& event) {
        auto range = events.equal_range(time);
        
        for (auto it = range.first; it != range.second; ++it) {
            if (it->second == event) {
                events.erase(it);
                cout << "Cancelled: '" << event << "' at " << time << "\n";
                return;
            }
        }
        cout << "Event not found!\n";
    }
    
    size_t countEventsAt(Time time) const {
        return events.count(time);
    }
};

int main() {
    EventScheduler scheduler;
    
    // Add events (note: multiple events at same time!)
    scheduler.addEvent({9, 0}, "Team Standup");
    scheduler.addEvent({9, 0}, "Check emails");
    scheduler.addEvent({10, 30}, "Client call");
    scheduler.addEvent({12, 0}, "Lunch with Bob");
    scheduler.addEvent({12, 0}, "Review documents");
    scheduler.addEvent({14, 0}, "Project meeting");
    scheduler.addEvent({14, 0}, "Code review");
    scheduler.addEvent({14, 0}, "Sprint planning");
    scheduler.addEvent({16, 30}, "Wrap up");
    
    scheduler.printSchedule();
    
    // Query specific time
    scheduler.getEventsAt({14, 0});
    cout << "Number of events at 14:00: " << scheduler.countEventsAt({14, 0}) << "\n\n";
    
    // Get events in range
    scheduler.getEventsBetween({10, 0}, {14, 30});
    
    // Cancel an event
    scheduler.cancelEvent({14, 0}, "Code review");
    scheduler.printSchedule();
    
    return 0;
}
```

---

### Example 3: Word Frequency Counter with Positions

```cpp
#include <iostream>
#include <map>
#include <string>
#include <sstream>
#include <algorithm>
using namespace std;

class TextAnalyzer {
private:
    // word -> position(s) where it appears
    multimap<string, int> wordPositions;
    
    string toLower(string s) {
        transform(s.begin(), s.end(), s.begin(), ::tolower);
        // Remove non-alphanumeric
        s.erase(remove_if(s.begin(), s.end(), 
            [](char c) { return !isalnum(c); }), s.end());
        return s;
    }

public:
    void analyze(const string& text) {
        wordPositions.clear();
        
        istringstream iss(text);
        string word;
        int position = 0;
        
        while (iss >> word) {
            word = toLower(word);
            if (!word.empty()) {
                wordPositions.insert({word, position});
            }
            position++;
        }
        
        cout << "Analyzed " << position << " words.\n";
    }
    
    void findWord(const string& word) const {
        string lowerWord = word;
        transform(lowerWord.begin(), lowerWord.end(), lowerWord.begin(), ::tolower);
        
        auto range = wordPositions.equal_range(lowerWord);
        
        size_t count = distance(range.first, range.second);
        cout << "\n'" << word << "' appears " << count << " time(s)";
        
        if (count > 0) {
            cout << " at position(s): ";
            for (auto it = range.first; it != range.second; ++it) {
                cout << it->second << " ";
            }
        }
        cout << "\n";
    }
    
    void printWordFrequencies() const {
        cout << "\n=== Word Frequencies ===\n";
        
        string currentWord = "";
        int count = 0;
        
        for (auto it = wordPositions.begin(); it != wordPositions.end(); ++it) {
            if (it->first != currentWord) {
                if (!currentWord.empty()) {
                    cout << setw(15) << left << currentWord << ": " << count << "\n";
                }
                currentWord = it->first;
                count = 1;
            } else {
                count++;
            }
        }
        
        // Print last word
        if (!currentWord.empty()) {
            cout << setw(15) << left << currentWord << ": " << count << "\n";
        }
    }
    
    void getMostFrequent(int topN) const {
        // First, count frequencies
        map<string, int> freqMap;
        for (const auto& [word, pos] : wordPositions) {
            freqMap[word]++;
        }
        
        // Convert to vector and sort by frequency
        vector<pair<string, int>> freqVec(freqMap.begin(), freqMap.end());
        sort(freqVec.begin(), freqVec.end(),
            [](const auto& a, const auto& b) {
                return a.second > b.second;  // Descending order
            });
        
        cout << "\n=== Top " << topN << " Most Frequent Words ===\n";
        for (int i = 0; i < min(topN, (int)freqVec.size()); i++) {
            cout << i+1 << ". " << freqVec[i].first 
                 << " (" << freqVec[i].second << " occurrences)\n";
        }
    }
};

int main() {
    TextAnalyzer analyzer;
    
    string text = "The quick brown fox jumps over the lazy dog. "
                  "The dog was not amused. The fox ran away quickly. "
                  "The quick fox was very quick indeed.";
    
    cout << "Text: \"" << text << "\"\n\n";
    
    analyzer.analyze(text);
    
    analyzer.findWord("the");
    analyzer.findWord("quick");
    analyzer.findWord("fox");
    analyzer.findWord("cat");  // Not found
    
    analyzer.printWordFrequencies();
    
    analyzer.getMostFrequent(5);
    
    return 0;
}
```

---

## 🧱 BLOCK 9: Map vs Multimap Comparison

```cpp
#include <iostream>
#include <map>
using namespace std;

int main() {
    cout << "=== MAP vs MULTIMAP ===\n\n";
    
    // ─────────────────────────────────────────────────────
    // DIFFERENCE 1: Duplicate Keys
    // ─────────────────────────────────────────────────────
    cout << "1. Duplicate Keys:\n";
    
    map<int, string> m;
    m.insert({1, "First"});
    m.insert({1, "Second"});  // ❌ Ignored! Key already exists
    cout << "   map[1] = " << m[1] << " (only one value)\n";
    
    multimap<int, string> mm;
    mm.insert({1, "First"});
    mm.insert({1, "Second"});  // ✅ Both stored!
    cout << "   multimap has " << mm.count(1) << " elements with key 1\n\n";
    
    
    // ─────────────────────────────────────────────────────
    // DIFFERENCE 2: operator[]
    // ─────────────────────────────────────────────────────
    cout << "2. Operator []:\n";
    cout << "   map: m[5] = " << m[5] << " (creates empty entry if not exist)\n";
    // mm[5];  // ❌ ERROR! multimap doesn't have operator[]
    cout << "   multimap: NO operator[] available\n\n";
    
    
    // ─────────────────────────────────────────────────────
    // DIFFERENCE 3: Insert Return Type
    // ─────────────────────────────────────────────────────
    cout << "3. Insert Return Type:\n";
    
    auto mapResult = m.insert({10, "Ten"});
    cout << "   map::insert returns pair<iterator, bool>\n";
    cout << "   Inserted: " << (mapResult.second ? "Yes" : "No") << "\n";
    
    auto mmapResult = mm.insert({10, "Ten"});
    cout << "   multimap::insert returns iterator (always succeeds)\n\n";
    
    
    // ─────────────────────────────────────────────────────
    // DIFFERENCE 4: Finding Elements
    // ─────────────────────────────────────────────────────
    cout << "4. Finding Elements:\n";
    
    // In map - at most one element per key
    auto itMap = m.find(1);
    if (itMap != m.end()) {
        cout << "   map: Found key 1 -> " << itMap->second << "\n";
    }
    
    // In multimap - might be multiple elements per key
    auto range = mm.equal_range(1);
    cout << "   multimap: Found key 1 ->";
    for (auto it = range.first; it != range.second; ++it) {
        cout << " " << it->second;
    }
    cout << "\n\n";
    
    
    // ─────────────────────────────────────────────────────
    // Summary Table
    // ─────────────────────────────────────────────────────
    cout << "┌─────────────────────┬──────────────┬────────────────┐\n";
    cout << "│     Feature         │     map      │   multimap     │\n";
    cout << "├─────────────────────┼──────────────┼────────────────┤\n";
    cout << "│ Duplicate keys      │     ❌       │      ✅        │\n";
    cout << "│ operator[]          │     ✅       │      ❌        │\n";
    cout << "│ at()                │     ✅       │      ❌        │\n";
    cout << "│ count() returns     │    0 or 1    │    0, 1, ...   │\n";
    cout << "│ insert returns      │ pair<it,bool>│    iterator    │\n";
    cout << "│ equal_range useful  │   Rarely     │   Very often   │\n";
    cout << "└─────────────────────┴──────────────┴────────────────┘\n";
    
    return 0;
}
```

---

## 🧱 BLOCK 10: Custom Comparators

```cpp
#include <iostream>
#include <map>
#include <string>
#include <functional>
using namespace std;

// Custom comparator as a struct
struct CaseInsensitiveCompare {
    bool operator()(const string& a, const string& b) const {
        string lowerA = a, lowerB = b;
        transform(lowerA.begin(), lowerA.end(), lowerA.begin(), ::tolower);
        transform(lowerB.begin(), lowerB.end(), lowerB.begin(), ::tolower);
        return lowerA < lowerB;
    }
};

// Comparator for reverse order
struct ReverseCompare {
    bool operator()(int a, int b) const {
        return a > b;  // Greater than for descending order
    }
};

int main() {
    cout << "=== Custom Comparators ===\n\n";
    
    // ─────────────────────────────────────────────────────
    // Example 1: Descending order (using greater<>)
    // ─────────────────────────────────────────────────────
    multimap<int, string, greater<int>> mmDesc = {
        {1, "One"}, {5, "Five"}, {3, "Three"}, {2, "Two"}
    };
    
    cout << "1. Descending order (greater<int>):\n";
    for (const auto& [k, v] : mmDesc) {
        cout << "   " << k << " => " << v << "\n";
    }
    cout << "\n";
    
    
    // ─────────────────────────────────────────────────────
    // Example 2: Custom struct comparator
    // ─────────────────────────────────────────────────────
    multimap<int, string, ReverseCompare> mmReverse = {
        {1, "One"}, {5, "Five"}, {3, "Three"}
    };
    
    cout << "2. Custom struct comparator (ReverseCompare):\n";
    for (const auto& [k, v] : mmReverse) {
        cout << "   " << k << " => " << v << "\n";
    }
    cout << "\n";
    
    
    // ─────────────────────────────────────────────────────
    // Example 3: Case-insensitive string comparison
    // ─────────────────────────────────────────────────────
    multimap<string, int, CaseInsensitiveCompare> mmCaseInsensitive;
    mmCaseInsensitive.insert({"Apple", 1});
    mmCaseInsensitive.insert({"apple", 2});   // Considered same key!
    mmCaseInsensitive.insert({"APPLE", 3});   // Considered same key!
    mmCaseInsensitive.insert({"Banana", 4});
    mmCaseInsensitive.insert({"BANANA", 5});
    
    cout << "3. Case-insensitive comparator:\n";
    for (const auto& [k, v] : mmCaseInsensitive) {
        cout << "   " << k << " => " << v << "\n";
    }
    cout << "\n";
    
    // Search is also case-insensitive
    cout << "   Count of 'APPLE': " << mmCaseInsensitive.count("APPLE") << "\n";
    cout << "   Count of 'apple': " << mmCaseInsensitive.count("apple") << "\n";
    cout << "\n";
    
    
    // ─────────────────────────────────────────────────────
    // Example 4: Lambda comparator
    // ─────────────────────────────────────────────────────
    auto lengthCompare = [](const string& a, const string& b) {
        if (a.length() != b.length()) {
            return a.length() < b.length();
        }
        return a < b;  // If same length, compare alphabetically
    };
    
    multimap<string, int, decltype(lengthCompare)> mmByLength(lengthCompare);
    mmByLength.insert({"cat", 1});
    mmByLength.insert({"elephant", 2});
    mmByLength.insert({"dog", 3});
    mmByLength.insert({"ant", 4});
    mmByLength.insert({"tiger", 5});
    
    cout << "4. Lambda comparator (by string length):\n";
    for (const auto& [k, v] : mmByLength) {
        cout << "   " << k << " (len=" << k.length() << ") => " << v << "\n";
    }
    cout << "\n";
    
    
    // ─────────────────────────────────────────────────────
    // Example 5: std::function comparator
    // ─────────────────────────────────────────────────────
    function<bool(int, int)> absCompare = [](int a, int b) {
        return abs(a) < abs(b);
    };
    
    multimap<int, string, function<bool(int, int)>> mmAbs(absCompare);
    mmAbs.insert({-5, "Neg Five"});
    mmAbs.insert({3, "Three"});
    mmAbs.insert({-1, "Neg One"});
    mmAbs.insert({4, "Four"});
    mmAbs.insert({-4, "Neg Four"});
    
    cout << "5. std::function comparator (by absolute value):\n";
    for (const auto& [k, v] : mmAbs) {
        cout << "   " << k << " (|" << k << "|=" << abs(k) << ") => " << v << "\n";
    }
    
    return 0;
}
```

---

## 🧱 BLOCK 11: When to Use Multimap

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    WHEN TO USE MULTIMAP                                    │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ✅ USE MULTIMAP WHEN:                                                     │
│     • You need to store MULTIPLE values per key                            │
│     • You need elements SORTED by key                                      │
│     • You need efficient lookup by key O(log n)                            │
│     • You need to find ALL values for a given key                          │
│     • Order of insertion matters for same keys                             │
│                                                                            │
│  ❌ DON'T USE MULTIMAP WHEN:                                               │
│     • You need only ONE value per key → Use map                            │
│     • You need O(1) lookup → Use unordered_multimap                        │
│     • You don't care about ordering → Use unordered_multimap               │
│     • You need index-based access → Use vector                             │
│                                                                            │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  COMMON USE CASES:                                                         │
│     📞 Phone directories (one person, multiple numbers)                    │
│     📚 Bookstore inventory (author → multiple books)                       │
│     📅 Event scheduling (time → multiple events)                           │
│     🏫 Student grades (student → multiple grades)                          │
│     🏷️ Product categories (category → multiple products)                   │
│     📊 Logging systems (timestamp → multiple log entries)                  │
│     🔤 Dictionary (word → multiple definitions)                            │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 🧱 BLOCK 12: Container Comparison Chart

```
┌──────────────────┬──────────────┬────────────────┬──────────────────────────┐
│   Container      │  Duplicates  │    Sorted?     │   Best For               │
├──────────────────┼──────────────┼────────────────┼──────────────────────────┤
│ map              │     ❌       │      ✅        │ Unique key-value pairs   │
│ multimap         │     ✅       │      ✅        │ Multiple values per key  │
│ unordered_map    │     ❌       │      ❌        │ Fast lookup, unique keys │
│ unordered_       │     ✅       │      ❌        │ Fast lookup, multi-vals  │
│   multimap       │              │                │                          │
├──────────────────┼──────────────┼────────────────┼──────────────────────────┤
│ set              │     ❌       │      ✅        │ Sorted unique values     │
│ multiset         │     ✅       │      ✅        │ Sorted with duplicates   │
│ unordered_set    │     ❌       │      ❌        │ Fast lookup, unique      │
│ unordered_       │     ✅       │      ❌        │ Fast lookup, duplicates  │
│   multiset       │              │                │                          │
└──────────────────┴──────────────┴────────────────┴──────────────────────────┘

Performance Comparison:
┌──────────────────┬─────────────┬─────────────┬─────────────┐
│   Container      │   Insert    │   Search    │   Memory    │
├──────────────────┼─────────────┼─────────────┼─────────────┤
│ multimap         │  O(log n)   │  O(log n)   │   Higher    │
│ unordered_       │    O(1)*    │    O(1)*    │   Lower     │
│   multimap       │             │             │             │
└──────────────────┴─────────────┴─────────────┴─────────────┘
* Average case; worst case is O(n)
```

---

## 🎯 Final Summary

```
╔═══════════════════════════════════════════════════════════════════════════╗
║                        MULTIMAP CHEAT SHEET                               ║
╠═══════════════════════════════════════════════════════════════════════════╣
║                                                                           ║
║  #include <map>                                                           ║
║                                                                           ║
║  DECLARATION:                                                             ║
║    multimap<KeyType, ValueType> mm;                                       ║
║    multimap<KeyType, ValueType, Comparator> mm;                           ║
║                                                                           ║
║  KEY OPERATIONS:                                                          ║
║    mm.insert({key, value});     // Always succeeds                        ║
║    mm.emplace(key, value);      // More efficient                         ║
║    mm.erase(key);               // Removes ALL with that key              ║
║    mm.find(key);                // Returns iterator to first match        ║
║    mm.count(key);               // How many elements have this key        ║
║    mm.equal_range(key);         // Get ALL elements with key              ║
║                                                                           ║
║  ITERATING ALL VALUES FOR A KEY:                                          ║
║    auto [begin, end] = mm.equal_range(key);                               ║
║    for (auto it = begin; it != end; ++it) {                               ║
║        // it->first is key, it->second is value                           ║
║    }                                                                      ║
║                                                                           ║
║  REMEMBER:                                                                ║
║    ❌ NO operator[]                                                       ║
║    ❌ NO at() function                                                    ║
║    ✅ Keys are CONST (cannot modify)                                      ║
║    ✅ Values CAN be modified                                              ║
║    ✅ Always sorted by key                                                ║
║                                                                           ║
╚═══════════════════════════════════════════════════════════════════════════╝
```

---

You now have a complete understanding of `multimap` in C++ STL! From basic concepts to advanced usage patterns, you're ready to use this container effectively in your programs. 

Would you like me to elaborate on any specific section or provide more examples for particular use cases?
