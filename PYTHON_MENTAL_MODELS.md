# PYTHON - MENTAL MODELS

> Master Guide for Python Mastery & Technical Interviews

---

## TABLE OF CONTENTS
1. [Python Object Model](#1-python-object-model)
2. [Memory Management](#2-memory-management)
3. [Data Structures Deep Dive](#3-data-structures-deep-dive)
4. [Mutability & Immutability](#4-mutability--immutability)
5. [Functions & Closures](#5-functions--closures)
6. [Decorators Mental Model](#6-decorators-mental-model)
7. [Object-Oriented Python](#7-object-oriented-python)
8. [Iterators & Generators](#8-iterators--generators)
9. [Context Managers](#9-context-managers)
10. [Exception Handling](#10-exception-handling)
11. [Concurrency & Parallelism](#11-concurrency--parallelism)
12. [Python Internals](#12-python-internals)
13. [Pythonic Patterns](#13-pythonic-patterns)
14. [Type Hints & Static Typing](#14-type-hints--static-typing)
15. [Common Gotchas](#15-common-gotchas)
16. [Performance Optimization](#16-performance-optimization)
17. [Quick Reference Card](#17-quick-reference-card)
18. [Learning Path](#18-learning-path)

---

## 1. PYTHON OBJECT MODEL

### Everything Is An Object

```
In Python, EVERYTHING is an object:
  - Integers, strings, floats
  - Functions, classes, modules
  - Even None, True, False

┌─────────────────────────────────────────────────────────────────┐
│                     Python Object Model                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   x = 42                                                        │
│                                                                 │
│   Variable 'x'          Object in Memory                        │
│   ┌─────────┐           ┌─────────────────┐                    │
│   │    x    │──────────►│ type: int       │                    │
│   └─────────┘           │ value: 42       │                    │
│   (name/reference)      │ id: 140234...   │                    │
│                         │ refcount: 1     │                    │
│                         └─────────────────┘                    │
│                                                                 │
│   x is a NAME that REFERENCES an object                         │
│   Assignment binds names to objects, doesn't copy values        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Object Identity, Type, and Value

```python
x = [1, 2, 3]

# Three fundamental properties of every object:
id(x)       # Identity: unique integer (memory address in CPython)
type(x)     # Type: determines behavior and allowed operations
x           # Value: the actual data

# Identity comparison vs Value comparison
a = [1, 2, 3]
b = [1, 2, 3]
c = a

a == b      # True  (same VALUE)
a is b      # False (different IDENTITY/objects)
a is c      # True  (same object)

# Visual representation:
#   a ─────►┌─────────┐
#           │ [1,2,3] │
#   c ─────►└─────────┘
#
#   b ─────►┌─────────┐
#           │ [1,2,3] │  (different object, same value)
#           └─────────┘
```

### Type Hierarchy

```
                        object
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
    Numeric            Sequence           Mapping
        │                  │                  │
   ┌────┼────┐      ┌──────┼──────┐          │
   │    │    │      │      │      │          │
  int float complex str   list  tuple       dict
        │
      bool (subclass of int!)

# Everything inherits from 'object'
isinstance(42, object)      # True
isinstance("hi", object)    # True
isinstance(None, object)    # True
isinstance(len, object)     # True (functions too!)

# Type checking
isinstance(x, int)          # Preferred
type(x) == int              # Less flexible (no inheritance)
```

### Attribute Lookup (MRO)

```
When you access obj.attr, Python searches in order:

1. Instance __dict__
2. Class __dict__
3. Parent classes (following MRO)
4. __getattr__ if defined

class A:
    x = 'class A'

class B(A):
    pass

b = B()
b.y = 'instance'

# Lookup b.x:
# 1. b.__dict__ → {'y': 'instance'} - not found
# 2. B.__dict__ → {} - not found
# 3. A.__dict__ → {'x': 'class A'} - FOUND!

B.__mro__  # (<class 'B'>, <class 'A'>, <class 'object'>)
```

---

## 2. MEMORY MANAGEMENT

### Reference Counting

```
┌─────────────────────────────────────────────────────────────────┐
│                   Reference Counting                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   a = [1, 2, 3]     # refcount = 1                              │
│   b = a             # refcount = 2                              │
│   c = a             # refcount = 3                              │
│                                                                 │
│       a ──────┐                                                 │
│               │     ┌─────────────────┐                        │
│       b ──────┼────►│ [1, 2, 3]       │                        │
│               │     │ refcount: 3     │                        │
│       c ──────┘     └─────────────────┘                        │
│                                                                 │
│   del b             # refcount = 2                              │
│   c = None          # refcount = 1                              │
│   a = "new"         # refcount = 0 → FREED!                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

import sys
sys.getrefcount(obj)  # Note: +1 because getrefcount itself holds a ref
```

### Garbage Collection (Cycle Detection)

```
PROBLEM: Reference counting fails for circular references

a = []
b = []
a.append(b)  # a → b
b.append(a)  # b → a

del a, b
# Both objects have refcount 1 (from each other)
# But they're unreachable! Memory leak?

SOLUTION: Python's garbage collector detects cycles

┌───────────────────────────────────────────────────────────────┐
│                 Generational GC                               │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  Generation 0: New objects (collected most frequently)        │
│       ↓ survives                                              │
│  Generation 1: Survived one collection                        │
│       ↓ survives                                              │
│  Generation 2: Long-lived objects (collected rarely)          │
│                                                               │
│  import gc                                                    │
│  gc.get_threshold()  # (700, 10, 10) - default thresholds     │
│  gc.collect()        # Force collection                       │
│  gc.disable()        # Disable (for performance-critical)     │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### Small Integer & String Caching

```python
# Python caches small integers (-5 to 256)
a = 256
b = 256
a is b  # True (same object!)

a = 257
b = 257
a is b  # False (different objects)

# String interning (automatic for identifier-like strings)
a = "hello"
b = "hello"
a is b  # True (interned)

a = "hello world"
b = "hello world"
a is b  # Might be False (spaces prevent auto-interning)

# Force interning
import sys
a = sys.intern("hello world")
b = sys.intern("hello world")
a is b  # True

# WHY THIS MATTERS:
# - Memory efficiency for common values
# - Faster comparisons (id check vs value check)
# - Don't rely on this for correctness! Use ==
```

### Copy Semantics

```python
import copy

# SHALLOW COPY: New container, same nested objects
original = [[1, 2], [3, 4]]
shallow = copy.copy(original)
# or: shallow = original[:]
# or: shallow = list(original)

original[0][0] = 'X'
print(shallow)  # [['X', 2], [3, 4]] - nested list affected!

# DEEP COPY: New container AND new nested objects
original = [[1, 2], [3, 4]]
deep = copy.deepcopy(original)

original[0][0] = 'X'
print(deep)  # [[1, 2], [3, 4]] - independent copy

# Visual:
# SHALLOW:                      DEEP:
# original ──► [ref1, ref2]     original ──► [ref1, ref2]
# shallow  ──► [ref1, ref2]                    ↓      ↓
#                ↓      ↓                   [1,2]  [3,4]
#             [1,2]  [3,4]
#                                deep     ──► [ref3, ref4]
#                                               ↓      ↓
#                                            [1,2]  [3,4]
```

---

## 3. DATA STRUCTURES DEEP DIVE

### List (Dynamic Array)

```
┌─────────────────────────────────────────────────────────────────┐
│                        Python List                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Internal: Dynamic array of PyObject* pointers                  │
│                                                                 │
│  lst = [1, 'hello', 3.14]                                       │
│                                                                 │
│  ┌─────┬─────┬─────┬─────┬─────┐                               │
│  │ ptr │ ptr │ ptr │     │     │  ← Over-allocated for growth   │
│  └──┬──┴──┬──┴──┬──┴─────┴─────┘                               │
│     │     │     │                                               │
│     ▼     ▼     ▼                                               │
│    [1]  ['hi'] [3.14]   (objects stored elsewhere)              │
│                                                                 │
│  Time Complexities:                                             │
│  ┌────────────────────┬─────────────┐                          │
│  │ Operation          │ Complexity  │                          │
│  ├────────────────────┼─────────────┤                          │
│  │ Index access [i]   │ O(1)        │                          │
│  │ Append             │ O(1) amort  │                          │
│  │ Pop from end       │ O(1)        │                          │
│  │ Pop from start     │ O(n)        │                          │
│  │ Insert at index    │ O(n)        │                          │
│  │ Search (in)        │ O(n)        │                          │
│  │ Slice [i:j]        │ O(j-i)      │                          │
│  │ Sort               │ O(n log n)  │                          │
│  └────────────────────┴─────────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

# List methods
lst.append(x)      # Add to end
lst.extend(iter)   # Add all from iterable
lst.insert(i, x)   # Insert at position
lst.pop()          # Remove & return last
lst.pop(i)         # Remove & return at index
lst.remove(x)      # Remove first occurrence (ValueError if not found)
lst.index(x)       # Find index of first occurrence
lst.count(x)       # Count occurrences
lst.sort()         # In-place sort
lst.reverse()      # In-place reverse
```

### Dictionary (Hash Map)

```
┌─────────────────────────────────────────────────────────────────┐
│                      Python Dict                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Hash Table: key → hash → index → value                         │
│                                                                 │
│  d = {'a': 1, 'b': 2}                                           │
│                                                                 │
│  hash('a') = 12416037344  →  index = hash % table_size          │
│                                                                 │
│  ┌─────┬──────────┬───────┐                                    │
│  │ idx │   key    │ value │                                    │
│  ├─────┼──────────┼───────┤                                    │
│  │  0  │   None   │  None │                                    │
│  │  1  │   'a'    │   1   │  ← hash('a') % size = 1            │
│  │  2  │   None   │  None │                                    │
│  │  3  │   'b'    │   2   │  ← hash('b') % size = 3            │
│  │ ... │   ...    │  ...  │                                    │
│  └─────┴──────────┴───────┘                                    │
│                                                                 │
│  Since Python 3.7: Maintains insertion order!                   │
│                                                                 │
│  Time Complexities:                                             │
│  ┌────────────────────┬────────────────┐                       │
│  │ Operation          │ Average │ Worst│                       │
│  ├────────────────────┼─────────┼──────┤                       │
│  │ Get/Set/Delete     │ O(1)    │ O(n) │                       │
│  │ Search (in)        │ O(1)    │ O(n) │                       │
│  │ Iteration          │ O(n)    │ O(n) │                       │
│  └────────────────────┴─────────┴──────┘                       │
│  Worst case: hash collisions (rare with good hash function)     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

# Keys must be HASHABLE (immutable): str, int, tuple (if contents hashable)
# Lists, dicts, sets are NOT hashable

# Dict methods
d.get(key, default)        # Get with default (no KeyError)
d.setdefault(key, default) # Get or set if missing
d.pop(key, default)        # Remove and return
d.update(other_dict)       # Merge another dict
d.keys()                   # View of keys
d.values()                 # View of values
d.items()                  # View of (key, value) tuples

# Python 3.9+ merge operators
d1 | d2                    # Merge (new dict)
d1 |= d2                   # Update in-place
```

### Set (Hash Set)

```
┌─────────────────────────────────────────────────────────────────┐
│                       Python Set                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Hash table with keys only (no values)                          │
│  Unordered, no duplicates                                       │
│                                                                 │
│  s = {1, 2, 3}                                                  │
│                                                                 │
│  Time Complexities:                                             │
│  ┌────────────────────┬─────────────┐                          │
│  │ Operation          │ Average     │                          │
│  ├────────────────────┼─────────────┤                          │
│  │ Add                │ O(1)        │                          │
│  │ Remove             │ O(1)        │                          │
│  │ Search (in)        │ O(1)        │  ← Big advantage!        │
│  │ Union              │ O(n+m)      │                          │
│  │ Intersection       │ O(min(n,m)) │                          │
│  └────────────────────┴─────────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

# Set operations
a = {1, 2, 3}
b = {2, 3, 4}

a | b    # Union:        {1, 2, 3, 4}
a & b    # Intersection: {2, 3}
a - b    # Difference:   {1}
a ^ b    # Symmetric:    {1, 4}

a.add(x)          # Add element
a.remove(x)       # Remove (KeyError if missing)
a.discard(x)      # Remove (no error if missing)
a.pop()           # Remove & return arbitrary element

# frozenset: immutable set (hashable, can be dict key)
fs = frozenset([1, 2, 3])
```

### Tuple (Immutable Sequence)

```
┌─────────────────────────────────────────────────────────────────┐
│                       Python Tuple                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Immutable sequence - fixed size, contents can't change         │
│                                                                 │
│  t = (1, 'hello', 3.14)                                         │
│                                                                 │
│  WHY USE TUPLES?                                                │
│  • Hashable (can be dict keys if contents hashable)             │
│  • Slightly faster than lists                                   │
│  • Signal "this data won't change"                              │
│  • Unpacking: x, y = (1, 2)                                     │
│  • Named tuples for lightweight objects                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

# Named tuples - lightweight classes
from collections import namedtuple

Point = namedtuple('Point', ['x', 'y'])
p = Point(3, 4)
p.x, p.y    # 3, 4
p[0], p[1]  # Also works

# typing.NamedTuple (Python 3.6+)
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float

    def distance(self):
        return (self.x**2 + self.y**2) ** 0.5
```

### Collections Module Highlights

```python
from collections import (
    Counter,
    defaultdict,
    deque,
    OrderedDict,
    ChainMap
)

# COUNTER - Count hashable objects
words = ['a', 'b', 'a', 'c', 'a', 'b']
count = Counter(words)        # Counter({'a': 3, 'b': 2, 'c': 1})
count.most_common(2)          # [('a', 3), ('b', 2)]
count['a']                    # 3
count['z']                    # 0 (no KeyError!)
count.update(['a', 'a'])      # Add more
count1 + count2               # Combine counters
count1 - count2               # Subtract (keeps positive)

# DEFAULTDICT - Dict with default factory
graph = defaultdict(list)
graph['a'].append('b')        # No KeyError!
word_count = defaultdict(int)
word_count['hello'] += 1      # Starts at 0

# DEQUE - Double-ended queue O(1) both ends
dq = deque([1, 2, 3])
dq.appendleft(0)              # O(1)
dq.popleft()                  # O(1)
dq.append(4)                  # O(1)
dq.pop()                      # O(1)
dq.rotate(1)                  # Rotate right
deque(maxlen=3)               # Fixed-size, auto-discard oldest

# ORDEREDDICT - Remember insertion order (less needed in 3.7+)
# Still useful for: move_to_end(), equality considers order

# CHAINMAP - Search multiple dicts
defaults = {'color': 'red', 'size': 10}
overrides = {'color': 'blue'}
combined = ChainMap(overrides, defaults)
combined['color']  # 'blue' (found in first dict)
combined['size']   # 10 (found in second dict)
```

---

## 4. MUTABILITY & IMMUTABILITY

### The Mutability Matrix

```
┌─────────────────────────────────────────────────────────────────┐
│                    IMMUTABLE                                    │
├─────────────────────────────────────────────────────────────────┤
│  int, float, complex, bool                                      │
│  str                                                            │
│  tuple, frozenset                                               │
│  bytes                                                          │
│  None                                                           │
│                                                                 │
│  Properties:                                                    │
│  • Cannot change after creation                                 │
│  • Operations create NEW objects                                │
│  • Hashable (can be dict keys)                                  │
│  • Thread-safe                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     MUTABLE                                     │
├─────────────────────────────────────────────────────────────────┤
│  list                                                           │
│  dict                                                           │
│  set                                                            │
│  bytearray                                                      │
│  Custom objects (by default)                                    │
│                                                                 │
│  Properties:                                                    │
│  • Can be changed in-place                                      │
│  • Same object, different value                                 │
│  • NOT hashable                                                 │
│  • Beware shared references!                                    │
└─────────────────────────────────────────────────────────────────┘
```

### Mutability Gotchas

```python
# GOTCHA 1: Mutable default arguments
def add_item(item, lst=[]):  # WRONG!
    lst.append(item)
    return lst

add_item(1)  # [1]
add_item(2)  # [1, 2] - Same list!

# FIX:
def add_item(item, lst=None):
    if lst is None:
        lst = []
    lst.append(item)
    return lst

# GOTCHA 2: Aliasing
a = [1, 2, 3]
b = a           # b is ALIAS, not copy
b.append(4)
print(a)        # [1, 2, 3, 4] - a changed too!

# FIX:
b = a[:]        # Shallow copy
b = list(a)     # Shallow copy
b = a.copy()    # Shallow copy

# GOTCHA 3: Nested mutables in tuple
t = ([1, 2], [3, 4])
t[0].append(5)  # Works! Tuple is immutable, but contents aren't
print(t)        # ([1, 2, 5], [3, 4])

# GOTCHA 4: Multiplying lists
a = [[]] * 3    # [[refs to SAME list] * 3]
a[0].append(1)
print(a)        # [[1], [1], [1]] - All same!

# FIX:
a = [[] for _ in range(3)]  # Three DIFFERENT lists
```

---

## 5. FUNCTIONS & CLOSURES

### Functions Are First-Class Objects

```
┌─────────────────────────────────────────────────────────────────┐
│              Functions as First-Class Citizens                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Functions can be:                                              │
│  • Assigned to variables                                        │
│  • Passed as arguments                                          │
│  • Returned from functions                                      │
│  • Stored in data structures                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

def greet(name):
    return f"Hello, {name}"

# Assign to variable
say_hi = greet
say_hi("World")  # "Hello, World"

# Pass as argument
def apply(func, value):
    return func(value)

apply(greet, "Python")  # "Hello, Python"

# Return from function
def make_greeter(greeting):
    def greeter(name):
        return f"{greeting}, {name}"
    return greeter

spanish_greet = make_greeter("Hola")
spanish_greet("Mundo")  # "Hola, Mundo"

# Store in data structures
operations = {
    'add': lambda a, b: a + b,
    'sub': lambda a, b: a - b,
}
operations['add'](5, 3)  # 8
```

### Closure Mental Model

```
CLOSURE = Function + Captured Environment

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  def outer(x):                                                  │
│      def inner(y):         ← inner is a closure                 │
│          return x + y      ← x is "closed over" (captured)      │
│      return inner                                               │
│                                                                 │
│  add_5 = outer(5)          ← Returns inner with x=5 captured    │
│  add_5(3)                  → 8                                  │
│                                                                 │
│  Memory View:                                                   │
│  ┌────────────────────────────────────────────┐                │
│  │ add_5 function object                      │                │
│  │ ├── __code__: bytecode for inner           │                │
│  │ └── __closure__: (cell for x=5,)           │                │
│  └────────────────────────────────────────────┘                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

# Closure applications:
# 1. Data hiding / encapsulation
def counter():
    count = 0
    def increment():
        nonlocal count  # Required to modify outer variable
        count += 1
        return count
    return increment

c = counter()
c()  # 1
c()  # 2
c()  # 3

# 2. Factory functions
def power_factory(exponent):
    return lambda x: x ** exponent

square = power_factory(2)
cube = power_factory(3)
```

### Variable Scopes (LEGB)

```
┌─────────────────────────────────────────────────────────────────┐
│                    LEGB Rule                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  L - Local:     Inside current function                         │
│  E - Enclosing: Inside enclosing functions (closures)           │
│  G - Global:    Module-level                                    │
│  B - Built-in:  Python's built-in names                         │
│                                                                 │
│  Python searches in this order: L → E → G → B                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

x = 'global'        # Global scope

def outer():
    x = 'enclosing' # Enclosing scope

    def inner():
        x = 'local' # Local scope
        print(x)    # 'local' (found in L)

    inner()

# To MODIFY outer scopes:
def outer():
    x = 10

    def inner():
        nonlocal x  # Modify enclosing scope
        x = 20

    inner()
    print(x)  # 20

# Global modification:
count = 0

def increment():
    global count  # Declare intent to modify global
    count += 1
```

### Function Arguments Deep Dive

```python
# POSITIONAL AND KEYWORD
def func(a, b, c=10):
    pass

func(1, 2)           # a=1, b=2, c=10
func(1, 2, 3)        # a=1, b=2, c=3
func(1, c=5, b=2)    # a=1, b=2, c=5

# *args - Variable positional
def func(*args):     # args is a tuple
    for arg in args:
        print(arg)

func(1, 2, 3)        # Prints 1, 2, 3

# **kwargs - Variable keyword
def func(**kwargs):  # kwargs is a dict
    for k, v in kwargs.items():
        print(f"{k}={v}")

func(a=1, b=2)       # Prints a=1, b=2

# Combined (order matters!)
def func(pos, *args, kw_only, **kwargs):
    pass

# POSITIONAL-ONLY (Python 3.8+)
def func(x, y, /, z):  # x and y are positional-only
    pass

func(1, 2, 3)        # OK
func(1, 2, z=3)      # OK
func(x=1, y=2, z=3)  # ERROR! x and y must be positional

# KEYWORD-ONLY
def func(x, *, y, z):  # y and z are keyword-only
    pass

func(1, y=2, z=3)    # OK
func(1, 2, 3)        # ERROR! y and z must be keyword

# Unpacking in calls
args = [1, 2, 3]
kwargs = {'a': 1, 'b': 2}
func(*args)          # Unpack list as positional args
func(**kwargs)       # Unpack dict as keyword args
```

---

## 6. DECORATORS MENTAL MODEL

### Decorator Concept

```
DECORATOR = Function that modifies another function

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  @decorator                    ┌─────────────────────┐         │
│  def func():          ═══►     │ func = decorator(   │         │
│      pass                      │          func       │         │
│                                │        )            │         │
│                                └─────────────────────┘         │
│                                                                 │
│  The @ syntax is just syntactic sugar!                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Building Decorators

```python
# BASIC DECORATOR TEMPLATE
def my_decorator(func):
    def wrapper(*args, **kwargs):
        # Before function call
        print("Before")

        result = func(*args, **kwargs)  # Call original

        # After function call
        print("After")

        return result
    return wrapper

@my_decorator
def greet(name):
    print(f"Hello, {name}")

greet("World")
# Output:
# Before
# Hello, World
# After

# PRESERVING FUNCTION METADATA
from functools import wraps

def my_decorator(func):
    @wraps(func)  # Preserves __name__, __doc__, etc.
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper

# DECORATOR WITH ARGUMENTS
def repeat(times):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(times=3)
def greet(name):
    print(f"Hello, {name}")

# DECORATOR THAT WORKS WITH OR WITHOUT ARGUMENTS
def decorator(func=None, *, option=False):
    def actual_decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            if option:
                print("Option enabled")
            return func(*args, **kwargs)
        return wrapper

    if func is None:
        return actual_decorator
    return actual_decorator(func)

@decorator  # Works without ()
@decorator()  # Works with ()
@decorator(option=True)  # Works with arguments
```

### Common Decorator Patterns

```python
# 1. TIMING DECORATOR
import time
from functools import wraps

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        end = time.perf_counter()
        print(f"{func.__name__} took {end - start:.4f}s")
        return result
    return wrapper

# 2. CACHING / MEMOIZATION
def memoize(func):
    cache = {}
    @wraps(func)
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    return wrapper

# Or use built-in:
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

# 3. RETRY DECORATOR
def retry(max_attempts=3, exceptions=(Exception,)):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts - 1:
                        raise
                    print(f"Attempt {attempt + 1} failed: {e}")
        return wrapper
    return decorator

# 4. AUTHENTICATION DECORATOR
def require_auth(func):
    @wraps(func)
    def wrapper(request, *args, **kwargs):
        if not request.user.is_authenticated:
            raise PermissionError("Authentication required")
        return func(request, *args, **kwargs)
    return wrapper

# 5. CLASS DECORATOR
def singleton(cls):
    instances = {}
    @wraps(cls)
    def wrapper(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return wrapper

@singleton
class Database:
    pass
```

### Decorator Execution Order

```python
@decorator_a
@decorator_b
@decorator_c
def func():
    pass

# Equivalent to:
func = decorator_a(decorator_b(decorator_c(func)))

# Execution order:
# 1. decorator_c wraps func
# 2. decorator_b wraps result of step 1
# 3. decorator_a wraps result of step 2

# When func() is called:
# decorator_a's wrapper runs FIRST (outermost)
# then decorator_b's wrapper
# then decorator_c's wrapper
# then original func
# then back up through wrappers
```

---

## 7. OBJECT-ORIENTED PYTHON

### Class Anatomy

```python
class Dog:
    # CLASS ATTRIBUTE (shared by all instances)
    species = "Canis familiaris"

    # INITIALIZER (not constructor!)
    def __init__(self, name, age):
        # INSTANCE ATTRIBUTES
        self.name = name
        self.age = age
        self._protected = "convention only"  # Single underscore
        self.__private = "name mangled"      # Double underscore

    # INSTANCE METHOD
    def bark(self):
        return f"{self.name} says woof!"

    # CLASS METHOD (receives class, not instance)
    @classmethod
    def from_birth_year(cls, name, year):
        age = 2024 - year
        return cls(name, age)  # Creates instance of cls

    # STATIC METHOD (no implicit first argument)
    @staticmethod
    def is_adult(age):
        return age >= 2

    # PROPERTY (computed attribute)
    @property
    def human_age(self):
        return self.age * 7

    @human_age.setter
    def human_age(self, value):
        self.age = value // 7

    # REPRESENTATION
    def __repr__(self):
        return f"Dog(name={self.name!r}, age={self.age})"

    def __str__(self):
        return f"{self.name} is {self.age} years old"
```

### Inheritance & MRO

```
┌─────────────────────────────────────────────────────────────────┐
│                Method Resolution Order (MRO)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  class A: pass                                                  │
│  class B(A): pass                                               │
│  class C(A): pass                                               │
│  class D(B, C): pass  # Multiple inheritance                    │
│                                                                 │
│            A                                                    │
│           / \                                                   │
│          B   C                                                  │
│           \ /                                                   │
│            D                                                    │
│                                                                 │
│  D.__mro__: (D, B, C, A, object)                               │
│                                                                 │
│  C3 Linearization ensures:                                      │
│  1. Children come before parents                                │
│  2. Order of parents preserved                                  │
│  3. No class appears twice                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

# Using super() follows MRO
class B(A):
    def method(self):
        super().method()  # Calls next in MRO, not necessarily A

# Cooperative multiple inheritance
class A:
    def __init__(self, **kwargs):
        super().__init__(**kwargs)

class B(A):
    def __init__(self, x, **kwargs):
        self.x = x
        super().__init__(**kwargs)

class C(A):
    def __init__(self, y, **kwargs):
        self.y = y
        super().__init__(**kwargs)

class D(B, C):
    def __init__(self, x, y, z, **kwargs):
        self.z = z
        super().__init__(x=x, y=y, **kwargs)
```

### Dunder (Magic) Methods

```python
class Vector:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    # STRING REPRESENTATION
    def __repr__(self):  # For developers (unambiguous)
        return f"Vector({self.x}, {self.y})"

    def __str__(self):   # For users (readable)
        return f"({self.x}, {self.y})"

    # COMPARISON
    def __eq__(self, other):
        return self.x == other.x and self.y == other.y

    def __lt__(self, other):
        return (self.x, self.y) < (other.x, other.y)

    # Define __lt__ and use @functools.total_ordering for others

    # ARITHMETIC
    def __add__(self, other):
        return Vector(self.x + other.x, self.y + other.y)

    def __mul__(self, scalar):
        return Vector(self.x * scalar, self.y * scalar)

    def __rmul__(self, scalar):  # Right multiply: 3 * vector
        return self.__mul__(scalar)

    # CONTAINER BEHAVIOR
    def __len__(self):
        return 2

    def __getitem__(self, index):
        if index == 0: return self.x
        if index == 1: return self.y
        raise IndexError

    def __iter__(self):
        yield self.x
        yield self.y

    # CALLABLE
    def __call__(self):
        return (self.x ** 2 + self.y ** 2) ** 0.5

    # CONTEXT MANAGER
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        pass  # Return True to suppress exception

    # ATTRIBUTE ACCESS
    def __getattr__(self, name):       # Called when attr not found
        raise AttributeError(f"No '{name}'")

    def __setattr__(self, name, value): # Called on every assignment
        super().__setattr__(name, value)

    # HASHING (for dict keys, set membership)
    def __hash__(self):
        return hash((self.x, self.y))
```

### Data Classes (Python 3.7+)

```python
from dataclasses import dataclass, field
from typing import List

@dataclass
class Point:
    x: float
    y: float

# Automatically generates:
# __init__, __repr__, __eq__

@dataclass(frozen=True)  # Immutable (hashable)
class FrozenPoint:
    x: float
    y: float

@dataclass(order=True)   # Generates comparison methods
class SortablePoint:
    x: float
    y: float

@dataclass
class Config:
    name: str
    values: List[int] = field(default_factory=list)  # Mutable default
    _cache: dict = field(default_factory=dict, repr=False)  # Exclude from repr

    def __post_init__(self):  # Called after __init__
        self.name = self.name.upper()
```

### Abstract Base Classes

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self):
        """Calculate area"""
        pass

    @abstractmethod
    def perimeter(self):
        """Calculate perimeter"""
        pass

    # Concrete method - can have implementation
    def description(self):
        return f"A shape with area {self.area()}"

class Rectangle(Shape):
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def area(self):
        return self.width * self.height

    def perimeter(self):
        return 2 * (self.width + self.height)

# shape = Shape()  # TypeError: Can't instantiate abstract class
rect = Rectangle(5, 3)  # OK
```

---

## 8. ITERATORS & GENERATORS

### Iterator Protocol

```
┌─────────────────────────────────────────────────────────────────┐
│                    Iterator Protocol                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ITERABLE: Object with __iter__() method                        │
│            Returns an iterator                                  │
│                                                                 │
│  ITERATOR: Object with __next__() method                        │
│            Returns next value or raises StopIteration           │
│                                                                 │
│  for item in iterable:                                          │
│      process(item)                                              │
│                                                                 │
│  Is equivalent to:                                              │
│                                                                 │
│  iterator = iter(iterable)    # Calls __iter__()                │
│  while True:                                                    │
│      try:                                                       │
│          item = next(iterator)  # Calls __next__()              │
│          process(item)                                          │
│      except StopIteration:                                      │
│          break                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Custom Iterator

```python
class CountDown:
    def __init__(self, start):
        self.start = start

    def __iter__(self):
        return CountDownIterator(self.start)

class CountDownIterator:
    def __init__(self, start):
        self.current = start

    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

# Usage
for num in CountDown(5):
    print(num)  # 5, 4, 3, 2, 1

# Simpler: combine iterable and iterator
class CountDown:
    def __init__(self, start):
        self.current = start

    def __iter__(self):
        return self

    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1
```

### Generator Functions

```python
# GENERATOR FUNCTION: Uses yield
def countdown(n):
    while n > 0:
        yield n      # Pause and return value
        n -= 1       # Resume here on next()

gen = countdown(3)
next(gen)  # 3
next(gen)  # 2
next(gen)  # 1
next(gen)  # StopIteration

# Memory advantage:
# List:      [0, 1, 2, ..., 999999]  # All in memory
# Generator: (x for x in range(1000000))  # One at a time

# GENERATOR EXPRESSION (like list comp, but lazy)
squares_list = [x**2 for x in range(1000)]    # Creates list immediately
squares_gen = (x**2 for x in range(1000))     # Creates generator

# YIELD FROM (delegate to sub-generator)
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)  # Delegate
        else:
            yield item

list(flatten([1, [2, 3, [4, 5]], 6]))  # [1, 2, 3, 4, 5, 6]

# SEND VALUES TO GENERATOR
def accumulator():
    total = 0
    while True:
        value = yield total  # Receive value, send total
        if value is None:
            break
        total += value

acc = accumulator()
next(acc)        # Initialize, returns 0
acc.send(10)     # Returns 10
acc.send(20)     # Returns 30
acc.send(5)      # Returns 35
```

### itertools Module

```python
from itertools import (
    count, cycle, repeat,           # Infinite iterators
    chain, compress, dropwhile,     # Terminating iterators
    takewhile, groupby, islice,
    accumulate, product, permutations,
    combinations, combinations_with_replacement
)

# INFINITE ITERATORS
count(10, 2)      # 10, 12, 14, 16, ...
cycle([1, 2, 3])  # 1, 2, 3, 1, 2, 3, ...
repeat('A', 3)    # 'A', 'A', 'A'

# CHAIN - Combine iterables
list(chain([1, 2], [3, 4]))  # [1, 2, 3, 4]
list(chain.from_iterable([[1, 2], [3, 4]]))  # Same

# ISLICE - Slice an iterator
list(islice(count(), 5))  # [0, 1, 2, 3, 4]

# GROUPBY - Group consecutive elements
data = [('a', 1), ('a', 2), ('b', 3)]
for key, group in groupby(data, key=lambda x: x[0]):
    print(key, list(group))
# a [('a', 1), ('a', 2)]
# b [('b', 3)]

# COMBINATIONS & PERMUTATIONS
list(combinations('ABC', 2))   # [('A','B'), ('A','C'), ('B','C')]
list(permutations('ABC', 2))   # [('A','B'), ('A','C'), ('B','A'), ...]

# PRODUCT - Cartesian product
list(product([1, 2], ['a', 'b']))  # [(1,'a'), (1,'b'), (2,'a'), (2,'b')]

# ACCUMULATE - Running totals
list(accumulate([1, 2, 3, 4]))  # [1, 3, 6, 10]
```

---

## 9. CONTEXT MANAGERS

### Context Manager Protocol

```
┌─────────────────────────────────────────────────────────────────┐
│                  Context Manager Protocol                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  with expression as variable:                                   │
│      # body                                                     │
│                                                                 │
│  Is equivalent to:                                              │
│                                                                 │
│  manager = expression                                           │
│  variable = manager.__enter__()                                 │
│  try:                                                           │
│      # body                                                     │
│  except:                                                        │
│      if not manager.__exit__(*sys.exc_info()):                 │
│          raise                                                  │
│  else:                                                          │
│      manager.__exit__(None, None, None)                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Custom Context Manager

```python
# CLASS-BASED
class FileManager:
    def __init__(self, filename, mode):
        self.filename = filename
        self.mode = mode
        self.file = None

    def __enter__(self):
        self.file = open(self.filename, self.mode)
        return self.file  # This becomes 'as' variable

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.file.close()
        # Return True to suppress exception
        # Return False/None to propagate exception
        return False

with FileManager('test.txt', 'w') as f:
    f.write('hello')

# GENERATOR-BASED (using contextlib)
from contextlib import contextmanager

@contextmanager
def file_manager(filename, mode):
    f = open(filename, mode)
    try:
        yield f        # Everything before yield = __enter__
    finally:
        f.close()      # Everything after yield = __exit__

# COMMON PATTERNS
@contextmanager
def timer(label):
    start = time.time()
    try:
        yield
    finally:
        print(f"{label}: {time.time() - start:.4f}s")

with timer("Operation"):
    # Some operation
    pass

@contextmanager
def temporary_change(obj, attr, value):
    original = getattr(obj, attr)
    setattr(obj, attr, value)
    try:
        yield
    finally:
        setattr(obj, attr, original)
```

### contextlib Utilities

```python
from contextlib import (
    suppress,
    redirect_stdout,
    ExitStack,
    closing,
    nullcontext
)

# SUPPRESS - Ignore specific exceptions
with suppress(FileNotFoundError):
    os.remove('nonexistent.txt')  # No error raised

# REDIRECT_STDOUT
import io
f = io.StringIO()
with redirect_stdout(f):
    print("Hello")
output = f.getvalue()  # "Hello\n"

# EXITSTACK - Manage dynamic number of context managers
with ExitStack() as stack:
    files = [stack.enter_context(open(f)) for f in filenames]
    # All files closed when block exits

# CLOSING - Call .close() on exit
from urllib.request import urlopen
with closing(urlopen('http://example.com')) as page:
    content = page.read()

# NULLCONTEXT - No-op context manager
cm = nullcontext() if condition else real_context_manager()
with cm:
    pass
```

---

## 10. EXCEPTION HANDLING

### Exception Hierarchy

```
BaseException
├── SystemExit
├── KeyboardInterrupt
├── GeneratorExit
└── Exception
    ├── StopIteration
    ├── ArithmeticError
    │   ├── ZeroDivisionError
    │   └── OverflowError
    ├── AttributeError
    ├── EOFError
    ├── ImportError
    │   └── ModuleNotFoundError
    ├── LookupError
    │   ├── IndexError
    │   └── KeyError
    ├── NameError
    ├── OSError
    │   ├── FileNotFoundError
    │   ├── PermissionError
    │   └── TimeoutError
    ├── TypeError
    ├── ValueError
    │   └── UnicodeError
    └── RuntimeError
        └── RecursionError
```

### Exception Handling Patterns

```python
# BASIC TRY/EXCEPT
try:
    result = risky_operation()
except SomeException:
    handle_error()

# MULTIPLE EXCEPTIONS
try:
    result = risky_operation()
except (TypeError, ValueError) as e:
    print(f"Error: {e}")
except KeyError:
    print("Key not found")
except Exception as e:
    print(f"Unexpected: {e}")
    raise  # Re-raise same exception

# ELSE AND FINALLY
try:
    result = operation()
except SomeError:
    handle_error()
else:
    # Runs if NO exception occurred
    process_result(result)
finally:
    # ALWAYS runs (cleanup)
    cleanup()

# CUSTOM EXCEPTIONS
class ValidationError(Exception):
    def __init__(self, message, field=None):
        super().__init__(message)
        self.field = field

try:
    raise ValidationError("Invalid email", field="email")
except ValidationError as e:
    print(f"Field {e.field}: {e}")

# EXCEPTION CHAINING
try:
    data = load_data()
except FileNotFoundError as e:
    raise ConfigError("Config file missing") from e
    # Shows: "The above exception was the direct cause..."

# SUPPRESS CONTEXT (hide original)
raise ConfigError("Failed") from None
```

### Best Practices

```python
# ✓ DO: Be specific with exceptions
try:
    value = data['key']
except KeyError:
    value = default

# ✗ DON'T: Bare except
try:
    value = data['key']
except:  # Catches EVERYTHING including KeyboardInterrupt!
    value = default

# ✓ DO: Use EAFP (Easier to Ask Forgiveness than Permission)
try:
    return data['key']
except KeyError:
    return default

# ✗ DON'T: Use LBYL when EAFP is cleaner
if 'key' in data:  # Race condition possible
    return data['key']
else:
    return default

# ✓ DO: Re-raise with context
except SomeError as e:
    logger.error(f"Failed: {e}")
    raise  # Preserves traceback

# ✗ DON'T: Lose traceback
except SomeError as e:
    raise SomeError(str(e))  # Loses original traceback

# ✓ DO: Cleanup in finally
f = open('file.txt')
try:
    process(f)
finally:
    f.close()

# Better: Use context manager
with open('file.txt') as f:
    process(f)
```

---

## 11. CONCURRENCY & PARALLELISM

### Concurrency vs Parallelism

```
┌─────────────────────────────────────────────────────────────────┐
│           CONCURRENCY vs PARALLELISM                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CONCURRENCY: Managing multiple tasks (may not run together)    │
│  PARALLELISM: Executing multiple tasks simultaneously           │
│                                                                 │
│  Concurrent (single core):        Parallel (multi-core):        │
│  ┌──────────────────────┐         ┌─────────┐ ┌─────────┐      │
│  │ Task A ██░░██░░██    │         │ Task A  │ │ Task B  │      │
│  │ Task B ░░██░░██░░    │         │ ████████│ │ ████████│      │
│  └──────────────────────┘         └─────────┘ └─────────┘      │
│   (interleaved)                    (truly simultaneous)         │
│                                                                 │
│  Python Options:                                                │
│  • Threading: Concurrency (GIL limits parallelism)              │
│  • Multiprocessing: True parallelism (separate processes)       │
│  • Asyncio: Concurrency (single-threaded, event loop)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Threading

```python
import threading
from concurrent.futures import ThreadPoolExecutor

# BASIC THREADING
def worker(name):
    print(f"Worker {name} starting")
    time.sleep(1)
    print(f"Worker {name} done")

# Create and start threads
threads = []
for i in range(3):
    t = threading.Thread(target=worker, args=(i,))
    t.start()
    threads.append(t)

# Wait for completion
for t in threads:
    t.join()

# THREAD POOL (preferred)
with ThreadPoolExecutor(max_workers=3) as executor:
    # Submit individual tasks
    future = executor.submit(worker, "A")
    result = future.result()  # Blocks until done

    # Map function over iterables
    results = executor.map(worker, ["A", "B", "C"])

# THREAD SYNCHRONIZATION
lock = threading.Lock()
counter = 0

def increment():
    global counter
    with lock:  # Only one thread at a time
        counter += 1

# Other synchronization primitives:
# RLock - Reentrant lock (same thread can acquire multiple times)
# Semaphore - Allow N threads
# Event - Signal between threads
# Condition - Wait for condition
# Barrier - Wait for all threads
```

### Multiprocessing

```python
import multiprocessing as mp
from concurrent.futures import ProcessPoolExecutor

# BASIC MULTIPROCESSING
def cpu_intensive(n):
    return sum(i * i for i in range(n))

# Create processes
if __name__ == '__main__':  # Required on Windows
    processes = []
    for i in range(4):
        p = mp.Process(target=cpu_intensive, args=(10**6,))
        p.start()
        processes.append(p)

    for p in processes:
        p.join()

# PROCESS POOL (preferred)
with ProcessPoolExecutor(max_workers=4) as executor:
    results = executor.map(cpu_intensive, [10**6] * 4)

# SHARING DATA BETWEEN PROCESSES
# Shared memory
shared_value = mp.Value('i', 0)  # Shared integer
shared_array = mp.Array('d', [0.0] * 10)  # Shared array

# Queue for message passing
queue = mp.Queue()
queue.put("message")
message = queue.get()

# Pipe for two-way communication
parent_conn, child_conn = mp.Pipe()
```

### Asyncio

```python
import asyncio

# ASYNC FUNCTION (coroutine)
async def fetch_data(url):
    print(f"Fetching {url}")
    await asyncio.sleep(1)  # Non-blocking sleep
    return f"Data from {url}"

# RUN SINGLE COROUTINE
result = asyncio.run(fetch_data("http://example.com"))

# RUN MULTIPLE COROUTINES CONCURRENTLY
async def main():
    # Create tasks
    tasks = [
        asyncio.create_task(fetch_data("url1")),
        asyncio.create_task(fetch_data("url2")),
        asyncio.create_task(fetch_data("url3")),
    ]

    # Wait for all
    results = await asyncio.gather(*tasks)
    return results

# AWAIT VS CREATE_TASK
async def sequential():
    # Sequential - total time = 3 seconds
    a = await fetch_data("url1")  # Wait 1s
    b = await fetch_data("url2")  # Wait 1s
    c = await fetch_data("url3")  # Wait 1s

async def concurrent():
    # Concurrent - total time = 1 second
    a, b, c = await asyncio.gather(
        fetch_data("url1"),
        fetch_data("url2"),
        fetch_data("url3"),
    )

# ASYNC CONTEXT MANAGER
class AsyncResource:
    async def __aenter__(self):
        await self.connect()
        return self

    async def __aexit__(self, *args):
        await self.disconnect()

async with AsyncResource() as resource:
    await resource.do_something()

# ASYNC ITERATOR
class AsyncCounter:
    def __init__(self, stop):
        self.stop = stop

    def __aiter__(self):
        self.current = 0
        return self

    async def __anext__(self):
        if self.current >= self.stop:
            raise StopAsyncIteration
        await asyncio.sleep(0.1)
        self.current += 1
        return self.current

async for num in AsyncCounter(5):
    print(num)

# ASYNC GENERATOR
async def async_range(stop):
    for i in range(stop):
        await asyncio.sleep(0.1)
        yield i
```

### When to Use What

```
┌─────────────────────────────────────────────────────────────────┐
│               CONCURRENCY DECISION GUIDE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Task Type          │ Best Choice        │ Why                  │
│  ──────────────────────────────────────────────────────────────│
│  I/O-bound          │ asyncio            │ Efficient, scalable  │
│  (network, files)   │ or threading       │ GIL released during  │
│                     │                    │ I/O waits            │
│                     │                    │                      │
│  CPU-bound          │ multiprocessing    │ True parallelism,    │
│  (computation)      │                    │ bypasses GIL         │
│                     │                    │                      │
│  Mixed I/O + CPU    │ multiprocessing    │ Each process can     │
│                     │ + asyncio          │ run event loop       │
│                     │                    │                      │
│  Simple scripts     │ threading          │ Easy to understand   │
│                     │                    │ and implement        │
│                     │                    │                      │
│  High concurrency   │ asyncio            │ Handles thousands    │
│  (10000+ tasks)     │                    │ of connections       │
│                     │                    │                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 12. PYTHON INTERNALS

### The GIL (Global Interpreter Lock)

```
┌─────────────────────────────────────────────────────────────────┐
│                Global Interpreter Lock (GIL)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  • Mutex that protects Python objects                           │
│  • Only ONE thread executes Python bytecode at a time           │
│  • Released during I/O operations                               │
│                                                                 │
│  Thread 1: ████████░░░░░░░░████████░░░░░░░░                    │
│  Thread 2: ░░░░░░░░████████░░░░░░░░████████                    │
│            └─ Holding GIL ─┘                                    │
│                                                                 │
│  IMPLICATIONS:                                                  │
│  • CPU-bound threads don't run in parallel                      │
│  • I/O-bound threads CAN run concurrently (GIL released)        │
│  • Use multiprocessing for CPU parallelism                      │
│                                                                 │
│  WHY IT EXISTS:                                                 │
│  • Simplifies memory management (reference counting)            │
│  • Makes C extensions easier to write                           │
│  • Historical design decision                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Python Bytecode

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)
# Output:
#   2           0 LOAD_FAST                0 (a)
#               2 LOAD_FAST                1 (b)
#               4 BINARY_ADD
#               6 RETURN_VALUE

# Bytecode is:
# 1. Compiled from source (.py → .pyc)
# 2. Executed by Python Virtual Machine (PVM)
# 3. Cached in __pycache__ directory

# Access bytecode
add.__code__.co_code        # Raw bytecode bytes
add.__code__.co_varnames    # ('a', 'b')
add.__code__.co_consts      # (None,)
```

### Object Internals

```python
# Every Python object has:
# 1. Reference count
# 2. Type pointer
# 3. Value

import sys

x = [1, 2, 3]
sys.getsizeof(x)       # Size in bytes (doesn't include referenced objects)

# Object's __dict__ stores instance attributes
class MyClass:
    def __init__(self):
        self.x = 1
        self.y = 2

obj = MyClass()
obj.__dict__  # {'x': 1, 'y': 2}

# __slots__ - Avoid __dict__ for memory savings
class Point:
    __slots__ = ['x', 'y']

    def __init__(self, x, y):
        self.x = x
        self.y = y

# No __dict__, fixed attributes, less memory
```

---

## 13. PYTHONIC PATTERNS

### Comprehensions

```python
# LIST COMPREHENSION
squares = [x**2 for x in range(10)]
evens = [x for x in range(10) if x % 2 == 0]
matrix = [[i*j for j in range(3)] for i in range(3)]

# DICT COMPREHENSION
word_lengths = {word: len(word) for word in words}
inverted = {v: k for k, v in original.items()}

# SET COMPREHENSION
unique_lengths = {len(word) for word in words}

# GENERATOR EXPRESSION (lazy)
sum_squares = sum(x**2 for x in range(1000000))  # Memory efficient

# NESTED WITH CONDITION
# Flatten and filter
flat = [x for row in matrix for x in row if x > 0]
# Equivalent to:
# for row in matrix:
#     for x in row:
#         if x > 0:
#             flat.append(x)
```

### Unpacking

```python
# BASIC UNPACKING
a, b, c = [1, 2, 3]
x, y = y, x  # Swap

# STAR UNPACKING
first, *rest = [1, 2, 3, 4]        # first=1, rest=[2,3,4]
first, *middle, last = [1,2,3,4]   # first=1, middle=[2,3], last=4
*start, last = [1, 2, 3, 4]        # start=[1,2,3], last=4

# NESTED UNPACKING
data = ('John', (25, 'Engineer'))
name, (age, job) = data

# IGNORE VALUES
_, b, _ = [1, 2, 3]  # Only need b
a, *_ = [1, 2, 3, 4]  # Only need first

# FUNCTION ARGUMENT UNPACKING
def func(a, b, c):
    pass

args = [1, 2, 3]
func(*args)  # Unpack list

kwargs = {'a': 1, 'b': 2, 'c': 3}
func(**kwargs)  # Unpack dict

# MERGE DICTS (Python 3.9+)
merged = {**dict1, **dict2}
# Or: merged = dict1 | dict2
```

### Pythonic Idioms

```python
# ENUMERATE (instead of range(len(...)))
for i, item in enumerate(items):
    print(f"{i}: {item}")

for i, item in enumerate(items, start=1):  # Start from 1
    pass

# ZIP (iterate multiple sequences)
for name, age in zip(names, ages):
    print(f"{name} is {age}")

# zip_longest for unequal lengths
from itertools import zip_longest
for a, b in zip_longest(short, long, fillvalue=None):
    pass

# ANY / ALL
if any(x > 10 for x in numbers):  # At least one
    pass

if all(x > 0 for x in numbers):  # All items
    pass

# TERNARY EXPRESSION
result = value if condition else default

# CHAINED COMPARISON
if 0 < x < 10:  # Same as: 0 < x and x < 10
    pass

# TRUTHINESS
if items:  # Instead of: if len(items) > 0
    pass

if not items:  # Instead of: if len(items) == 0
    pass

# DEFAULT DICT VALUE
value = d.get('key', default)  # Instead of: d['key'] if 'key' in d else default

# SETDEFAULT
graph = {}
graph.setdefault(node, []).append(neighbor)

# OR use defaultdict
from collections import defaultdict
graph = defaultdict(list)
graph[node].append(neighbor)

# WITH STATEMENT FOR RESOURCES
with open('file.txt') as f:
    content = f.read()

# CONTEXT MANAGER FOR MULTIPLE
with open('in.txt') as f_in, open('out.txt', 'w') as f_out:
    f_out.write(f_in.read())
```

### f-strings (Formatted String Literals)

```python
name = "Alice"
age = 30

# Basic interpolation
f"Hello, {name}!"

# Expressions
f"Next year: {age + 1}"

# Format specifiers
pi = 3.14159
f"{pi:.2f}"          # "3.14" (2 decimal places)
f"{1000000:,}"       # "1,000,000" (comma separator)
f"{42:08d}"          # "00000042" (zero-padded)
f"{0.5:.1%}"         # "50.0%" (percentage)

# Alignment
f"{name:>10}"        # "     Alice" (right align)
f"{name:<10}"        # "Alice     " (left align)
f"{name:^10}"        # "  Alice   " (center)

# Debug (Python 3.8+)
x = 42
f"{x=}"              # "x=42"
f"{x = }"            # "x = 42"
f"{x=:.2f}"          # "x=42.00"

# Multiline
message = f"""
Name: {name}
Age: {age}
"""

# Raw f-string (for regex, paths)
pattern = rf"\b{word}\b"
```

---

## 14. TYPE HINTS & STATIC TYPING

### Basic Type Hints

```python
from typing import (
    List, Dict, Set, Tuple, Optional, Union,
    Callable, Any, TypeVar, Generic,
    Literal, Final, TypedDict
)

# BASIC TYPES
def greet(name: str) -> str:
    return f"Hello, {name}"

def add(a: int, b: int) -> int:
    return a + b

# COLLECTIONS (Python 3.9+ can use lowercase)
def process(items: list[int]) -> dict[str, int]:
    pass

# Pre-3.9
def process(items: List[int]) -> Dict[str, int]:
    pass

# OPTIONAL (None allowed)
def find(key: str) -> Optional[int]:  # Same as: Union[int, None]
    return None

# UNION (multiple types)
def process(value: Union[int, str]) -> None:
    pass

# Python 3.10+
def process(value: int | str) -> None:
    pass

# CALLABLE
def apply(func: Callable[[int, int], int], a: int, b: int) -> int:
    return func(a, b)

# ANY (escape hatch)
def process(data: Any) -> Any:
    pass
```

### Advanced Type Hints

```python
# TYPEVAR (generics)
T = TypeVar('T')

def first(items: list[T]) -> T:
    return items[0]

# Bounded TypeVar
Number = TypeVar('Number', int, float)

def double(x: Number) -> Number:
    return x * 2

# GENERIC CLASSES
class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

stack: Stack[int] = Stack()

# LITERAL (specific values)
def set_mode(mode: Literal["read", "write"]) -> None:
    pass

# FINAL (constant)
MAX_SIZE: Final = 100
MAX_SIZE = 200  # Type checker error

# TYPEDDICT
class Movie(TypedDict):
    title: str
    year: int
    rating: float

movie: Movie = {"title": "Inception", "year": 2010, "rating": 8.8}

# PROTOCOL (structural typing)
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> None: ...

def render(item: Drawable) -> None:
    item.draw()

# Any class with draw() method works, no inheritance needed
```

### Type Checking Tools

```python
# MYPY - Static type checker
# pip install mypy
# mypy script.py

# Common mypy flags:
# --strict           Enable all strict checks
# --ignore-missing-imports  Ignore missing stubs
# --disallow-untyped-defs   Require type hints

# TYPE IGNORE COMMENTS
x = some_untyped_func()  # type: ignore

# REVEAL TYPE (debugging)
reveal_type(variable)  # mypy shows inferred type

# CAST (tell type checker)
from typing import cast
x = cast(int, some_value)  # Trust me, it's an int

# OVERLOAD (multiple signatures)
from typing import overload

@overload
def process(x: int) -> int: ...
@overload
def process(x: str) -> str: ...

def process(x):
    return x
```

---

## 15. COMMON GOTCHAS

### Mutable Default Arguments

```python
# ✗ WRONG
def add_item(item, items=[]):
    items.append(item)
    return items

add_item(1)  # [1]
add_item(2)  # [1, 2] - Same list!

# ✓ CORRECT
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

### Late Binding Closures

```python
# ✗ WRONG
funcs = []
for i in range(3):
    funcs.append(lambda: i)

[f() for f in funcs]  # [2, 2, 2] - All same!

# ✓ CORRECT - Capture value
funcs = []
for i in range(3):
    funcs.append(lambda i=i: i)  # Default arg captures current value

[f() for f in funcs]  # [0, 1, 2]
```

### Integer Caching

```python
# Small integers are cached
a = 256
b = 256
a is b  # True

a = 257
b = 257
a is b  # False (or True in some contexts)

# ALWAYS use == for value comparison
a == b  # True (always correct)
```

### String Interning

```python
a = "hello"
b = "hello"
a is b  # True (interned)

a = "hello world"
b = "hello world"
a is b  # Might be False

# Don't rely on identity for strings!
a == b  # Always correct
```

### Chained Assignment

```python
# ✗ WRONG assumption
a = b = []
a.append(1)
print(b)  # [1] - Same list!

# ✓ CORRECT
a = []
b = []
# Or: a, b = [], []
```

### Modifying List While Iterating

```python
# ✗ WRONG
items = [1, 2, 3, 4, 5]
for item in items:
    if item % 2 == 0:
        items.remove(item)
# Result: [1, 3, 5]? No! [1, 3, 5] skips some items

# ✓ CORRECT - Iterate over copy
for item in items[:]:
    if item % 2 == 0:
        items.remove(item)

# ✓ BETTER - List comprehension
items = [x for x in items if x % 2 != 0]
```

### Class vs Instance Attributes

```python
class Counter:
    count = 0  # Class attribute (shared!)

c1 = Counter()
c2 = Counter()
c1.count += 1
print(c2.count)  # Still 0! (c1.count is now instance attr)

Counter.count += 1
print(c2.count)  # 1 (class attr changed)

# ✓ CORRECT - Use instance attributes
class Counter:
    def __init__(self):
        self.count = 0
```

### Import Cycles

```python
# module_a.py
from module_b import func_b  # Imports module_b

def func_a():
    return func_b()

# module_b.py
from module_a import func_a  # Circular import!

def func_b():
    return func_a()

# ✓ FIX - Import inside function, or restructure
def func_b():
    from module_a import func_a
    return func_a()
```

---

## 16. PERFORMANCE OPTIMIZATION

### Profiling First

```python
# TIMING
import time

start = time.perf_counter()
# ... code ...
elapsed = time.perf_counter() - start

# CPROFILE
import cProfile
cProfile.run('my_function()')

# Or from command line:
# python -m cProfile -s cumtime script.py

# LINE PROFILER (pip install line_profiler)
@profile
def my_function():
    pass
# kernprof -l -v script.py

# MEMORY PROFILER (pip install memory_profiler)
from memory_profiler import profile

@profile
def my_function():
    pass
```

### Common Optimizations

```python
# 1. USE BUILT-INS (C-implemented)
sum(items)              # Faster than manual loop
max(items), min(items)
''.join(strings)        # Faster than += concatenation
any(), all()            # Short-circuit

# 2. LIST COMPREHENSION VS LOOP
# Faster:
squares = [x**2 for x in range(1000)]
# Slower:
squares = []
for x in range(1000):
    squares.append(x**2)

# 3. LOCAL VARIABLES FASTER THAN GLOBAL
def process():
    local_func = global_func  # Cache lookup
    for item in items:
        local_func(item)

# 4. USE GENERATORS FOR LARGE DATA
# Memory efficient:
total = sum(x**2 for x in range(10**8))
# Memory hog:
total = sum([x**2 for x in range(10**8)])

# 5. SET FOR MEMBERSHIP TESTING
items_set = set(items)  # O(1) lookup
if x in items_set:      # vs O(n) for list
    pass

# 6. DICT.GET() INSTEAD OF TRY/EXCEPT
# Faster:
value = d.get(key, default)
# Slower for missing keys:
try:
    value = d[key]
except KeyError:
    value = default

# 7. STRING CONCATENATION
# Fast:
result = ''.join(parts)
# Slow:
result = ''
for part in parts:
    result += part  # Creates new string each time

# 8. USE __SLOTS__ FOR MANY OBJECTS
class Point:
    __slots__ = ['x', 'y']  # No __dict__, saves memory

# 9. AVOID REPEATED ATTRIBUTE LOOKUPS
# Faster:
append = items.append
for x in data:
    append(x)
# Slower:
for x in data:
    items.append(x)
```

### NumPy for Numerical Work

```python
import numpy as np

# NumPy operations are vectorized (C-level loops)
arr = np.array([1, 2, 3, 4, 5])
squared = arr ** 2  # Much faster than list comprehension

# Broadcasting
a = np.array([1, 2, 3])
b = np.array([[1], [2], [3]])
c = a + b  # 3x3 result

# Use NumPy for numerical computations
# Use lists for general-purpose collections
```

---

## 17. QUICK REFERENCE CARD

### Essential Built-ins

```python
# TYPE CONVERSION
int('42')           # String to int
float('3.14')       # String to float
str(42)             # To string
list('abc')         # To list: ['a', 'b', 'c']
tuple([1,2,3])      # To tuple
set([1,2,2,3])      # To set: {1, 2, 3}
dict([('a',1)])     # To dict: {'a': 1}
bool(0)             # To bool: False

# USEFUL FUNCTIONS
len(obj)            # Length
range(start, stop, step)
enumerate(iterable, start=0)
zip(*iterables)
map(func, iterable)
filter(func, iterable)
sorted(iterable, key=None, reverse=False)
reversed(sequence)
sum(iterable, start=0)
min(iterable), max(iterable)
abs(x)
round(x, ndigits)
pow(base, exp, mod=None)
divmod(a, b)        # (a // b, a % b)

# STRING METHODS
s.strip()           # Remove whitespace
s.split(sep)        # Split to list
s.join(iterable)    # Join with separator
s.replace(old, new)
s.find(sub)         # Index or -1
s.index(sub)        # Index or ValueError
s.startswith(prefix)
s.endswith(suffix)
s.upper(), s.lower()
s.isdigit(), s.isalpha()
s.format(*args, **kwargs)
f"{var}"            # f-string

# LIST METHODS
lst.append(x)       # Add to end
lst.extend(iterable)
lst.insert(i, x)
lst.pop(i=-1)       # Remove & return
lst.remove(x)       # Remove first occurrence
lst.sort()          # In-place sort
lst.reverse()       # In-place reverse
lst.copy()          # Shallow copy
lst.clear()         # Remove all

# DICT METHODS
d.get(key, default)
d.setdefault(key, default)
d.pop(key, default)
d.update(other)
d.keys(), d.values(), d.items()

# FILE I/O
with open('file.txt', 'r') as f:
    content = f.read()          # Entire file
    lines = f.readlines()       # List of lines
    for line in f:              # Iterate lines

with open('file.txt', 'w') as f:
    f.write('text')
    f.writelines(lines)
```

### Common Imports

```python
# STANDARD LIBRARY
import os                    # OS interface
import sys                   # System-specific
import json                  # JSON encode/decode
import re                    # Regular expressions
import datetime              # Date/time
import collections           # Specialized containers
import itertools             # Iterator functions
import functools             # Higher-order functions
import pathlib               # Object-oriented paths
import logging               # Logging facility
import unittest              # Testing framework
import typing                # Type hints
import dataclasses           # Data classes
import contextlib            # Context managers
import concurrent.futures    # Parallelism
import asyncio               # Async I/O
import copy                  # Shallow/deep copy
import random                # Random numbers
import math                  # Math functions
import statistics            # Statistics functions
import hashlib               # Secure hashes
import pickle                # Object serialization
import sqlite3               # SQLite database
import urllib                # URL handling
import http                  # HTTP modules
import socket                # Low-level networking
import threading             # Thread-based parallelism
import multiprocessing       # Process-based parallelism
```

---

## 18. LEARNING PATH

### Phase 1: Foundations (Week 1-2)
```
□ Data Types & Variables
  - Numbers, strings, booleans
  - None type
  - Type conversion

□ Data Structures
  - Lists, tuples, sets, dicts
  - Mutability
  - Common operations

□ Control Flow
  - if/elif/else
  - for/while loops
  - break/continue/pass
```

### Phase 2: Functions & OOP (Week 3-4)
```
□ Functions
  - Parameters & arguments
  - *args, **kwargs
  - Return values
  - Scope (LEGB)

□ Object-Oriented Programming
  - Classes & objects
  - __init__ and self
  - Instance vs class attributes
  - Inheritance
  - Magic methods
```

### Phase 3: Intermediate (Week 5-6)
```
□ Iterators & Generators
  - Iterator protocol
  - Generator functions
  - Generator expressions
  - itertools

□ Decorators
  - Function decorators
  - @wraps
  - Decorators with arguments
  - Class decorators

□ Context Managers
  - with statement
  - __enter__/__exit__
  - contextlib
```

### Phase 4: Advanced (Week 7-8)
```
□ Concurrency
  - Threading vs multiprocessing
  - GIL understanding
  - asyncio basics
  - concurrent.futures

□ Error Handling
  - Exception hierarchy
  - Custom exceptions
  - Best practices

□ Memory Management
  - Reference counting
  - Garbage collection
  - Weak references
```

### Phase 5: Mastery (Week 9-12)
```
□ Type Hints
  - Basic annotations
  - Generic types
  - mypy

□ Testing
  - unittest/pytest
  - Mocking
  - Coverage

□ Performance
  - Profiling
  - Optimization techniques
  - When to optimize

□ Metaprogramming
  - Metaclasses
  - Descriptors
  - __getattr__, __setattr__
```

### Practice Resources

```
CODING PRACTICE:
  - LeetCode (algorithms)
  - HackerRank (Python track)
  - Exercism (Python)
  - Codewars

PROJECT IDEAS:
  Week 1-2: CLI tools (file organizer, todo app)
  Week 3-4: Data processor (CSV parser, JSON transformer)
  Week 5-6: Web scraper (with async)
  Week 7-8: REST API (FastAPI/Flask)
  Week 9-12: Full application (with tests, typing, docs)

READING:
  - "Fluent Python" by Luciano Ramalho
  - "Python Cookbook" by David Beazley
  - "Effective Python" by Brett Slatkin
  - Python documentation (especially tutorials)
```

---

## FINAL WISDOM

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE PYTHONIC WAY                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  import this                                                    │
│                                                                 │
│  Beautiful is better than ugly.                                 │
│  Explicit is better than implicit.                              │
│  Simple is better than complex.                                 │
│  Complex is better than complicated.                            │
│  Flat is better than nested.                                    │
│  Sparse is better than dense.                                   │
│  Readability counts.                                            │
│  Special cases aren't special enough to break the rules.        │
│  Although practicality beats purity.                            │
│  Errors should never pass silently.                             │
│  Unless explicitly silenced.                                    │
│  In the face of ambiguity, refuse the temptation to guess.      │
│  There should be one-- and preferably only one --obvious way.   │
│  Although that way may not be obvious at first.                 │
│  Now is better than never.                                      │
│  Although never is often better than *right* now.               │
│  If the implementation is hard to explain, it's a bad idea.     │
│  If the implementation is easy to explain, it may be good.      │
│  Namespaces are one honking great idea -- let's do more!        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

*"Python is a language that lets you work quickly and integrate systems more effectively."*
