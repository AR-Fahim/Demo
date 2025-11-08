# Python Lists & Tuples: Complete Guide

## 1. Lists vs Tuples: Key Differences

| Feature | List | Tuple |
|---------|------|-------|
| Syntax | `[1, 2, 3]` | `(1, 2, 3)` |
| Mutability | Mutable | Immutable |
| Performance | Slower | Faster |
| Memory | More | Less |
| Methods | Many | Few (2) |
| Use case | Dynamic data | Fixed data, dict keys |

## 2. List Creation

```python
# Basic
lst = [1, 2, 3]
lst = list()              # Empty list
lst = []                  # Empty list (preferred)

# From iterable
lst = list("abc")         # ['a', 'b', 'c']
lst = list(range(5))      # [0, 1, 2, 3, 4]
lst = list((1, 2, 3))     # From tuple

# List comprehension
lst = [x*2 for x in range(5)]           # [0, 2, 4, 6, 8]
lst = [x for x in range(10) if x % 2]  # [1, 3, 5, 7, 9]
lst = [x if x > 0 else 0 for x in [-1, 2, -3]]  # [0, 2, 0]

# Nested comprehension
matrix = [[i+j for j in range(3)] for i in range(3)]
# [[0,1,2], [1,2,3], [2,3,4]]

# Mixed types (allowed but avoid)
lst = [1, "two", 3.0, [4, 5]]

# Pre-allocated list
lst = [0] * 5             # [0, 0, 0, 0, 0]
lst = [None] * 3          # [None, None, None]
```

## 3. Tuple Creation

```python
# Basic
tup = (1, 2, 3)
tup = tuple()             # Empty tuple
tup = ()                  # Empty tuple (preferred)

# Single element (comma required!)
tup = (1,)                # Tuple with one element
not_tuple = (1)           # This is int, not tuple!

# Without parentheses (tuple packing)
tup = 1, 2, 3             # (1, 2, 3)
a, b, c = tup             # Tuple unpacking

# From iterable
tup = tuple([1, 2, 3])    # From list
tup = tuple("abc")        # ('a', 'b', 'c')

# Generator expression (creates tuple)
tup = tuple(x*2 for x in range(5))  # (0, 2, 4, 6, 8)
```

## 4. Indexing & Slicing (Both List & Tuple)

```python
lst = [0, 1, 2, 3, 4, 5]

# Indexing
lst[0]                    # 0 (first)
lst[-1]                   # 5 (last)
lst[-2]                   # 4 (second last)
# lst[100]                # IndexError

# Slicing [start:stop:step]
lst[1:4]                  # [1, 2, 3]
lst[:3]                   # [0, 1, 2] (first 3)
lst[3:]                   # [3, 4, 5] (from index 3)
lst[::2]                  # [0, 2, 4] (every 2nd)
lst[::-1]                 # [5, 4, 3, 2, 1, 0] (reverse)
lst[1::2]                 # [1, 3, 5] (odd indices)
lst[-3:]                  # [3, 4, 5] (last 3)
lst[:-2]                  # [0, 1, 2, 3] (all except last 2)

# Edge cases
lst[100:]                 # [] (no error)
lst[5:2]                  # [] (invalid range)
lst[:100]                 # [0, 1, 2, 3, 4, 5] (clips to end)

# Negative step
lst[4:1:-1]               # [4, 3, 2] (reverse slice)
lst[::-1]                 # [5, 4, 3, 2, 1, 0] (full reverse)
```

## 5. List Methods (11 Total)

### **Adding Elements**

```python
lst = [1, 2, 3]

# append() - add single element at end
lst.append(4)             # [1, 2, 3, 4]
lst.append([5, 6])        # [1, 2, 3, 4, [5, 6]] (nested)

# extend() - add multiple elements
lst = [1, 2, 3]
lst.extend([4, 5])        # [1, 2, 3, 4, 5]
lst.extend("ab")          # [1, 2, 3, 4, 5, 'a', 'b']
# lst += [4, 5]           # Same as extend

# insert() - add at specific position
lst = [1, 2, 3]
lst.insert(1, 'x')        # [1, 'x', 2, 3]
lst.insert(0, 'y')        # ['y', 1, 'x', 2, 3] (beginning)
lst.insert(100, 'z')      # Adds at end (no error)
```

### **Removing Elements**

```python
lst = [1, 2, 3, 2, 4]

# remove() - remove first occurrence by value
lst.remove(2)             # [1, 3, 2, 4]
# lst.remove(99)          # ValueError: not in list

# pop() - remove and return by index
lst = [1, 2, 3]
x = lst.pop()             # x=3, lst=[1, 2] (last)
x = lst.pop(0)            # x=1, lst=[2] (first)
# lst.pop()               # IndexError if empty

# clear() - remove all
lst = [1, 2, 3]
lst.clear()               # []

# del statement (not a method)
lst = [1, 2, 3, 4, 5]
del lst[0]                # [2, 3, 4, 5]
del lst[1:3]              # [2, 5]
del lst[:]                # [] (clear)
```

### **Searching & Counting**

```python
lst = [1, 2, 3, 2, 4]

# index() - find first occurrence
lst.index(2)              # 1
lst.index(2, 2)           # 3 (start from index 2)
# lst.index(99)           # ValueError

# count() - count occurrences
lst.count(2)              # 2
lst.count(99)             # 0 (no error)

# Membership (not a method)
2 in lst                  # True
99 not in lst             # True
```

### **Sorting & Reversing**

```python
lst = [3, 1, 4, 1, 5]

# sort() - in-place sorting
lst.sort()                # [1, 1, 3, 4, 5]
lst.sort(reverse=True)    # [5, 4, 3, 1, 1]

# Sort by key
words = ["apple", "pie", "a", "cherry"]
words.sort(key=len)       # ['a', 'pie', 'apple', 'cherry']
words.sort(key=str.lower) # Case-insensitive

# Custom key
data = [(1, 'b'), (2, 'a'), (3, 'c')]
data.sort(key=lambda x: x[1])  # [(2,'a'), (1,'b'), (3,'c')]

# sorted() - returns new list (not a method)
lst = [3, 1, 4]
new_lst = sorted(lst)     # [1, 3, 4] (lst unchanged)

# reverse() - in-place reverse
lst = [1, 2, 3]
lst.reverse()             # [3, 2, 1]

# reversed() - returns iterator (not a method)
lst = [1, 2, 3]
list(reversed(lst))       # [3, 2, 1] (lst unchanged)
```

### **Copying**

```python
# copy() - shallow copy
lst = [1, 2, 3]
new_lst = lst.copy()      # New list
new_lst = lst[:]          # Same effect
new_lst = list(lst)       # Same effect

# Shallow copy gotcha
lst = [[1, 2], [3, 4]]
new_lst = lst.copy()
new_lst[0][0] = 99        # lst = [[99, 2], [3, 4]] (affected!)

# Deep copy (for nested)
import copy
lst = [[1, 2], [3, 4]]
new_lst = copy.deepcopy(lst)
new_lst[0][0] = 99        # lst unchanged
```

## 6. Tuple Methods (2 Only)

```python
tup = (1, 2, 3, 2, 4)

# count() - count occurrences
tup.count(2)              # 2

# index() - find first occurrence
tup.index(3)              # 2
tup.index(2, 2)           # 3 (start from index 2)
# tup.index(99)           # ValueError
```

## 7. List Operators

```python
# Concatenation
[1, 2] + [3, 4]           # [1, 2, 3, 4]
(1, 2) + (3, 4)           # (1, 2, 3, 4)

# Repetition
[1, 2] * 3                # [1, 2, 1, 2, 1, 2]
(1,) * 3                  # (1, 1, 1)

# Comparison (lexicographic)
[1, 2] < [1, 3]           # True
[1, 2] == [1, 2]          # True
[1, 2, 3] > [1, 2]        # True (longer wins if equal)

# Membership
1 in [1, 2, 3]            # True
[1, 2] in [[1, 2], [3]]   # True (sublist)

# Identity
a = [1, 2]
b = [1, 2]
a == b                    # True (same content)
a is b                    # False (different objects)

# Length
len([1, 2, 3])            # 3
len(())                   # 0
```

## 8. List Modification (Mutability)

```python
lst = [1, 2, 3, 4, 5]

# Item assignment
lst[0] = 99               # [99, 2, 3, 4, 5]
lst[-1] = 88              # [99, 2, 3, 4, 88]

# Slice assignment
lst[1:3] = [7, 8, 9]      # [99, 7, 8, 9, 4, 88]
lst[::2] = [0, 0, 0]      # [0, 7, 0, 9, 0, 88]

# Insert via slice
lst = [1, 2, 3]
lst[1:1] = [99]           # [1, 99, 2, 3]

# Delete via slice
lst = [1, 2, 3, 4, 5]
lst[1:3] = []             # [1, 4, 5]

# Tuples are immutable
tup = (1, 2, 3)
# tup[0] = 99             # TypeError
# tup.append(4)           # AttributeError
```

## 9. Unpacking & Packing

```python
# Basic unpacking
a, b, c = [1, 2, 3]       # a=1, b=2, c=3
a, b, c = (1, 2, 3)       # Works with tuples too

# Star unpacking (Python 3.5+)
a, *b, c = [1, 2, 3, 4, 5]  # a=1, b=[2,3,4], c=5
a, *b = [1, 2, 3]             # a=1, b=[2,3]
*a, b = [1, 2, 3]             # a=[1,2], b=3

# Ignore values
a, _, c = [1, 2, 3]       # _ conventionally means "ignore"
a, *_ = [1, 2, 3, 4]      # a=1, ignore rest

# Swap values
a, b = 1, 2
a, b = b, a               # a=2, b=1 (no temp variable!)

# Multiple assignment
a = b = c = []            # DANGER: all point to same list!
a, b, c = [], [], []      # Correct: separate lists

# Extended unpacking in function
def func(a, b, *args, **kwargs):
    pass
func(1, 2, 3, 4, x=5)     # args=(3,4), kwargs={'x':5}
```

## 10. Nested Lists & Tuples

```python
# 2D list (matrix)
matrix = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
]
matrix[0][1]              # 2
matrix[1]                 # [4, 5, 6]

# Flattening
nested = [[1, 2], [3, 4], [5]]
flat = [item for sublist in nested for item in sublist]
# [1, 2, 3, 4, 5]

# Deep nesting
data = [1, [2, [3, [4, [5]]]]]
data[1][1][1][1][0]       # 5

# Create 2D list (WRONG)
matrix = [[0] * 3] * 3    # All rows reference same list!
matrix[0][0] = 1          # All rows change!

# Create 2D list (CORRECT)
matrix = [[0] * 3 for _ in range(3)]
matrix[0][0] = 1          # Only first row changes
```

## 11. List Comprehensions (Advanced)

```python
# Basic
[x*2 for x in range(5)]                    # [0,2,4,6,8]

# With condition
[x for x in range(10) if x % 2]            # [1,3,5,7,9]

# If-else
[x if x > 0 else 0 for x in [-1, 2, -3]]   # [0,2,0]

# Nested loops
[(i,j) for i in range(3) for j in range(2)]
# [(0,0),(0,1),(1,0),(1,1),(2,0),(2,1)]

# Filter with condition
[x for x in range(10) if x % 2 if x > 5]   # [7, 9]

# Flatten 2D list
matrix = [[1,2], [3,4]]
[item for row in matrix for item in row]   # [1,2,3,4]

# Dictionary from list
{x: x**2 for x in range(5)}                # {0:0,1:1,2:4,3:9,4:16}

# Set from list
{x % 3 for x in range(10)}                 # {0, 1, 2}

# Generator (not list comprehension)
gen = (x*2 for x in range(5))              # Generator object
list(gen)                                  # [0,2,4,6,8]
```

## 12. Common Patterns & Idioms

### **Check Empty**
```python
lst = []
if not lst:               # Pythonic
    print("empty")

if len(lst) == 0:         # Works but verbose
    print("empty")
```

### **Reverse**
```python
lst = [1, 2, 3]
lst[::-1]                 # [3, 2, 1] (new list)
reversed(lst)             # Iterator
list(reversed(lst))       # [3, 2, 1] (new list)
lst.reverse()             # In-place
```

### **Find Max/Min**
```python
lst = [3, 1, 4, 1, 5]
max(lst)                  # 5
min(lst)                  # 1
max(lst, default=0)       # 0 if empty (Python 3.4+)

# With key
words = ["apple", "pie", "a"]
max(words, key=len)       # "apple"
```

### **Sum, Any, All**
```python
sum([1, 2, 3])            # 6
sum([1, 2, 3], 10)        # 16 (start value)

any([False, True, False]) # True (at least one True)
all([True, True, True])   # True (all True)

any([])                   # False
all([])                   # True (vacuous truth)
```

### **Enumerate**
```python
lst = ['a', 'b', 'c']
for i, val in enumerate(lst):
    print(i, val)         # 0 a, 1 b, 2 c

for i, val in enumerate(lst, 1):  # Start from 1
    print(i, val)         # 1 a, 2 b, 3 c

list(enumerate(lst))      # [(0,'a'), (1,'b'), (2,'c')]
```

### **Zip**
```python
a = [1, 2, 3]
b = ['a', 'b', 'c']
list(zip(a, b))           # [(1,'a'), (2,'b'), (3,'c')]

# Unequal lengths (stops at shortest)
a = [1, 2, 3, 4]
b = ['a', 'b']
list(zip(a, b))           # [(1,'a'), (2,'b')]

# Unzip
pairs = [(1,'a'), (2,'b'), (3,'c')]
nums, letters = zip(*pairs)  # nums=(1,2,3), letters=('a','b','c')

# zip_longest (from itertools)
from itertools import zip_longest
a = [1, 2, 3, 4]
b = ['a', 'b']
list(zip_longest(a, b, fillvalue='-'))  # [(1,'a'),(2,'b'),(3,'-'),(4,'-')]
```

### **Filter, Map, Reduce**
```python
# Filter (prefer list comprehension)
list(filter(lambda x: x > 0, [-1, 2, -3, 4]))  # [2, 4]
[x for x in [-1, 2, -3, 4] if x > 0]           # [2, 4] (better)

# Map (prefer list comprehension)
list(map(lambda x: x*2, [1, 2, 3]))            # [2, 4, 6]
[x*2 for x in [1, 2, 3]]                       # [2, 4, 6] (better)

# Reduce
from functools import reduce
reduce(lambda x, y: x+y, [1, 2, 3, 4])         # 10
sum([1, 2, 3, 4])                              # 10 (better)
```

### **Remove Duplicates**
```python
lst = [1, 2, 2, 3, 3, 3]

# Using set (order lost)
list(set(lst))            # [1, 2, 3]

# Preserve order (dict.fromkeys)
list(dict.fromkeys(lst))  # [1, 2, 3]

# Preserve order (manual)
seen = set()
result = [x for x in lst if not (x in seen or seen.add(x))]
```

### **List Rotation**
```python
lst = [1, 2, 3, 4, 5]

# Rotate right by 2
n = 2
lst[-n:] + lst[:-n]       # [4, 5, 1, 2, 3]

# Rotate left by 2
lst[n:] + lst[:n]         # [3, 4, 5, 1, 2]

# Using collections.deque (efficient)
from collections import deque
d = deque([1, 2, 3, 4, 5])
d.rotate(2)               # deque([4, 5, 1, 2, 3])
```

## 13. Performance & Memory

### **Time Complexity**

| Operation | List | Tuple |
|-----------|------|-------|
| Access `lst[i]` | O(1) | O(1) |
| Search `x in lst` | O(n) | O(n) |
| Append | O(1) amortized | N/A |
| Pop last | O(1) | N/A |
| Pop first | O(n) | N/A |
| Insert | O(n) | N/A |
| Delete | O(n) | N/A |
| Copy | O(n) | O(n) |
| Sort | O(n log n) | N/A |

### **Memory Usage**
```python
import sys

sys.getsizeof([])         # 56 bytes (empty list)
sys.getsizeof([1,2,3])    # 88 bytes
sys.getsizeof(())         # 40 bytes (empty tuple)
sys.getsizeof((1,2,3))    # 64 bytes

# Lists over-allocate for growth
lst = []
for i in range(10):
    lst.append(i)
    print(sys.getsizeof(lst))  # Grows in chunks

# Tuples use exact size
```

### **When to Use What**

**Use Lists when:**
- Data changes (add/remove/modify)
- Need sorting/reversing
- Building collections dynamically
- Order matters and changes

**Use Tuples when:**
- Data is fixed/constant
- Need hashable type (dict keys, set members)
- Returning multiple values from function
- Performance critical (faster)
- Want to prevent accidental modification

## 14. Edge Cases & Gotchas

```python
# Reference vs copy
a = [1, 2, 3]
b = a                     # b points to same list!
b.append(4)               # a is also [1, 2, 3, 4]

# Correct copy
b = a.copy()              # or a[:] or list(a)

# Mutable default argument (DANGER!)
def add_to_list(item, lst=[]):  # BAD!
    lst.append(item)
    return lst

add_to_list(1)            # [1]
add_to_list(2)            # [1, 2] (reuses same list!)

# Correct way
def add_to_list(item, lst=None):
    if lst is None:
        lst = []
    lst.append(item)
    return lst

# List multiplication reference
lst = [[]] * 3            # [[],[],[]] (same object!)
lst[0].append(1)          # [[1],[1],[1]] (all change!)

# Correct
lst = [[] for _ in range(3)]

# Tuple immutability doesn't mean content is immutable
tup = ([1, 2], [3, 4])
# tup[0] = [5, 6]         # TypeError
tup[0].append(5)          # Works! ([1,2,5], [3,4])

# Comparison with None
lst = [1, 2, None, 4]
None in lst               # True
lst == None               # False (correct)
lst is None               # False (correct)

# Sorting mixed types (Python 3)
# [1, 'a', 3].sort()      # TypeError in Python 3
# Python 2 allowed this (numbers < strings)

# Index on empty list
lst = []
# lst[0]                  # IndexError
# lst.pop()               # IndexError
# lst.remove(1)           # ValueError

# Negative indexing wrap
lst = [1, 2, 3]
lst[-10]                  # IndexError (even though negative)

# Slice assignment length mismatch (OK)
lst = [1, 2, 3, 4, 5]
lst[1:3] = [99]           # [1, 99, 4, 5] (length changes)

# Extended slice assignment must match
lst = [1, 2, 3, 4, 5]
# lst[::2] = [99]         # ValueError: length mismatch
lst[::2] = [0, 0, 0]      # OK: [0, 2, 0, 4, 0]
```

## 15. List vs Array vs Deque

```python
# List (flexible, general purpose)
lst = [1, 2, 3, 4]

# Array (typed, memory efficient)
import array
arr = array.array('i', [1, 2, 3, 4])  # 'i' = signed int
# Faster for numeric data, less memory

# Deque (double-ended queue)
from collections import deque
dq = deque([1, 2, 3, 4])
dq.appendleft(0)          # O(1) - fast!
dq.popleft()              # O(1) - fast!
# Use for queues/stacks, not random access

# NumPy array (scientific computing)
import numpy as np
arr = np.array([1, 2, 3, 4])
# Vectorized operations, mathematical functions
```

## 16. Named Tuples

```python
from collections import namedtuple

# Define
Point = namedtuple('Point', ['x', 'y'])
p = Point(1, 2)

# Access
p.x                       # 1
p[0]                      # 1 (still works)

# Immutable
# p.x = 3                 # AttributeError

# Convert to dict
p._asdict()               # {'x': 1, 'y': 2}

# Replace (returns new)
p2 = p._replace(x=3)      # Point(x=3, y=2)

# Useful for returning multiple values
def get_coordinates():
    return Point(10, 20)

# Better than regular tuple
def get_coords():
    return (10, 20)       # What is 0 and 1?
```

## 17. Advanced Techniques

### **List as Stack (LIFO)**
```python
stack = []
stack.append(1)           # Push
stack.append(2)
stack.pop()               # 2 (Pop)
```

### **List as Queue (FIFO) - DON'T**
```python
queue = []
queue.append(1)           # Enqueue
queue.pop(0)              # Dequeue (O(n) - SLOW!)

# Use deque instead
from collections import deque
queue = deque()
queue.append(1)           # Enqueue
queue.popleft()           # Dequeue (O(1) - FAST!)
```

### **Priority Queue**
```python
import heapq

heap = []
heapq.heappush(heap, 3)
heapq.heappush(heap, 1)
heapq.heappush(heap, 2)
heapq.heappop(heap)       # 1 (smallest)
```

### **Binary Search**
```python
import bisect

lst = [1, 3, 5, 7, 9]
bisect.bisect_left(lst, 5)   # 2 (index)
bisect.bisect_right(lst, 5)  # 3
bisect.insort(lst, 4)        # [1,3,4,5,7,9] (maintains sort)
```

### **Chain Multiple Lists**
```python
from itertools import chain

a = [1, 2]
b = [3, 4]
c = [5, 6]
list(chain(a, b, c))      # [1, 2, 3, 4, 5, 6]

# vs concatenation
a + b + c                 # [1, 2, 3, 4, 5, 6]
# chain is lazy (memory efficient)
```

## 18. Complete Method Reference

### **List Methods (11)**
```
append(x)       - Add x to end
extend(iterable)- Add all items from iterable
insert(i, x)    - Insert x at position i
remove(x)       - Remove first occurrence of x
pop([i])        - Remove and return item at i (default: -1)
clear()         - Remove all items
index(x[,s,e])  - Return index of x (start, end)
count(x)        - Count occurrences of x
sort()          - Sort in place
reverse()       - Reverse in place
copy()          - Return shallow copy
```

### **Tuple Methods (2)**
```
count(x)        - Count occurrences of x
index(x[,s,e])  - Return index of x (start, end)
```

### **Built-in Functions (work on both)**
```
len()           - Length
max()           - Maximum value
min()           - Minimum value
sum()           - Sum (numeric only)
sorted()        - Return sorted list
reversed()      - Return reverse iterator
any()           - True if any element is True
all()           - True if all elements are True
enumerate()     - Return (index, value) pairs
zip()           - Combine multiple iterables
```

## 19. Performance Tips

1. **Use list comprehension** over `map()`/`filter()`
2. **Use `extend()`** instead of repeated `append()`
3. **Pre-allocate** if size known: `[None] * n`
4. **Avoid `insert(0, x)`** - use `deque.appendleft()`
5. **Avoid `pop(0)`** - use `deque.popleft()`
6. **Use tuples** for immutable data (faster, less memory)
7. **Use `in` for membership** instead of `index()` + try/except
8. **Sort once** then use `bisect` for insertions
9. **Use `any()`/`all()`** instead of comprehension + `True in`
10. **Use generators** for large datasets to save memory

## 20. Quick Reference Table

| Task | List | Tuple |
|------|------|-------|
| Create | `[1,2,3]` | `(1,2,3)` or `1,2,3` |
| Empty | `[]` | `()` |
| Single | `[1]` | `(1,)` (comma!) |
| Access | `lst[0]` | `tup[0]` |
| Slice | `lst[1:3]` | `tup[1:3]` |
| Length | `len(lst)` | `len(tup)` |
| Add | `.append(x)` | N/A |
| Remove | `.remove(x)` | N/A |
| Sort | `.sort()` | `sorted(tup)` |
| Reverse | `.reverse()` | `tup[::-1]` |
| Copy | `.copy()` | `tuple(tup)` |
| Count | `.count(x)` | `.count(x)` |
| Find | `.index(x)` | `.index(x)` |
| Concat | `+` | `+` |
| Repeat | `*` | `*` |
| Check | `x in lst` | `x in tup` |

## 21. Memory & Internals

### **List Growth Strategy**
```python
# Lists over-allocate to minimize reallocation
import sys

lst = []
for i in range(20):
    print(f"len={len(lst)}, size={sys.getsizeof(lst)}")
    lst.append(i)

# Output shows size grows in jumps:
# 0 items: 56 bytes
# 1-4 items: 88 bytes
# 5-8 items: 120 bytes
# etc. (growth pattern: 0, 4, 8, 16, 25, 35, 46, 58, 72, 88...)
```

### **Reference Counting**
```python
import sys

lst = [1, 2, 3]
sys.getrefcount(lst)      # 2 (one from lst, one from getrefcount)

a = lst                   # Share reference
sys.getrefcount(lst)      # 3

del a                     # Remove reference
sys.getrefcount(lst)      # 2
```

### **Tuple Immutability Benefits**
```python
# Tuples can be dict keys
point = (10, 20)
data = {point: "location"}  # Works!

# Lists cannot
# point = [10, 20]
# data = {point: "location"}  # TypeError: unhashable

# Tuples can be set members
coordinates = {(0, 0), (1, 1), (2, 2)}

# Multiple tuple references might share memory (CPython optimization)
a = (1, 2, 3)
b = (1, 2, 3)
a is b                    # Might be True (small tuples)

# Lists never share memory this way
a = [1, 2, 3]
b = [1, 2, 3]
a is b                    # Always False
```

## 22. Type Hints (Python 3.5+)

```python
from typing import List, Tuple, Optional, Union, Any

# Basic
def process_list(items: List[int]) -> List[str]:
    return [str(x) for x in items]

# Tuple with specific types
def get_user() -> Tuple[str, int, bool]:
    return ("Alice", 30, True)

# Variable length tuple
def get_scores() -> Tuple[int, ...]:
    return (95, 87, 92)

# Optional
def find_item(lst: List[int], target: int) -> Optional[int]:
    try:
        return lst.index(target)
    except ValueError:
        return None

# Union
def process(data: Union[List[int], Tuple[int, ...]]) -> int:
    return sum(data)

# Nested
matrix: List[List[int]] = [[1, 2], [3, 4]]

# Any type
mixed: List[Any] = [1, "two", 3.0, [4]]

# Python 3.9+ (lowercase)
def process_list(items: list[int]) -> list[str]:
    return [str(x) for x in items]
```

## 23. Common Interview Questions & Solutions

### **Two Sum**
```python
def two_sum(nums: List[int], target: int) -> List[int]:
    seen = {}
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    return []
```

### **Rotate Array**
```python
def rotate(nums: List[int], k: int) -> None:
    k = k % len(nums)
    nums[:] = nums[-k:] + nums[:-k]
```

### **Remove Duplicates (in-place)**
```python
def remove_duplicates(nums: List[int]) -> int:
    if not nums:
        return 0
    j = 0
    for i in range(1, len(nums)):
        if nums[i] != nums[j]:
            j += 1
            nums[j] = nums[i]
    return j + 1
```

### **Merge Sorted Lists**
```python
def merge(nums1: List[int], m: int, nums2: List[int], n: int) -> None:
    p1, p2, p = m - 1, n - 1, m + n - 1
    while p2 >= 0:
        if p1 >= 0 and nums1[p1] > nums2[p2]:
            nums1[p] = nums1[p1]
            p1 -= 1
        else:
            nums1[p] = nums2[p2]
            p2 -= 1
        p -= 1
```

### **Find Missing Number**
```python
def find_missing(nums: List[int]) -> int:
    n = len(nums)
    return n * (n + 1) // 2 - sum(nums)
```

### **Flatten Nested List**
```python
def flatten(nested):
    result = []
    for item in nested:
        if isinstance(item, list):
            result.extend(flatten(item))
        else:
            result.append(item)
    return result

# Example: flatten([[1, 2], [3, [4, 5]]]) -> [1, 2, 3, 4, 5]
```

## 24. Debugging & Best Practices

### **Debugging Techniques**
```python
# Print with index
lst = ['a', 'b', 'c']
for i, val in enumerate(lst):
    print(f"Index {i}: {val}")

# Check type
type(lst)                 # <class 'list'>
isinstance(lst, list)     # True

# Check if empty
if not lst:
    print("Empty!")

# Shallow vs deep copy visualization
import copy
original = [[1, 2], [3, 4]]
shallow = original.copy()
deep = copy.deepcopy(original)

shallow[0][0] = 99        # Affects original!
deep[1][0] = 88           # Doesn't affect original
```

### **Best Practices**

**DO:**
```python
# Use list comprehension for readability
squared = [x**2 for x in range(10)]

# Use meaningful names
user_names = ["Alice", "Bob"]  # Good
x = ["Alice", "Bob"]           # Bad

# Check before accessing
if lst:
    first = lst[0]

# Use enumerate for index+value
for i, val in enumerate(lst):
    print(i, val)

# Unpack for clarity
name, age, city = user_data
```

**DON'T:**
```python
# Don't use mutable default arguments
def bad(lst=[]):           # BAD!
    lst.append(1)
    return lst

# Don't modify list while iterating
for item in lst:
    if condition:
        lst.remove(item)   # BAD! (skips items)

# Correct way:
lst = [x for x in lst if not condition]

# Don't compare with empty list using ==
if lst == []:              # Works but verbose
    pass

# Better:
if not lst:
    pass

# Don't use + in loops
result = []
for item in items:
    result = result + [item]  # BAD! (O(n²))

# Use append or extend:
result = []
for item in items:
    result.append(item)    # GOOD! (O(n))
```

## 25. Python Version Differences

### **Python 2 vs 3**
```python
# Python 2: range() returned list
range(5)                  # [0, 1, 2, 3, 4]

# Python 3: range() returns range object
range(5)                  # range(0, 5)
list(range(5))            # [0, 1, 2, 3, 4]

# Python 2: Comparison of mixed types
[1, 2] < "string"         # True (in Python 2)

# Python 3: Comparison raises TypeError
# [1, 2] < "string"       # TypeError

# Python 2: map() returns list
map(str, [1, 2, 3])       # ['1', '2', '3']

# Python 3: map() returns iterator
map(str, [1, 2, 3])       # <map object>
list(map(str, [1, 2, 3])) # ['1', '2', '3']
```

### **Modern Features**
```python
# Python 3.5+: Extended unpacking
a, *b, c = [1, 2, 3, 4, 5]

# Python 3.6+: f-strings
name = "Alice"
f"Hello, {name}"

# Python 3.8+: Walrus operator
if (n := len(lst)) > 10:
    print(f"List has {n} items")

# Python 3.9+: Union operator
list1 | list2             # Still not implemented for lists
# Use: list1 + list2

# Python 3.9+: Lowercase type hints
def func(items: list[int]) -> list[str]:
    pass

# Python 3.10+: Match statement
match lst:
    case []:
        print("empty")
    case [x]:
        print(f"one item: {x}")
    case [x, y]:
        print(f"two items: {x}, {y}")
    case _:
        print("many items")
```

## 26. Real-World Examples

### **Data Processing Pipeline**
```python
# Raw data
raw_data = ["  Alice:30  ", "Bob:25", "  Charlie:35  "]

# Process pipeline
users = [
    tuple(item.strip().split(':'))
    for item in raw_data
]
# [('Alice', '30'), ('Bob', '25'), ('Charlie', '35')]

# Convert to proper types
users = [
    (name, int(age))
    for name, age in users
]

# Filter adults
adults = [(name, age) for name, age in users if age >= 18]
```

### **Moving Average**
```python
def moving_average(data, window):
    return [
        sum(data[i:i+window]) / window
        for i in range(len(data) - window + 1)
    ]

prices = [10, 12, 13, 11, 14, 15]
moving_average(prices, 3)  # [11.67, 12.0, 12.67, 13.33]
```

### **Group By**
```python
from itertools import groupby

data = [
    ('A', 1), ('A', 2), ('B', 3), ('B', 4), ('A', 5)
]

# Sort first (required for groupby)
data.sort(key=lambda x: x[0])

# Group
for key, group in groupby(data, key=lambda x: x[0]):
    print(key, list(group))
# A [('A', 1), ('A', 2), ('A', 5)]
# B [('B', 3), ('B', 4)]
```

### **Pagination**
```python
def paginate(items, page_size):
    return [
        items[i:i+page_size]
        for i in range(0, len(items), page_size)
    ]

items = list(range(10))
pages = paginate(items, 3)
# [[0,1,2], [3,4,5], [6,7,8], [9]]
```

### **LRU Cache (List Implementation)**
```python
class LRUCache:
    def __init__(self, capacity):
        self.cache = []
        self.capacity = capacity
    
    def get(self, key):
        if key in self.cache:
            self.cache.remove(key)
            self.cache.append(key)
            return key
        return -1
    
    def put(self, key):
        if key in self.cache:
            self.cache.remove(key)
        elif len(self.cache) >= self.capacity:
            self.cache.pop(0)
        self.cache.append(key)
```

## 27. Advanced Memory Optimization

### **Generators vs Lists**
```python
# List (stores everything in memory)
squares_list = [x**2 for x in range(1000000)]  # ~8MB

# Generator (computes on-the-fly)
squares_gen = (x**2 for x in range(1000000))   # ~128 bytes

# Use generator for large datasets
import sys
sys.getsizeof(squares_list)  # ~8000000 bytes
sys.getsizeof(squares_gen)   # ~128 bytes
```

### **Slots for Memory Efficiency**
```python
# Without slots
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

# With slots (less memory)
class Point:
    __slots__ = ['x', 'y']
    def __init__(self, x, y):
        self.x = x
        self.y = y

# For list of objects, slots save significant memory
points = [Point(i, i+1) for i in range(10000)]
```

### **Array Module for Numeric Data**
```python
import array

# List of integers (8 bytes each + overhead)
lst = [1, 2, 3, 4, 5] * 1000  # ~40KB

# Array of integers (4 bytes each, no overhead)
arr = array.array('i', [1, 2, 3, 4, 5] * 1000)  # ~20KB

# 50% memory savings for large numeric datasets
```

## 28. Testing & Validation

```python
import unittest

class TestListOperations(unittest.TestCase):
    def test_empty_list(self):
        lst = []
        self.assertEqual(len(lst), 0)
        self.assertFalse(lst)
    
    def test_append(self):
        lst = [1, 2]
        lst.append(3)
        self.assertEqual(lst, [1, 2, 3])
    
    def test_slicing(self):
        lst = [1, 2, 3, 4, 5]
        self.assertEqual(lst[1:3], [2, 3])
        self.assertEqual(lst[::-1], [5, 4, 3, 2, 1])
    
    def test_tuple_immutability(self):
        tup = (1, 2, 3)
        with self.assertRaises(TypeError):
            tup[0] = 99

# Run tests
# unittest.main()
```

## 29. Cheat Sheet Summary

```python
# CREATION
lst = [1, 2, 3]                    # List
tup = (1, 2, 3)                    # Tuple
lst = list(range(5))               # [0, 1, 2, 3, 4]
lst = [x*2 for x in range(5)]      # [0, 2, 4, 6, 8]

# ACCESS
lst[0], lst[-1]                    # First, last
lst[1:3], lst[::2], lst[::-1]      # Slice, step, reverse

# MODIFY (list only)
lst.append(x)                      # Add to end
lst.extend([x, y])                 # Add multiple
lst.insert(i, x)                   # Insert at position
lst.remove(x)                      # Remove first x
lst.pop()                          # Remove & return last
lst[i] = x                         # Replace item

# SEARCH
x in lst                           # Membership
lst.index(x)                       # Find index
lst.count(x)                       # Count occurrences

# SORT/REVERSE
lst.sort()                         # In-place sort
sorted(lst)                        # Return new sorted
lst.reverse()                      # In-place reverse
lst[::-1]                          # Return new reversed

# AGGREGATE
len(lst), sum(lst)                 # Length, sum
max(lst), min(lst)                 # Max, min
any(lst), all(lst)                 # Any true, all true

# ITERATION
for x in lst: ...                  # Simple loop
for i, x in enumerate(lst): ...    # With index
[f(x) for x in lst if cond]        # Comprehension

# UNPACKING
a, b, c = [1, 2, 3]                # Basic
a, *b, c = [1, 2, 3, 4, 5]         # Extended (a=1, b=[2,3,4], c=5)

# COPY
new = lst.copy()                   # Shallow copy
new = lst[:]                       # Same
import copy; new = copy.deepcopy(lst)  # Deep copy

# COMBINE
lst1 + lst2                        # Concatenate
lst * 3                            # Repeat
zip(lst1, lst2)                    # Combine parallel lists
```

## 30. Final Tips & Reminders

✅ **Remember:**
- Lists are mutable, tuples are immutable
- Use tuples for data that shouldn't change
- List comprehensions > map/filter
- Don't modify list while iterating over it
- Shallow copy vs deep copy matters for nested structures
- Single element tuple needs comma: `(1,)`
- Empty list check: `if not lst:`
- Use `extend()` not `+` for adding multiple items
- `append()` is O(1), `insert(0, x)` is O(n)
- Tuples can be dict keys, lists cannot

⚡ **Performance:**
- Tuple operations are faster than list
- Use generators for large datasets
- Pre-allocate lists when size is known
- Use `deque` for frequent insertions/deletions at both ends
- Use `array.array` for large numeric datasets

🐛 **Common Bugs:**
- Mutable default arguments
- Modifying list during iteration
- Reference vs copy confusion
- List multiplication creating references
- Forgetting comma in single-element tuple

🎯 **Best Use Cases:**
- **List**: Shopping cart, todo list, log entries, dynamic collections
- **Tuple**: Coordinates, RGB colors, database records, function returns, dict keys
