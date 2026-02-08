# DATA STRUCTURES & ALGORITHMS - MENTAL MODELS

> Master Guide for Technical Interview Excellence

---

## TABLE OF CONTENTS
1. [Big O Complexity Framework](#1-big-o-complexity-framework)
2. [Data Structure Selection Matrix](#2-data-structure-selection-matrix)
3. [Arrays & Strings Mental Model](#3-arrays--strings-mental-model)
4. [Linked Lists Mental Model](#4-linked-lists-mental-model)
5. [Stacks & Queues Mental Model](#5-stacks--queues-mental-model)
6. [Hash Tables Mental Model](#6-hash-tables-mental-model)
7. [Trees Mental Model](#7-trees-mental-model)
8. [Heaps & Priority Queues Mental Model](#8-heaps--priority-queues-mental-model)
9. [Graphs Mental Model](#9-graphs-mental-model)
10. [Sorting Algorithms Mental Model](#10-sorting-algorithms-mental-model)
11. [Searching Algorithms Mental Model](#11-searching-algorithms-mental-model)
12. [Recursion & Backtracking Mental Model](#12-recursion--backtracking-mental-model)
13. [Dynamic Programming Mental Model](#13-dynamic-programming-mental-model)
14. [Greedy Algorithms Mental Model](#14-greedy-algorithms-mental-model)
15. [Two Pointers & Sliding Window](#15-two-pointers--sliding-window)
16. [Problem Pattern Recognition](#16-problem-pattern-recognition)
17. [Quick Reference Card](#17-quick-reference-card)
18. [Learning Path](#18-learning-path)

---

## 1. BIG O COMPLEXITY FRAMEWORK

### The Complexity Hierarchy (Best to Worst)

```
Performance
    ▲
    │
    │  O(1)        ←── Constant      [Array access, Hash lookup]
    │  O(log n)    ←── Logarithmic   [Binary search, Balanced BST]
    │  O(n)        ←── Linear        [Simple loop, Linear search]
    │  O(n log n)  ←── Linearithmic  [Merge sort, Quick sort avg]
    │  O(n²)       ←── Quadratic     [Nested loops, Bubble sort]
    │  O(n³)       ←── Cubic         [Triple nested loops]
    │  O(2ⁿ)       ←── Exponential   [Recursive fib, Subsets]
    │  O(n!)       ←── Factorial     [Permutations, TSP brute]
    │
    └──────────────────────────────────────────► Input Size (n)
```

### Visual Growth Comparison

```
n=10 operations:
┌─────────────────────────────────────────────────────────────────┐
│ O(1)      │ 1                                                   │
│ O(log n)  │ ███ 3                                               │
│ O(n)      │ ██████████ 10                                       │
│ O(n log n)│ ██████████████████████████████ 33                   │
│ O(n²)     │ ████████████████████████████████████████████████... │ 100
│ O(2ⁿ)     │ [Would need 1024 blocks!]                           │
└─────────────────────────────────────────────────────────────────┘
```

### The Time-Space Tradeoff Scale

```
    FASTER                                              LESS MEMORY
      ▲                                                      ▲
      │    ┌─────────────────────────────────────────┐      │
      │    │  Hash Table   │ O(1) time, O(n) space   │      │
      │    ├─────────────────────────────────────────┤      │
      │    │  Memoization  │ Trade space for time    │      │
      │    ├─────────────────────────────────────────┤      │
      │    │  Two Pointers │ O(1) space, same time   │      │
      │    ├─────────────────────────────────────────┤      │
      │    │  Recursion    │ O(n) call stack space   │      │
      │    └─────────────────────────────────────────┘      │
      ▼                                                      ▼
    SLOWER                                              MORE MEMORY
```

### Complexity Analysis Cheat Sheet

```python
# O(1) - Constant
def get_first(arr):
    return arr[0]  # Always one operation

# O(log n) - Logarithmic (halving each step)
def binary_search(arr, target):
    left, right = 0, len(arr) - 1
    while left <= right:          # Halves search space
        mid = (left + right) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return -1

# O(n) - Linear
def find_max(arr):
    max_val = arr[0]
    for num in arr:               # Visit each element once
        max_val = max(max_val, num)
    return max_val

# O(n log n) - Linearithmic
def merge_sort(arr):              # Divide: log n levels
    # Each level processes n elements
    pass

# O(n²) - Quadratic
def bubble_sort(arr):
    for i in range(len(arr)):           # n iterations
        for j in range(len(arr) - 1):   # n iterations each
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]

# O(2ⁿ) - Exponential
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)  # Two calls each level
```

### Amortized Analysis Concept

```
Dynamic Array (ArrayList) append:
┌─────────────────────────────────────────────────────────────┐
│ Operation: 1  2  3  4  5  6  7  8  9  ...                   │
│ Cost:      1  1  1  C  1  1  1  C  1  ...                   │
│                    ↑           ↑                             │
│              Resize to 4   Resize to 8                       │
│                                                              │
│ C = Copy all elements (expensive but rare)                   │
│ Average (Amortized) = O(1) per operation                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. DATA STRUCTURE SELECTION MATRIX

### Decision Flowchart

```
                    What operation is most frequent?
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
    ACCESS by              SEARCH for           INSERT/DELETE
    index/key?             element?             frequently?
        │                     │                     │
   ┌────┴────┐           ┌────┴────┐          ┌────┴────┐
   ▼         ▼           ▼         ▼          ▼         ▼
Index?    Key?      Sorted?   Unsorted?   Where?    Order
   │         │           │         │          │      matters?
   ▼         ▼           ▼         ▼          │         │
ARRAY    HASH MAP   BINARY    LINEAR     ┌───┴───┐    │
O(1)      O(1)      SEARCH    SEARCH     ▼       ▼    ▼
                    O(log n)   O(n)    Start/   Middle
                                       End         │
                                        │          ▼
                              ┌─────────┴──┐   LINKED
                              ▼            ▼    LIST
                           STACK/       DEQUE
                           QUEUE
```

### Data Structure Comparison Table

```
┌──────────────────┬─────────┬─────────┬─────────┬─────────┬─────────┐
│ Data Structure   │ Access  │ Search  │ Insert  │ Delete  │ Space   │
├──────────────────┼─────────┼─────────┼─────────┼─────────┼─────────┤
│ Array            │ O(1)    │ O(n)    │ O(n)    │ O(n)    │ O(n)    │
│ Sorted Array     │ O(1)    │ O(logn) │ O(n)    │ O(n)    │ O(n)    │
│ Linked List      │ O(n)    │ O(n)    │ O(1)*   │ O(1)*   │ O(n)    │
│ Stack            │ O(n)    │ O(n)    │ O(1)    │ O(1)    │ O(n)    │
│ Queue            │ O(n)    │ O(n)    │ O(1)    │ O(1)    │ O(n)    │
│ Hash Table       │ O(1)    │ O(1)    │ O(1)    │ O(1)    │ O(n)    │
│ BST (balanced)   │ O(logn) │ O(logn) │ O(logn) │ O(logn) │ O(n)    │
│ Heap             │ O(1)**  │ O(n)    │ O(logn) │ O(logn) │ O(n)    │
│ Trie             │ O(k)    │ O(k)    │ O(k)    │ O(k)    │ O(n*k)  │
└──────────────────┴─────────┴─────────┴─────────┴─────────┴─────────┘
* If position known   ** Only for min/max   k = key length
```

### When to Use What - Quick Guide

```
┌────────────────────────────────────────────────────────────────────┐
│ SCENARIO                          │ BEST CHOICE                    │
├────────────────────────────────────────────────────────────────────┤
│ Fast lookup by key                │ Hash Map / Dictionary          │
│ Maintain sorted order             │ BST / TreeMap / Sorted Array   │
│ FIFO processing                   │ Queue                          │
│ LIFO / Undo operations            │ Stack                          │
│ Find min/max repeatedly           │ Heap / Priority Queue          │
│ Prefix matching / Autocomplete    │ Trie                           │
│ Range queries                     │ Segment Tree / BIT             │
│ Disjoint sets / Union-Find        │ Union-Find (DSU)               │
│ Frequent insert at both ends      │ Deque                          │
│ Graph with dense connections      │ Adjacency Matrix               │
│ Graph with sparse connections     │ Adjacency List                 │
│ Need both O(1) access & order     │ LinkedHashMap                  │
│ Count frequencies                 │ Hash Map / Counter             │
│ Check membership                  │ Hash Set                       │
│ LRU Cache                         │ Hash Map + Doubly Linked List  │
└────────────────────────────────────────────────────────────────────┘
```

---

## 3. ARRAYS & STRINGS MENTAL MODEL

### Array Memory Layout (Contiguous)

```
Memory Address:  1000   1004   1008   1012   1016
                ┌──────┬──────┬──────┬──────┬──────┐
Array [5,2,8,1] │  5   │  2   │  8   │  1   │      │
                └──────┴──────┴──────┴──────┴──────┘
                  [0]    [1]    [2]    [3]

Access arr[2]:
  address = base_address + (index * element_size)
  address = 1000 + (2 * 4) = 1008  →  Returns 8
  Time: O(1)
```

### Common Array Patterns

```
1. TWO POINTERS (Sorted Array)
   ┌───┬───┬───┬───┬───┬───┬───┐
   │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │   Find pair with sum = 8
   └───┴───┴───┴───┴───┴───┴───┘
     ▲                       ▲
     L                       R
     └─── Sum = 8? Move pointers based on comparison

2. SLIDING WINDOW (Subarray problems)
   ┌───┬───┬───┬───┬───┬───┬───┐
   │ 1 │ 3 │ 2 │ 6 │ 4 │ 8 │ 2 │   Max sum of k=3 elements
   └───┴───┴───┴───┴───┴───┴───┘
     └─────────┘
      Window slides →

3. PREFIX SUM (Range queries)
   Original: [1, 2, 3, 4, 5]
   Prefix:   [1, 3, 6, 10, 15]

   Sum(i to j) = prefix[j] - prefix[i-1]
   Sum(2 to 4) = prefix[4] - prefix[1] = 15 - 3 = 12

4. IN-PLACE MODIFICATION
   Remove duplicates from sorted array:
   [1,1,2,2,3] → [1,2,3,_,_]
    w r          w = write pointer, r = read pointer
```

### String-Specific Patterns

```python
# ANAGRAM CHECK - Use character frequency
def is_anagram(s1, s2):
    return Counter(s1) == Counter(s2)

# PALINDROME CHECK - Two pointers
def is_palindrome(s):
    left, right = 0, len(s) - 1
    while left < right:
        if s[left] != s[right]:
            return False
        left += 1
        right -= 1
    return True

# SUBSTRING SEARCH - Sliding window + hash
def find_anagrams(s, p):
    """Find all anagram starting indices"""
    result = []
    p_count = Counter(p)
    window = Counter(s[:len(p)])

    if window == p_count:
        result.append(0)

    for i in range(len(p), len(s)):
        # Add new char, remove old char
        window[s[i]] += 1
        window[s[i - len(p)]] -= 1
        if window[s[i - len(p)]] == 0:
            del window[s[i - len(p)]]

        if window == p_count:
            result.append(i - len(p) + 1)

    return result
```

### String Immutability Concept

```
Python String (Immutable):
┌───────────────────────────────────────────────────┐
│  s = "hello"                                      │
│  s[0] = 'H'  ← ERROR! Cannot modify               │
│                                                   │
│  s = s + " world"  ← Creates NEW string           │
│                                                   │
│  For many modifications: Use list, then join()   │
│  chars = list(s)                                  │
│  chars[0] = 'H'                                   │
│  s = ''.join(chars)                               │
└───────────────────────────────────────────────────┘

String Building Complexity:
  Naive concatenation in loop: O(n²)
  Using list + join:           O(n)
  Using StringBuilder (Java):  O(n)
```

---

## 4. LINKED LISTS MENTAL MODEL

### Structure Visualization

```
SINGLY LINKED LIST:
┌──────┬────┐   ┌──────┬────┐   ┌──────┬────┐   ┌──────┬────┐
│  1   │  ──┼──►│  2   │  ──┼──►│  3   │  ──┼──►│  4   │ ∅  │
└──────┴────┘   └──────┴────┘   └──────┴────┘   └──────┴────┘
   HEAD                                             TAIL

DOUBLY LINKED LIST:
┌────┬──────┬────┐   ┌────┬──────┬────┐   ┌────┬──────┬────┐
│ ∅  │  1   │  ──┼──►│◄───│  2   │  ──┼──►│◄───│  3   │ ∅  │
└────┴──────┴────┘   └────┴──────┴────┘   └────┴──────┴────┘
       HEAD                                        TAIL
```

### Key Operations

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

# INSERT at position (O(n) to find, O(1) to insert)
def insert_after(node, val):
    new_node = ListNode(val)
    new_node.next = node.next
    node.next = new_node

# DELETE node (O(1) if have reference)
def delete_next(node):
    if node.next:
        node.next = node.next.next

# REVERSE (Iterative - O(n) time, O(1) space)
def reverse(head):
    prev = None
    curr = head
    while curr:
        next_temp = curr.next  # Save next
        curr.next = prev       # Reverse pointer
        prev = curr            # Move prev forward
        curr = next_temp       # Move curr forward
    return prev

# FIND MIDDLE (Fast-Slow pointers)
def find_middle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next       # Move 1 step
        fast = fast.next.next  # Move 2 steps
    return slow  # Slow is at middle
```

### The Fast-Slow Pointer Pattern

```
CYCLE DETECTION (Floyd's Algorithm):
┌───┐   ┌───┐   ┌───┐   ┌───┐
│ 1 │──►│ 2 │──►│ 3 │──►│ 4 │
└───┘   └───┘   └───┘   └─┬─┘
                    ▲     │
                    │     ▼
                  ┌───┐ ┌───┐
                  │ 6 │◄│ 5 │
                  └───┘ └───┘

Fast pointer moves 2x speed → If cycle exists, they WILL meet
If fast reaches null → No cycle

def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            return True
    return False
```

### Linked List Patterns Summary

```
┌─────────────────────────────────────────────────────────────────┐
│ PATTERN              │ USE CASE                                 │
├─────────────────────────────────────────────────────────────────┤
│ Dummy Head           │ Simplify edge cases (empty list, head)   │
│ Fast-Slow Pointers   │ Find middle, detect cycle, find cycle    │
│                      │ start, find nth from end                 │
│ Reverse              │ Palindrome check, reverse in groups      │
│ Merge                │ Merge sorted lists, merge k lists        │
│ Recursion            │ Natural for many operations              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. STACKS & QUEUES MENTAL MODEL

### Stack: LIFO (Last In, First Out)

```
STACK (Like a stack of plates):
         ┌─────┐
   push→ │  4  │ ←pop (most recent)
         ├─────┤
         │  3  │
         ├─────┤
         │  2  │
         ├─────┤
         │  1  │ (oldest)
         └─────┘

Operations:
  push(x)  - Add to top     O(1)
  pop()    - Remove top     O(1)
  peek()   - View top       O(1)
  isEmpty()- Check empty    O(1)
```

### Queue: FIFO (First In, First Out)

```
QUEUE (Like a line at store):
         ┌─────┬─────┬─────┬─────┐
enqueue→ │  4  │  3  │  2  │  1  │ →dequeue
         └─────┴─────┴─────┴─────┘
         back               front

Operations:
  enqueue(x) - Add to back    O(1)
  dequeue()  - Remove front   O(1)
  front()    - View front     O(1)
  isEmpty()  - Check empty    O(1)
```

### Stack Applications

```python
# 1. BALANCED PARENTHESES
def is_valid(s):
    stack = []
    mapping = {')': '(', '}': '{', ']': '['}

    for char in s:
        if char in mapping:  # Closing bracket
            if not stack or stack[-1] != mapping[char]:
                return False
            stack.pop()
        else:  # Opening bracket
            stack.append(char)

    return len(stack) == 0

# 2. MONOTONIC STACK (Next Greater Element)
def next_greater(nums):
    result = [-1] * len(nums)
    stack = []  # Stores indices

    for i, num in enumerate(nums):
        while stack and nums[stack[-1]] < num:
            idx = stack.pop()
            result[idx] = num
        stack.append(i)

    return result

# Example: [4, 5, 2, 25] → [5, 25, 25, -1]

# 3. EXPRESSION EVALUATION
def eval_rpn(tokens):
    """Reverse Polish Notation: ["2","1","+","3","*"] = 9"""
    stack = []
    ops = {'+': lambda a,b: a+b, '-': lambda a,b: a-b,
           '*': lambda a,b: a*b, '/': lambda a,b: int(a/b)}

    for token in tokens:
        if token in ops:
            b, a = stack.pop(), stack.pop()
            stack.append(ops[token](a, b))
        else:
            stack.append(int(token))

    return stack[0]
```

### Queue Applications

```python
# 1. BFS (Level Order Traversal)
from collections import deque

def level_order(root):
    if not root:
        return []

    result = []
    queue = deque([root])

    while queue:
        level = []
        for _ in range(len(queue)):  # Process current level
            node = queue.popleft()
            level.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        result.append(level)

    return result

# 2. SLIDING WINDOW MAXIMUM (Monotonic Deque)
def max_sliding_window(nums, k):
    dq = deque()  # Stores indices
    result = []

    for i, num in enumerate(nums):
        # Remove indices outside window
        while dq and dq[0] <= i - k:
            dq.popleft()

        # Remove smaller elements (they'll never be max)
        while dq and nums[dq[-1]] < num:
            dq.pop()

        dq.append(i)

        if i >= k - 1:
            result.append(nums[dq[0]])

    return result
```

### Monotonic Stack/Queue Pattern

```
MONOTONIC DECREASING STACK:
┌──────────────────────────────────────────────────────────────┐
│ Maintains elements in decreasing order from bottom to top    │
│                                                              │
│ Input: [3, 1, 4, 1, 5, 9, 2, 6]                              │
│                                                              │
│ Process 3: Stack [3]                                         │
│ Process 1: Stack [3, 1]      (1 < 3, push)                   │
│ Process 4: Stack [4]         (4 > 1 and 3, pop both, push 4) │
│ Process 1: Stack [4, 1]                                      │
│ Process 5: Stack [5]         (5 > 1 and 4)                   │
│ Process 9: Stack [9]                                         │
│ Process 2: Stack [9, 2]                                      │
│ Process 6: Stack [9, 6]      (6 > 2)                         │
│                                                              │
│ Use: Next greater/smaller element, histogram problems        │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. HASH TABLES MENTAL MODEL

### Hash Table Internals

```
Hash Function: key → hash code → bucket index

Example: hash("apple") = 4829503 % 8 = 7

┌─────────────────────────────────────────────────────────┐
│ Index │ Bucket (with chaining for collisions)          │
├───────┼─────────────────────────────────────────────────┤
│   0   │ ["banana": 2]                                   │
│   1   │ []                                              │
│   2   │ ["grape": 5] → ["kiwi": 1]  ← Collision chain   │
│   3   │ []                                              │
│   4   │ ["orange": 3]                                   │
│   5   │ []                                              │
│   6   │ []                                              │
│   7   │ ["apple": 4]                                    │
└───────┴─────────────────────────────────────────────────┘
```

### Collision Resolution Strategies

```
1. CHAINING (Linked List):
   Bucket[i] → [key1:val1] → [key2:val2] → [key3:val3]

2. OPEN ADDRESSING (Linear Probing):
   If slot taken, try next: (hash + 1) % size, (hash + 2) % size...

   ┌───┬───┬───┬───┬───┬───┬───┬───┐
   │ A │   │ B │ C │   │ D │ E │   │
   └───┴───┴───┴───┴───┴───┴───┴───┘
     ↑       ↑
   hash(A)  hash(B) collides, probe to next empty
```

### Common Hash Table Patterns

```python
# 1. FREQUENCY COUNT
def top_k_frequent(nums, k):
    count = Counter(nums)
    return [x for x, _ in count.most_common(k)]

# 2. TWO SUM (Classic)
def two_sum(nums, target):
    seen = {}  # num → index
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    return []

# 3. GROUP ANAGRAMS
def group_anagrams(strs):
    groups = defaultdict(list)
    for s in strs:
        key = tuple(sorted(s))  # Or use char count tuple
        groups[key].append(s)
    return list(groups.values())

# 4. SUBARRAY SUM EQUALS K
def subarray_sum(nums, k):
    count = 0
    prefix_sum = 0
    sum_count = {0: 1}  # prefix_sum → frequency

    for num in nums:
        prefix_sum += num
        # If (prefix_sum - k) exists, we found subarrays
        count += sum_count.get(prefix_sum - k, 0)
        sum_count[prefix_sum] = sum_count.get(prefix_sum, 0) + 1

    return count

# 5. LRU CACHE (Hash Map + Doubly Linked List)
class LRUCache:
    def __init__(self, capacity):
        self.cache = OrderedDict()
        self.capacity = capacity

    def get(self, key):
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)  # Mark as recently used
        return self.cache[key]

    def put(self, key, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)  # Remove oldest
```

### Hash Set Applications

```python
# 1. CONTAINS DUPLICATE
def contains_duplicate(nums):
    return len(nums) != len(set(nums))

# 2. INTERSECTION OF ARRAYS
def intersection(nums1, nums2):
    return list(set(nums1) & set(nums2))

# 3. LONGEST CONSECUTIVE SEQUENCE
def longest_consecutive(nums):
    num_set = set(nums)
    max_length = 0

    for num in num_set:
        # Only start counting from sequence start
        if num - 1 not in num_set:
            current = num
            length = 1

            while current + 1 in num_set:
                current += 1
                length += 1

            max_length = max(max_length, length)

    return max_length
```

---

## 7. TREES MENTAL MODEL

### Tree Terminology Visual

```
                    ┌───┐
         root  ──►  │ 1 │  ◄── depth 0
                    └─┬─┘
            ┌─────────┴─────────┐
            ▼                   ▼
          ┌───┐               ┌───┐
          │ 2 │               │ 3 │  ◄── depth 1
          └─┬─┘               └─┬─┘
        ┌───┴───┐           ┌───┴───┐
        ▼       ▼           ▼       ▼
      ┌───┐   ┌───┐       ┌───┐   ┌───┐
      │ 4 │   │ 5 │       │ 6 │   │ 7 │  ◄── depth 2 (leaves)
      └───┘   └───┘       └───┘   └───┘

Terminology:
  - Node 1: Root (no parent)
  - Node 2, 3: Children of 1, Siblings
  - Node 4, 5: Children of 2
  - Nodes 4,5,6,7: Leaves (no children)
  - Height = max depth = 2
  - Node 2's subtree: {2, 4, 5}
```

### Binary Search Tree (BST) Property

```
BST Invariant: Left < Node < Right (for all nodes)

              ┌────┐
              │ 8  │
              └──┬─┘
         ┌──────┴──────┐
         ▼             ▼
       ┌────┐        ┌────┐
       │ 3  │        │ 10 │
       └──┬─┘        └──┬─┘
      ┌───┴───┐         └───┐
      ▼       ▼             ▼
    ┌───┐   ┌───┐         ┌───┐
    │ 1 │   │ 6 │         │ 14│
    └───┘   └─┬─┘         └───┘
           ┌──┴──┐
           ▼     ▼
         ┌───┐ ┌───┐
         │ 4 │ │ 7 │
         └───┘ └───┘

Search for 6:  8→3→6  ✓  O(log n) average
Insert 5:     8→3→6→4→(insert as right child of 4)
```

### Tree Traversals

```
              1
            /   \
           2     3
          / \   / \
         4   5 6   7

PREORDER (Root-Left-Right):  1, 2, 4, 5, 3, 6, 7
  "Visit before children" - Used for copying tree

INORDER (Left-Root-Right):   4, 2, 5, 1, 6, 3, 7
  "Visit between children" - BST gives sorted order

POSTORDER (Left-Right-Root): 4, 5, 2, 6, 7, 3, 1
  "Visit after children" - Used for deletion, eval expression

LEVEL ORDER (BFS):           1, 2, 3, 4, 5, 6, 7
  "Level by level" - Used for level-wise operations
```

### Traversal Implementations

```python
# RECURSIVE TRAVERSALS
def preorder(root):
    if not root:
        return []
    return [root.val] + preorder(root.left) + preorder(root.right)

def inorder(root):
    if not root:
        return []
    return inorder(root.left) + [root.val] + inorder(root.right)

def postorder(root):
    if not root:
        return []
    return postorder(root.left) + postorder(root.right) + [root.val]

# ITERATIVE INORDER (Morris or Stack)
def inorder_iterative(root):
    result = []
    stack = []
    current = root

    while current or stack:
        # Go left as far as possible
        while current:
            stack.append(current)
            current = current.left

        current = stack.pop()
        result.append(current.val)
        current = current.right

    return result

# LEVEL ORDER (BFS)
def level_order(root):
    if not root:
        return []

    result = []
    queue = deque([root])

    while queue:
        level = []
        for _ in range(len(queue)):
            node = queue.popleft()
            level.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        result.append(level)

    return result
```

### Common Tree Patterns

```python
# 1. MAX DEPTH
def max_depth(root):
    if not root:
        return 0
    return 1 + max(max_depth(root.left), max_depth(root.right))

# 2. VALIDATE BST
def is_valid_bst(root, min_val=float('-inf'), max_val=float('inf')):
    if not root:
        return True
    if not (min_val < root.val < max_val):
        return False
    return (is_valid_bst(root.left, min_val, root.val) and
            is_valid_bst(root.right, root.val, max_val))

# 3. LOWEST COMMON ANCESTOR (LCA)
def lca(root, p, q):
    if not root or root == p or root == q:
        return root

    left = lca(root.left, p, q)
    right = lca(root.right, p, q)

    if left and right:  # p and q on different sides
        return root
    return left or right  # Both on same side

# 4. PATH SUM
def has_path_sum(root, target):
    if not root:
        return False
    if not root.left and not root.right:  # Leaf
        return root.val == target
    return (has_path_sum(root.left, target - root.val) or
            has_path_sum(root.right, target - root.val))

# 5. SERIALIZE/DESERIALIZE
def serialize(root):
    if not root:
        return "null"
    return f"{root.val},{serialize(root.left)},{serialize(root.right)}"
```

### Balanced Trees Overview

```
AVL TREE:
  - Balance factor = |height(left) - height(right)| <= 1
  - Rotations: Left, Right, Left-Right, Right-Left
  - Strictly balanced → O(log n) guaranteed

RED-BLACK TREE:
  - Each node is red or black
  - Root and leaves (NIL) are black
  - Red nodes have black children
  - Same black-height on all paths
  - Less strict → fewer rotations

B-TREE:
  - Multiple keys per node
  - Used in databases, filesystems
  - Minimizes disk reads
```

---

## 8. HEAPS & PRIORITY QUEUES MENTAL MODEL

### Heap Structure (Min-Heap Example)

```
COMPLETE BINARY TREE (Array representation):

Tree View:                    Array View:
        ┌───┐                 ┌───┬───┬───┬───┬───┬───┬───┐
        │ 1 │                 │ 1 │ 3 │ 2 │ 7 │ 4 │ 5 │ 6 │
        └─┬─┘                 └───┴───┴───┴───┴───┴───┴───┘
     ┌────┴────┐              idx: 0   1   2   3   4   5   6
     ▼         ▼
   ┌───┐     ┌───┐            Parent of i: (i-1) // 2
   │ 3 │     │ 2 │            Left child:  2*i + 1
   └─┬─┘     └─┬─┘            Right child: 2*i + 2
  ┌──┴──┐   ┌──┴──┐
  ▼     ▼   ▼     ▼
┌───┐ ┌───┐ ┌───┐ ┌───┐
│ 7 │ │ 4 │ │ 5 │ │ 6 │
└───┘ └───┘ └───┘ └───┘

HEAP PROPERTY:
  Min-Heap: parent <= children (root is minimum)
  Max-Heap: parent >= children (root is maximum)
```

### Heap Operations

```python
import heapq

# Python heapq is MIN-HEAP by default
nums = [3, 1, 4, 1, 5, 9]
heapq.heapify(nums)  # O(n) - Convert to heap

# Push - O(log n)
heapq.heappush(nums, 2)

# Pop minimum - O(log n)
min_val = heapq.heappop(nums)

# Peek minimum - O(1)
min_val = nums[0]

# Push and pop in one operation - O(log n)
heapq.heappushpop(nums, 6)  # Push 6, pop minimum

# For MAX-HEAP: negate values
max_heap = []
heapq.heappush(max_heap, -5)  # Insert 5
max_val = -heapq.heappop(max_heap)  # Get max
```

### Heap Operations Visualized

```
INSERT 0 into Min-Heap [1, 3, 2, 7, 4]:

Step 1: Add at end          Step 2: Bubble up (heapify up)
        1                           1                    0
       / \                         / \                  / \
      3   2         →             0   2       →        1   2
     / \  |                      / \                  / \
    7   4 0                     7   4                7   4
                                   ↑                    ↑
                               3 moved down         3 moved down

EXTRACT MIN from [1, 3, 2, 7, 4]:

Step 1: Swap root with last   Step 2: Bubble down (heapify down)
        1                           4                    2
       / \                         / \                  / \
      3   2         →             3   2       →        3   4
     / \                         /                    /
    7   4                       7                    7

    Remove 1                   4 > children, swap with min child (2)
```

### Priority Queue Applications

```python
# 1. K LARGEST ELEMENTS (Min-heap of size k)
def k_largest(nums, k):
    # Keep k largest in min-heap
    heap = nums[:k]
    heapq.heapify(heap)

    for num in nums[k:]:
        if num > heap[0]:  # Larger than smallest of k largest
            heapq.heapreplace(heap, num)

    return heap

# 2. MERGE K SORTED LISTS
def merge_k_lists(lists):
    heap = []
    for i, lst in enumerate(lists):
        if lst:
            heapq.heappush(heap, (lst.val, i, lst))

    dummy = ListNode()
    current = dummy

    while heap:
        val, i, node = heapq.heappop(heap)
        current.next = node
        current = current.next

        if node.next:
            heapq.heappush(heap, (node.next.val, i, node.next))

    return dummy.next

# 3. FIND MEDIAN FROM DATA STREAM
class MedianFinder:
    def __init__(self):
        self.small = []  # Max-heap (negated)
        self.large = []  # Min-heap

    def addNum(self, num):
        heapq.heappush(self.small, -num)

        # Ensure small's max <= large's min
        if self.large and -self.small[0] > self.large[0]:
            heapq.heappush(self.large, -heapq.heappop(self.small))

        # Balance sizes
        if len(self.small) > len(self.large) + 1:
            heapq.heappush(self.large, -heapq.heappop(self.small))
        elif len(self.large) > len(self.small):
            heapq.heappush(self.small, -heapq.heappop(self.large))

    def findMedian(self):
        if len(self.small) > len(self.large):
            return -self.small[0]
        return (-self.small[0] + self.large[0]) / 2

# 4. TASK SCHEDULER
def least_interval(tasks, n):
    counts = Counter(tasks)
    max_heap = [-c for c in counts.values()]
    heapq.heapify(max_heap)

    time = 0
    while max_heap:
        cycle = []
        for _ in range(n + 1):
            if max_heap:
                cycle.append(heapq.heappop(max_heap))

        for count in cycle:
            if count + 1 < 0:  # Still has tasks
                heapq.heappush(max_heap, count + 1)

        time += n + 1 if max_heap else len(cycle)

    return time
```

---

## 9. GRAPHS MENTAL MODEL

### Graph Representations

```
UNDIRECTED GRAPH:
    A --- B
    |   / |
    | /   |
    C --- D

ADJACENCY LIST:                    ADJACENCY MATRIX:
{                                      A  B  C  D
  'A': ['B', 'C'],                 A [ 0  1  1  0 ]
  'B': ['A', 'C', 'D'],            B [ 1  0  1  1 ]
  'C': ['A', 'B', 'D'],            C [ 1  1  0  1 ]
  'D': ['B', 'C']                  D [ 0  1  1  0 ]
}

Space: O(V + E)                    Space: O(V²)
Edge lookup: O(degree)             Edge lookup: O(1)
Best for: Sparse graphs            Best for: Dense graphs
```

### BFS vs DFS Visual

```
        1
       /|\
      2 3 4
     /|   |
    5 6   7

BFS (Level by level):              DFS (Go deep first):
Queue: [1]      Visit: 1           Stack: [1]      Visit: 1
Queue: [2,3,4]  Visit: 2           Stack: [2,3,4]  Visit: 4
Queue: [3,4,5,6] Visit: 3          Stack: [2,3,7]  Visit: 7
Queue: [4,5,6]  Visit: 4           Stack: [2,3]    Visit: 3
Queue: [5,6,7]  Visit: 5           Stack: [2]      Visit: 2
Queue: [6,7]    Visit: 6           Stack: [5,6]    Visit: 6
Queue: [7]      Visit: 7           Stack: [5]      Visit: 5

Order: 1,2,3,4,5,6,7               Order: 1,4,7,3,2,6,5
Use: Shortest path, level order    Use: Connectivity, cycle detection
```

### Graph Traversal Templates

```python
# BFS TEMPLATE
from collections import deque

def bfs(graph, start):
    visited = {start}
    queue = deque([start])

    while queue:
        node = queue.popleft()
        # Process node

        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)

# DFS TEMPLATE (Iterative)
def dfs_iterative(graph, start):
    visited = set()
    stack = [start]

    while stack:
        node = stack.pop()
        if node in visited:
            continue
        visited.add(node)
        # Process node

        for neighbor in graph[node]:
            if neighbor not in visited:
                stack.append(neighbor)

# DFS TEMPLATE (Recursive)
def dfs_recursive(graph, node, visited=None):
    if visited is None:
        visited = set()

    visited.add(node)
    # Process node

    for neighbor in graph[node]:
        if neighbor not in visited:
            dfs_recursive(graph, neighbor, visited)
```

### Common Graph Algorithms

```python
# 1. SHORTEST PATH (Unweighted) - BFS
def shortest_path(graph, start, end):
    queue = deque([(start, [start])])
    visited = {start}

    while queue:
        node, path = queue.popleft()
        if node == end:
            return path

        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append((neighbor, path + [neighbor]))

    return None

# 2. DETECT CYCLE (Undirected)
def has_cycle_undirected(graph):
    visited = set()

    def dfs(node, parent):
        visited.add(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                if dfs(neighbor, node):
                    return True
            elif neighbor != parent:  # Back edge found
                return True
        return False

    for node in graph:
        if node not in visited:
            if dfs(node, None):
                return True
    return False

# 3. DETECT CYCLE (Directed) - Three colors
def has_cycle_directed(graph):
    WHITE, GRAY, BLACK = 0, 1, 2
    color = {node: WHITE for node in graph}

    def dfs(node):
        color[node] = GRAY
        for neighbor in graph[node]:
            if color[neighbor] == GRAY:  # Back edge
                return True
            if color[neighbor] == WHITE and dfs(neighbor):
                return True
        color[node] = BLACK
        return False

    return any(color[node] == WHITE and dfs(node) for node in graph)

# 4. TOPOLOGICAL SORT (Kahn's Algorithm - BFS)
def topological_sort(graph):
    in_degree = {node: 0 for node in graph}
    for node in graph:
        for neighbor in graph[node]:
            in_degree[neighbor] += 1

    queue = deque([node for node in graph if in_degree[node] == 0])
    result = []

    while queue:
        node = queue.popleft()
        result.append(node)

        for neighbor in graph[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)

    return result if len(result) == len(graph) else []  # Empty if cycle

# 5. DIJKSTRA (Weighted Shortest Path)
def dijkstra(graph, start):
    distances = {node: float('inf') for node in graph}
    distances[start] = 0
    heap = [(0, start)]

    while heap:
        dist, node = heapq.heappop(heap)

        if dist > distances[node]:
            continue

        for neighbor, weight in graph[node]:
            new_dist = dist + weight
            if new_dist < distances[neighbor]:
                distances[neighbor] = new_dist
                heapq.heappush(heap, (new_dist, neighbor))

    return distances

# 6. UNION-FIND (Disjoint Set Union)
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n

    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])  # Path compression
        return self.parent[x]

    def union(self, x, y):
        px, py = self.find(x), self.find(y)
        if px == py:
            return False
        # Union by rank
        if self.rank[px] < self.rank[py]:
            px, py = py, px
        self.parent[py] = px
        if self.rank[px] == self.rank[py]:
            self.rank[px] += 1
        return True
```

### Graph Problem Patterns

```
┌─────────────────────────────────────────────────────────────────┐
│ PROBLEM TYPE              │ ALGORITHM/APPROACH                  │
├─────────────────────────────────────────────────────────────────┤
│ Shortest path (unweight)  │ BFS                                 │
│ Shortest path (weighted)  │ Dijkstra (no negative edges)        │
│ Shortest path (negative)  │ Bellman-Ford                        │
│ All pairs shortest        │ Floyd-Warshall                      │
│ Minimum spanning tree     │ Kruskal's / Prim's                  │
│ Connectivity              │ DFS / BFS / Union-Find              │
│ Cycle detection           │ DFS with colors / Union-Find        │
│ Topological sort          │ DFS post-order / Kahn's BFS         │
│ Bipartite check           │ BFS/DFS with 2-coloring             │
│ Strongly connected        │ Tarjan's / Kosaraju's               │
│ Network flow              │ Ford-Fulkerson / Edmonds-Karp       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10. SORTING ALGORITHMS MENTAL MODEL

### Sorting Algorithm Comparison

```
┌──────────────────┬─────────┬─────────┬─────────┬─────────┬──────────┐
│ Algorithm        │ Best    │ Average │ Worst   │ Space   │ Stable   │
├──────────────────┼─────────┼─────────┼─────────┼─────────┼──────────┤
│ Bubble Sort      │ O(n)    │ O(n²)   │ O(n²)   │ O(1)    │ Yes      │
│ Selection Sort   │ O(n²)   │ O(n²)   │ O(n²)   │ O(1)    │ No       │
│ Insertion Sort   │ O(n)    │ O(n²)   │ O(n²)   │ O(1)    │ Yes      │
│ Merge Sort       │ O(nlogn)│ O(nlogn)│ O(nlogn)│ O(n)    │ Yes      │
│ Quick Sort       │ O(nlogn)│ O(nlogn)│ O(n²)   │ O(logn) │ No       │
│ Heap Sort        │ O(nlogn)│ O(nlogn)│ O(nlogn)│ O(1)    │ No       │
│ Counting Sort    │ O(n+k)  │ O(n+k)  │ O(n+k)  │ O(k)    │ Yes      │
│ Radix Sort       │ O(nk)   │ O(nk)   │ O(nk)   │ O(n+k)  │ Yes      │
│ Bucket Sort      │ O(n+k)  │ O(n+k)  │ O(n²)   │ O(n)    │ Yes      │
└──────────────────┴─────────┴─────────┴─────────┴─────────┴──────────┘
k = range of input values, Stable = maintains relative order of equals
```

### Algorithm Visualizations

```
MERGE SORT (Divide and Conquer):
[38, 27, 43, 3, 9, 82, 10]
         │
    ┌────┴────┐
    ▼         ▼
[38,27,43,3] [9,82,10]
    │            │
  ┌─┴─┐       ┌──┴──┐
  ▼   ▼       ▼     ▼
[38,27][43,3] [9,82][10]
  │  │   │  │   │  │   │
  ▼  ▼   ▼  ▼   ▼  ▼   ▼
[38][27][43][3][9][82][10]  ← Divide complete

[27,38][3,43][9,82][10]      ← Merge step
   └──┬──┘  └──┬──┘
[3,27,38,43][9,10,82]        ← Merge step
      └────┬────┘
[3,9,10,27,38,43,82]         ← Final merge

QUICK SORT (Pivot partitioning):
[4, 2, 6, 5, 3, 9]  pivot=4
     │
  ┌──┴──┐
  ▼     ▼
[2,3] 4 [6,5,9]    Elements < pivot | pivot | Elements > pivot
  │       │
  ▼       ▼
[2,3]   [5,6,9]
  │       │
Result: [2,3,4,5,6,9]
```

### Key Sorting Implementations

```python
# MERGE SORT - O(n log n) guaranteed, stable
def merge_sort(arr):
    if len(arr) <= 1:
        return arr

    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])

    return merge(left, right)

def merge(left, right):
    result = []
    i = j = 0

    while i < len(left) and j < len(right):
        if left[i] <= right[j]:  # <= for stability
            result.append(left[i])
            i += 1
        else:
            result.append(right[j])
            j += 1

    result.extend(left[i:])
    result.extend(right[j:])
    return result

# QUICK SORT - O(n log n) average, in-place
def quick_sort(arr, low=0, high=None):
    if high is None:
        high = len(arr) - 1

    if low < high:
        pivot_idx = partition(arr, low, high)
        quick_sort(arr, low, pivot_idx - 1)
        quick_sort(arr, pivot_idx + 1, high)

def partition(arr, low, high):
    pivot = arr[high]
    i = low - 1

    for j in range(low, high):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]

    arr[i + 1], arr[high] = arr[high], arr[i + 1]
    return i + 1

# COUNTING SORT - O(n + k) for small range integers
def counting_sort(arr):
    if not arr:
        return arr

    min_val, max_val = min(arr), max(arr)
    count = [0] * (max_val - min_val + 1)

    for num in arr:
        count[num - min_val] += 1

    result = []
    for i, c in enumerate(count):
        result.extend([i + min_val] * c)

    return result
```

### Sorting Decision Guide

```
Which sorting algorithm to use?

Is data nearly sorted?
├── Yes → INSERTION SORT (O(n) for nearly sorted)
└── No
    │
    Is stability required?
    ├── Yes → MERGE SORT
    └── No
        │
        Is memory limited?
        ├── Yes → HEAP SORT (O(1) extra space)
        └── No
            │
            Are values in small range?
            ├── Yes → COUNTING SORT (O(n+k))
            └── No → QUICK SORT (fastest average case)

Small arrays (n < 20): INSERTION SORT is fastest due to low overhead
```

---

## 11. SEARCHING ALGORITHMS MENTAL MODEL

### Binary Search Template

```
Binary Search: Find target in SORTED array O(log n)

[1, 3, 5, 7, 9, 11, 13, 15]  Find: 7
 L           M            R

Step 1: mid = (0+7)/2 = 3, arr[3] = 7 ✓ Found!

Standard Template:
┌─────────────────────────────────────────────────────────────┐
│ def binary_search(arr, target):                             │
│     left, right = 0, len(arr) - 1                           │
│                                                             │
│     while left <= right:                  # Note: <=        │
│         mid = left + (right - left) // 2  # Avoid overflow  │
│                                                             │
│         if arr[mid] == target:                              │
│             return mid                                      │
│         elif arr[mid] < target:                             │
│             left = mid + 1                                  │
│         else:                                               │
│             right = mid - 1                                 │
│                                                             │
│     return -1  # Not found                                  │
└─────────────────────────────────────────────────────────────┘
```

### Binary Search Variants

```python
# 1. FIND FIRST OCCURRENCE (Lower Bound)
def first_occurrence(arr, target):
    left, right = 0, len(arr) - 1
    result = -1

    while left <= right:
        mid = left + (right - left) // 2
        if arr[mid] == target:
            result = mid
            right = mid - 1  # Keep searching left
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return result

# 2. FIND LAST OCCURRENCE (Upper Bound)
def last_occurrence(arr, target):
    left, right = 0, len(arr) - 1
    result = -1

    while left <= right:
        mid = left + (right - left) // 2
        if arr[mid] == target:
            result = mid
            left = mid + 1  # Keep searching right
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return result

# 3. FIND INSERT POSITION
def search_insert(arr, target):
    left, right = 0, len(arr)

    while left < right:
        mid = left + (right - left) // 2
        if arr[mid] < target:
            left = mid + 1
        else:
            right = mid

    return left

# 4. SEARCH IN ROTATED SORTED ARRAY
def search_rotated(arr, target):
    left, right = 0, len(arr) - 1

    while left <= right:
        mid = left + (right - left) // 2

        if arr[mid] == target:
            return mid

        # Left half is sorted
        if arr[left] <= arr[mid]:
            if arr[left] <= target < arr[mid]:
                right = mid - 1
            else:
                left = mid + 1
        # Right half is sorted
        else:
            if arr[mid] < target <= arr[right]:
                left = mid + 1
            else:
                right = mid - 1

    return -1

# 5. FIND PEAK ELEMENT
def find_peak(arr):
    left, right = 0, len(arr) - 1

    while left < right:
        mid = left + (right - left) // 2
        if arr[mid] > arr[mid + 1]:
            right = mid  # Peak is on left (including mid)
        else:
            left = mid + 1  # Peak is on right

    return left

# 6. BINARY SEARCH ON ANSWER (Template)
def binary_search_answer(low, high, is_valid):
    """Find minimum valid answer"""
    while low < high:
        mid = low + (high - low) // 2
        if is_valid(mid):
            high = mid  # Try smaller
        else:
            low = mid + 1
    return low
```

### Binary Search on Answer Pattern

```
Problem: Minimum capacity to ship packages in D days

packages = [1,2,3,4,5,6,7,8,9,10], D = 5

Binary search the answer (capacity):
- Low = max(packages) = 10  (must fit largest package)
- High = sum(packages) = 55 (ship all in one day)

Check if capacity works:
┌──────────────────────────────────────────────────────────────┐
│ def can_ship(packages, D, capacity):                         │
│     days = 1                                                 │
│     current_load = 0                                         │
│                                                              │
│     for pkg in packages:                                     │
│         if current_load + pkg > capacity:                    │
│             days += 1                                        │
│             current_load = 0                                 │
│         current_load += pkg                                  │
│                                                              │
│     return days <= D                                         │
└──────────────────────────────────────────────────────────────┘

Binary search finds minimum capacity = 15
```

---

## 12. RECURSION & BACKTRACKING MENTAL MODEL

### Recursion Anatomy

```
RECURSION = Base Case + Recursive Case

                    factorial(4)
                         │
                    4 × factorial(3)
                              │
                         3 × factorial(2)
                                   │
                              2 × factorial(1)
                                        │
                                   1 × factorial(0)
                                             │
                                        return 1  ← Base case
                                        │
                                   return 1 × 1 = 1
                                   │
                              return 2 × 1 = 2
                              │
                         return 3 × 2 = 6
                         │
                    return 4 × 6 = 24

Call Stack (LIFO):
┌─────────────┐
│ factorial(0)│ ← Top (returns first)
├─────────────┤
│ factorial(1)│
├─────────────┤
│ factorial(2)│
├─────────────┤
│ factorial(3)│
├─────────────┤
│ factorial(4)│ ← Bottom (called first)
└─────────────┘
```

### Recursion Patterns

```python
# 1. LINEAR RECURSION
def factorial(n):
    if n <= 1:           # Base case
        return 1
    return n * factorial(n - 1)  # Recursive case

# 2. BINARY RECURSION (Tree recursion)
def fibonacci(n):
    if n <= 1:           # Base case
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)  # Two recursive calls

# 3. TAIL RECURSION (Optimizable)
def factorial_tail(n, accumulator=1):
    if n <= 1:
        return accumulator
    return factorial_tail(n - 1, n * accumulator)  # Result passed along

# 4. DIVIDE AND CONQUER
def merge_sort(arr):
    if len(arr) <= 1:    # Base case
        return arr
    mid = len(arr) // 2
    left = merge_sort(arr[:mid])    # Divide
    right = merge_sort(arr[mid:])   # Divide
    return merge(left, right)        # Conquer
```

### Backtracking Framework

```
BACKTRACKING = Build solution incrementally, abandon paths that can't lead to solution

                        []
                      / | \
                    [1] [2] [3]         ← Choose
                   / \   |
                [1,2][1,3][2,3]         ← Explore
                 |     |    |
              [1,2,3][1,3,2][2,3,1]     ← Continue

              At each step: Choose → Explore → Unchoose (Backtrack)

TEMPLATE:
┌──────────────────────────────────────────────────────────────────┐
│ def backtrack(candidates, path, result):                         │
│     if is_solution(path):           # Base case                  │
│         result.append(path[:])      # Add copy of solution       │
│         return                                                   │
│                                                                  │
│     for candidate in candidates:                                 │
│         if is_valid(candidate):     # Pruning                    │
│             path.append(candidate)  # Choose                     │
│             backtrack(...)          # Explore                    │
│             path.pop()              # Unchoose (Backtrack)       │
└──────────────────────────────────────────────────────────────────┘
```

### Classic Backtracking Problems

```python
# 1. PERMUTATIONS
def permutations(nums):
    result = []

    def backtrack(path, remaining):
        if not remaining:
            result.append(path[:])
            return

        for i in range(len(remaining)):
            path.append(remaining[i])
            backtrack(path, remaining[:i] + remaining[i+1:])
            path.pop()

    backtrack([], nums)
    return result

# 2. COMBINATIONS (n choose k)
def combinations(n, k):
    result = []

    def backtrack(start, path):
        if len(path) == k:
            result.append(path[:])
            return

        for i in range(start, n + 1):
            path.append(i)
            backtrack(i + 1, path)
            path.pop()

    backtrack(1, [])
    return result

# 3. SUBSETS
def subsets(nums):
    result = []

    def backtrack(start, path):
        result.append(path[:])  # Every path is a valid subset

        for i in range(start, len(nums)):
            path.append(nums[i])
            backtrack(i + 1, path)
            path.pop()

    backtrack(0, [])
    return result

# 4. N-QUEENS
def solve_n_queens(n):
    result = []

    def backtrack(row, cols, diag1, diag2, board):
        if row == n:
            result.append([''.join(r) for r in board])
            return

        for col in range(n):
            if col in cols or (row-col) in diag1 or (row+col) in diag2:
                continue

            board[row][col] = 'Q'
            backtrack(row + 1, cols | {col}, diag1 | {row-col},
                     diag2 | {row+col}, board)
            board[row][col] = '.'

    board = [['.' for _ in range(n)] for _ in range(n)]
    backtrack(0, set(), set(), set(), board)
    return result

# 5. WORD SEARCH (Grid backtracking)
def word_search(board, word):
    rows, cols = len(board), len(board[0])

    def backtrack(r, c, idx):
        if idx == len(word):
            return True
        if (r < 0 or r >= rows or c < 0 or c >= cols or
            board[r][c] != word[idx]):
            return False

        temp = board[r][c]
        board[r][c] = '#'  # Mark visited

        found = (backtrack(r+1, c, idx+1) or backtrack(r-1, c, idx+1) or
                 backtrack(r, c+1, idx+1) or backtrack(r, c-1, idx+1))

        board[r][c] = temp  # Restore
        return found

    for i in range(rows):
        for j in range(cols):
            if backtrack(i, j, 0):
                return True
    return False
```

---

## 13. DYNAMIC PROGRAMMING MENTAL MODEL

### DP Core Concept

```
DYNAMIC PROGRAMMING = Optimal Substructure + Overlapping Subproblems

Fibonacci without DP:                Fibonacci with DP:
         fib(5)                      fib(5) → memo[5] = ?
        /      \                            = fib(4) + fib(3)
    fib(4)     fib(3)                       = memo[4] + memo[3]
    /   \      /    \                       = 3 + 2 = 5
 fib(3) fib(2) fib(2) fib(1)
   ...    ...   ...                  memo = {0:0, 1:1, 2:1, 3:2, 4:3, 5:5}

Time: O(2ⁿ)                          Time: O(n)
Repeated calculations!               Each subproblem solved once!
```

### Two DP Approaches

```
1. TOP-DOWN (Memoization):
   - Start from main problem
   - Recursively solve subproblems
   - Cache results

   def fib(n, memo={}):
       if n in memo:
           return memo[n]
       if n <= 1:
           return n
       memo[n] = fib(n-1, memo) + fib(n-2, memo)
       return memo[n]

2. BOTTOM-UP (Tabulation):
   - Start from smallest subproblems
   - Iteratively build to main problem
   - Use table/array

   def fib(n):
       if n <= 1:
           return n
       dp = [0] * (n + 1)
       dp[1] = 1
       for i in range(2, n + 1):
           dp[i] = dp[i-1] + dp[i-2]
       return dp[n]

   Space-optimized:
   def fib(n):
       if n <= 1:
           return n
       prev, curr = 0, 1
       for _ in range(2, n + 1):
           prev, curr = curr, prev + curr
       return curr
```

### DP Pattern Recognition

```
┌────────────────────────────────────────────────────────────────────┐
│ PATTERN                  │ PROBLEMS                                │
├────────────────────────────────────────────────────────────────────┤
│ 1D DP                    │ Climbing stairs, House robber,          │
│                          │ Coin change (min coins)                 │
├────────────────────────────────────────────────────────────────────┤
│ 2D DP (Grid)             │ Unique paths, Minimum path sum,         │
│                          │ Dungeon game                            │
├────────────────────────────────────────────────────────────────────┤
│ 2D DP (Two sequences)    │ LCS, Edit distance, Interleaving        │
│                          │ strings                                 │
├────────────────────────────────────────────────────────────────────┤
│ Interval DP              │ Matrix chain multiplication,            │
│                          │ Burst balloons, Palindrome partition    │
├────────────────────────────────────────────────────────────────────┤
│ Knapsack                 │ 0/1 Knapsack, Unbounded knapsack,       │
│                          │ Partition equal subset sum              │
├────────────────────────────────────────────────────────────────────┤
│ State Machine DP         │ Best time to buy/sell stock,            │
│                          │ Regex matching                          │
└────────────────────────────────────────────────────────────────────┘
```

### Classic DP Solutions

```python
# 1. CLIMBING STAIRS (1D DP)
def climb_stairs(n):
    if n <= 2:
        return n
    prev, curr = 1, 2
    for _ in range(3, n + 1):
        prev, curr = curr, prev + curr
    return curr

# 2. COIN CHANGE (Unbounded Knapsack)
def coin_change(coins, amount):
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0

    for coin in coins:
        for x in range(coin, amount + 1):
            dp[x] = min(dp[x], dp[x - coin] + 1)

    return dp[amount] if dp[amount] != float('inf') else -1

# 3. LONGEST COMMON SUBSEQUENCE (2D - Two sequences)
def lcs(text1, text2):
    m, n = len(text1), len(text2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i-1] == text2[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])

    return dp[m][n]

# 4. EDIT DISTANCE (2D - Two sequences)
def min_distance(word1, word2):
    m, n = len(word1), len(word2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]

    # Base cases
    for i in range(m + 1):
        dp[i][0] = i
    for j in range(n + 1):
        dp[0][j] = j

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if word1[i-1] == word2[j-1]:
                dp[i][j] = dp[i-1][j-1]
            else:
                dp[i][j] = 1 + min(dp[i-1][j],    # Delete
                                   dp[i][j-1],    # Insert
                                   dp[i-1][j-1])  # Replace

    return dp[m][n]

# 5. 0/1 KNAPSACK
def knapsack(weights, values, capacity):
    n = len(weights)
    dp = [[0] * (capacity + 1) for _ in range(n + 1)]

    for i in range(1, n + 1):
        for w in range(capacity + 1):
            if weights[i-1] <= w:
                dp[i][w] = max(dp[i-1][w],  # Don't take
                              dp[i-1][w-weights[i-1]] + values[i-1])  # Take
            else:
                dp[i][w] = dp[i-1][w]

    return dp[n][capacity]

# 6. LONGEST INCREASING SUBSEQUENCE
def lis(nums):
    if not nums:
        return 0

    # O(n²) DP
    dp = [1] * len(nums)
    for i in range(1, len(nums)):
        for j in range(i):
            if nums[j] < nums[i]:
                dp[i] = max(dp[i], dp[j] + 1)
    return max(dp)

    # O(n log n) with binary search
    # tails[i] = smallest tail element for LIS of length i+1

# 7. UNIQUE PATHS (Grid DP)
def unique_paths(m, n):
    dp = [[1] * n for _ in range(m)]

    for i in range(1, m):
        for j in range(1, n):
            dp[i][j] = dp[i-1][j] + dp[i][j-1]

    return dp[m-1][n-1]

# 8. HOUSE ROBBER (State Machine)
def rob(nums):
    if not nums:
        return 0

    prev_no = 0   # Max if previous not robbed
    prev_yes = 0  # Max if previous robbed

    for num in nums:
        curr_no = max(prev_no, prev_yes)
        curr_yes = prev_no + num
        prev_no, prev_yes = curr_no, curr_yes

    return max(prev_no, prev_yes)
```

### DP Problem Solving Framework

```
Step 1: Define State
  - What information do I need to describe a subproblem?
  - dp[i] = ? or dp[i][j] = ?

Step 2: Define Transition
  - How do I get from smaller subproblems to larger?
  - dp[i] = f(dp[i-1], dp[i-2], ...)

Step 3: Define Base Cases
  - What are the smallest subproblems I can solve directly?
  - dp[0] = ?, dp[1] = ?

Step 4: Define Answer
  - Which subproblem gives me the final answer?
  - Usually dp[n] or max(dp) or dp[n][m]

Step 5: Optimize Space (if needed)
  - Can I use only previous row/values instead of full table?
```

---

## 14. GREEDY ALGORITHMS MENTAL MODEL

### Greedy Concept

```
GREEDY = Make locally optimal choice at each step
         Hope it leads to globally optimal solution

Key Insight: Works when:
1. Greedy Choice Property: Local optimum leads to global optimum
2. Optimal Substructure: Optimal solution contains optimal sub-solutions

Example: Coin Change (Greedy FAILS!)
Coins: [1, 3, 4], Target: 6

Greedy: 4 + 1 + 1 = 3 coins  ✗ (picks largest first)
Optimal: 3 + 3 = 2 coins     ✓

Greedy works for: US coins [1, 5, 10, 25] because larger coins
are multiples or near-multiples of smaller ones.
```

### Classic Greedy Problems

```python
# 1. ACTIVITY SELECTION / MEETING ROOMS
def max_activities(intervals):
    """Maximum non-overlapping intervals"""
    intervals.sort(key=lambda x: x[1])  # Sort by end time
    count = 0
    end = float('-inf')

    for start, finish in intervals:
        if start >= end:
            count += 1
            end = finish

    return count

# 2. MINIMUM PLATFORMS / MEETING ROOMS II
def min_meeting_rooms(intervals):
    starts = sorted([i[0] for i in intervals])
    ends = sorted([i[1] for i in intervals])

    rooms = 0
    max_rooms = 0
    s = e = 0

    while s < len(intervals):
        if starts[s] < ends[e]:
            rooms += 1
            s += 1
        else:
            rooms -= 1
            e += 1
        max_rooms = max(max_rooms, rooms)

    return max_rooms

# 3. JUMP GAME
def can_jump(nums):
    max_reach = 0
    for i, jump in enumerate(nums):
        if i > max_reach:
            return False
        max_reach = max(max_reach, i + jump)
    return True

# 4. JUMP GAME II (Minimum jumps)
def min_jumps(nums):
    jumps = 0
    current_end = 0
    farthest = 0

    for i in range(len(nums) - 1):
        farthest = max(farthest, i + nums[i])
        if i == current_end:
            jumps += 1
            current_end = farthest

    return jumps

# 5. GAS STATION
def can_complete_circuit(gas, cost):
    total_tank = 0
    current_tank = 0
    start = 0

    for i in range(len(gas)):
        total_tank += gas[i] - cost[i]
        current_tank += gas[i] - cost[i]

        if current_tank < 0:
            start = i + 1
            current_tank = 0

    return start if total_tank >= 0 else -1

# 6. TASK SCHEDULER
def least_interval(tasks, n):
    counts = Counter(tasks).values()
    max_count = max(counts)
    max_count_tasks = list(counts).count(max_count)

    # Minimum slots needed
    return max(len(tasks), (max_count - 1) * (n + 1) + max_count_tasks)

# 7. PARTITION LABELS
def partition_labels(s):
    last = {c: i for i, c in enumerate(s)}
    result = []
    start = end = 0

    for i, c in enumerate(s):
        end = max(end, last[c])
        if i == end:
            result.append(end - start + 1)
            start = i + 1

    return result
```

### Greedy vs DP Decision

```
┌─────────────────────────────────────────────────────────────────┐
│ Use GREEDY when:                                                │
│  • Local optimal leads to global optimal (provable)             │
│  • Problem has greedy choice property                           │
│  • Simpler, often O(n) or O(n log n)                           │
│                                                                 │
│ Use DP when:                                                    │
│  • Greedy doesn't work (easy to find counterexample)            │
│  • Need to explore multiple paths/choices                       │
│  • Overlapping subproblems exist                                │
│                                                                 │
│ Common Greedy Problems:                                         │
│  • Interval scheduling                                          │
│  • Huffman coding                                               │
│  • Fractional knapsack                                          │
│  • Minimum spanning tree (Kruskal's, Prim's)                   │
│  • Dijkstra's shortest path                                     │
│  • Activity selection                                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 15. TWO POINTERS & SLIDING WINDOW

### Two Pointers Patterns

```
1. OPPOSITE ENDS (Sorted array, palindrome)
   ┌───┬───┬───┬───┬───┬───┬───┐
   │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │
   └───┴───┴───┴───┴───┴───┴───┘
     ▲                       ▲
     L ─────────────────────► R

   Use: Two sum (sorted), container with most water, valid palindrome

2. SAME DIRECTION (Fast-slow, read-write)
   ┌───┬───┬───┬───┬───┬───┬───┐
   │ 0 │ 1 │ 0 │ 3 │ 0 │ 5 │ 0 │  Remove zeros
   └───┴───┴───┴───┴───┴───┴───┘
     ▲   ▲
     W   R ─────────────────────►

   Use: Remove duplicates, move zeros, linked list cycle
```

### Two Pointers Solutions

```python
# 1. TWO SUM II (Sorted array)
def two_sum_sorted(nums, target):
    left, right = 0, len(nums) - 1
    while left < right:
        total = nums[left] + nums[right]
        if total == target:
            return [left + 1, right + 1]
        elif total < target:
            left += 1
        else:
            right -= 1
    return []

# 2. THREE SUM
def three_sum(nums):
    nums.sort()
    result = []

    for i in range(len(nums) - 2):
        if i > 0 and nums[i] == nums[i-1]:
            continue

        left, right = i + 1, len(nums) - 1
        while left < right:
            total = nums[i] + nums[left] + nums[right]
            if total == 0:
                result.append([nums[i], nums[left], nums[right]])
                while left < right and nums[left] == nums[left+1]:
                    left += 1
                while left < right and nums[right] == nums[right-1]:
                    right -= 1
                left += 1
                right -= 1
            elif total < 0:
                left += 1
            else:
                right -= 1

    return result

# 3. CONTAINER WITH MOST WATER
def max_area(height):
    left, right = 0, len(height) - 1
    max_water = 0

    while left < right:
        water = min(height[left], height[right]) * (right - left)
        max_water = max(max_water, water)

        if height[left] < height[right]:
            left += 1
        else:
            right -= 1

    return max_water

# 4. REMOVE DUPLICATES (Read-Write pointers)
def remove_duplicates(nums):
    if not nums:
        return 0

    write = 1
    for read in range(1, len(nums)):
        if nums[read] != nums[read - 1]:
            nums[write] = nums[read]
            write += 1

    return write
```

### Sliding Window Patterns

```
FIXED SIZE WINDOW:
┌───┬───┬───┬───┬───┬───┬───┐
│ 1 │ 3 │ 2 │ 6 │ 4 │ 8 │ 2 │   Window size k=3
└───┴───┴───┴───┴───┴───┴───┘
 └─────────┘
 sum=6    ──► slide ──► sum = sum - arr[i] + arr[i+k]

VARIABLE SIZE WINDOW:
┌───┬───┬───┬───┬───┬───┬───┐
│ 2 │ 1 │ 5 │ 2 │ 3 │ 2 │   │   Min subarray with sum ≥ 7
└───┴───┴───┴───┴───┴───┴───┘
     └─────────┘
     L         R

Expand R until condition met, then shrink L while condition holds
```

### Sliding Window Solutions

```python
# 1. MAX SUM SUBARRAY OF SIZE K (Fixed window)
def max_sum_subarray(nums, k):
    window_sum = sum(nums[:k])
    max_sum = window_sum

    for i in range(k, len(nums)):
        window_sum += nums[i] - nums[i - k]
        max_sum = max(max_sum, window_sum)

    return max_sum

# 2. MINIMUM SIZE SUBARRAY SUM (Variable window)
def min_subarray_len(target, nums):
    left = 0
    current_sum = 0
    min_len = float('inf')

    for right in range(len(nums)):
        current_sum += nums[right]

        while current_sum >= target:
            min_len = min(min_len, right - left + 1)
            current_sum -= nums[left]
            left += 1

    return min_len if min_len != float('inf') else 0

# 3. LONGEST SUBSTRING WITHOUT REPEATING CHARACTERS
def length_of_longest_substring(s):
    char_set = set()
    left = 0
    max_len = 0

    for right in range(len(s)):
        while s[right] in char_set:
            char_set.remove(s[left])
            left += 1
        char_set.add(s[right])
        max_len = max(max_len, right - left + 1)

    return max_len

# 4. MINIMUM WINDOW SUBSTRING
def min_window(s, t):
    if not t or not s:
        return ""

    need = Counter(t)
    have = {}
    required = len(need)
    formed = 0
    left = 0
    min_len = float('inf')
    result = ""

    for right in range(len(s)):
        char = s[right]
        have[char] = have.get(char, 0) + 1

        if char in need and have[char] == need[char]:
            formed += 1

        while formed == required:
            if right - left + 1 < min_len:
                min_len = right - left + 1
                result = s[left:right + 1]

            have[s[left]] -= 1
            if s[left] in need and have[s[left]] < need[s[left]]:
                formed -= 1
            left += 1

    return result

# 5. SLIDING WINDOW TEMPLATE
def sliding_window_template(s):
    """Template for most sliding window problems"""
    left = 0
    result = 0
    state = {}  # Track window state (counts, set, etc.)

    for right in range(len(s)):
        # 1. Expand: Add s[right] to window state

        # 2. Contract: While window is invalid
        while invalid_condition:
            # Remove s[left] from window state
            left += 1

        # 3. Update: Calculate result from current valid window
        result = max(result, right - left + 1)

    return result
```

---

## 16. PROBLEM PATTERN RECOGNITION

### Pattern Identification Cheat Sheet

```
┌────────────────────────────────────────────────────────────────────┐
│ IF YOU SEE...                    │ THINK...                        │
├────────────────────────────────────────────────────────────────────┤
│ "Sorted array"                   │ Binary Search, Two Pointers     │
│ "Find pair/triplet with sum"     │ Two Pointers, Hash Map          │
│ "Maximum/minimum subarray"       │ Sliding Window, Kadane's        │
│ "Substring with condition"       │ Sliding Window + Hash Map       │
│ "Linked list cycle"              │ Fast-Slow Pointers              │
│ "Tree traversal"                 │ DFS (recursive), BFS (queue)    │
│ "Level by level"                 │ BFS                             │
│ "Shortest path (unweighted)"     │ BFS                             │
│ "Shortest path (weighted)"       │ Dijkstra                        │
│ "All permutations/combinations"  │ Backtracking                    │
│ "Count ways to..."               │ Dynamic Programming             │
│ "Maximum/minimum ways"           │ Dynamic Programming             │
│ "Overlapping intervals"          │ Sort + Greedy                   │
│ "Top K elements"                 │ Heap, Quick Select              │
│ "Frequency counting"             │ Hash Map                        │
│ "String matching"                │ KMP, Rabin-Karp                 │
│ "Prefix/suffix"                  │ Trie, Prefix Sum                │
│ "Range queries"                  │ Segment Tree, BIT               │
│ "Connected components"           │ Union-Find, DFS                 │
│ "Detect cycle in graph"          │ DFS with colors                 │
│ "Topological ordering"           │ Topological Sort                │
│ "Parentheses/brackets"           │ Stack                           │
│ "Next greater/smaller"           │ Monotonic Stack                 │
│ "Median in stream"               │ Two Heaps                       │
│ "LRU Cache"                      │ Hash Map + Doubly Linked List   │
└────────────────────────────────────────────────────────────────────┘
```

### Problem Solving Framework

```
STEP 1: UNDERSTAND
  □ Read problem twice
  □ Identify inputs and outputs
  □ Note constraints (array size, value range)
  □ Work through examples manually

STEP 2: PATTERN MATCH
  □ What data structure fits?
  □ What algorithm pattern?
  □ Similar problems I've solved?

STEP 3: PLAN
  □ Write pseudocode
  □ Identify edge cases
  □ Estimate time/space complexity

STEP 4: IMPLEMENT
  □ Code solution
  □ Use meaningful variable names
  □ Handle edge cases

STEP 5: VERIFY
  □ Trace through with examples
  □ Check edge cases
  □ Verify complexity matches plan
```

### Complexity Targets by Constraint

```
┌─────────────────────────────────────────────────────────────────┐
│ N (Input Size)        │ Target Time Complexity                  │
├─────────────────────────────────────────────────────────────────┤
│ n ≤ 10                │ O(n!), O(2ⁿ)                           │
│ n ≤ 20                │ O(2ⁿ)                                   │
│ n ≤ 100               │ O(n³)                                   │
│ n ≤ 1,000             │ O(n²)                                   │
│ n ≤ 10,000            │ O(n²) with small constant               │
│ n ≤ 100,000           │ O(n log n)                              │
│ n ≤ 1,000,000         │ O(n), O(n log n)                        │
│ n ≤ 10,000,000        │ O(n)                                    │
│ n > 10,000,000        │ O(log n), O(1)                          │
└─────────────────────────────────────────────────────────────────┘

Rule of thumb: ~10⁸ operations per second
```

---

## 17. QUICK REFERENCE CARD

### Essential Code Templates

```python
# BINARY SEARCH
def binary_search(arr, target):
    left, right = 0, len(arr) - 1
    while left <= right:
        mid = left + (right - left) // 2
        if arr[mid] == target: return mid
        elif arr[mid] < target: left = mid + 1
        else: right = mid - 1
    return -1

# BFS
def bfs(graph, start):
    visited = {start}
    queue = deque([start])
    while queue:
        node = queue.popleft()
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)

# DFS
def dfs(graph, node, visited=set()):
    visited.add(node)
    for neighbor in graph[node]:
        if neighbor not in visited:
            dfs(graph, neighbor, visited)

# BACKTRACKING
def backtrack(path, choices):
    if is_solution(path):
        results.append(path[:])
        return
    for choice in choices:
        if is_valid(choice):
            path.append(choice)
            backtrack(path, remaining_choices)
            path.pop()

# SLIDING WINDOW
def sliding_window(arr, k):
    window = sum(arr[:k])
    max_sum = window
    for i in range(k, len(arr)):
        window += arr[i] - arr[i-k]
        max_sum = max(max_sum, window)
    return max_sum

# TWO POINTERS
def two_pointer(arr, target):
    left, right = 0, len(arr) - 1
    while left < right:
        current = arr[left] + arr[right]
        if current == target: return [left, right]
        elif current < target: left += 1
        else: right -= 1
    return []
```

### Python Built-in Shortcuts

```python
from collections import Counter, defaultdict, deque
from heapq import heappush, heappop, heapify
from bisect import bisect_left, bisect_right
from functools import lru_cache
import math

# Counter
freq = Counter([1,1,2,3,3,3])  # {3:3, 1:2, 2:1}
freq.most_common(2)             # [(3,3), (1,2)]

# defaultdict
graph = defaultdict(list)
graph['a'].append('b')          # No KeyError

# deque (O(1) both ends)
dq = deque([1,2,3])
dq.appendleft(0)
dq.popleft()

# heapq (min-heap)
heap = [3,1,4]
heapify(heap)
heappush(heap, 2)
smallest = heappop(heap)

# bisect (binary search)
arr = [1,2,4,4,5]
bisect_left(arr, 4)   # 2 (leftmost position)
bisect_right(arr, 4)  # 4 (rightmost position + 1)

# lru_cache (memoization)
@lru_cache(maxsize=None)
def fib(n):
    if n <= 1: return n
    return fib(n-1) + fib(n-2)

# Useful math
math.gcd(12, 18)       # 6
math.lcm(4, 6)         # 12 (Python 3.9+)
math.ceil(7/3)         # 3
math.floor(7/3)        # 2
float('inf')           # Infinity
```

---

## 18. LEARNING PATH

### Phase 1: Foundations (Week 1-2)
```
□ Big O Notation
  - Time and space complexity
  - Best, average, worst cases
  - Common growth rates

□ Arrays & Strings
  - Two pointers technique
  - Sliding window
  - Prefix sum

□ Hash Tables
  - Implementation concepts
  - Collision handling
  - Common patterns (frequency count, two sum)
```

### Phase 2: Linear Structures (Week 3-4)
```
□ Linked Lists
  - Single and doubly linked
  - Fast-slow pointers
  - Reversal techniques

□ Stacks & Queues
  - Implementation
  - Monotonic stack
  - BFS applications
```

### Phase 3: Trees & Heaps (Week 5-6)
```
□ Binary Trees
  - Traversals (pre, in, post, level)
  - Recursive thinking
  - Common patterns (height, paths, LCA)

□ Binary Search Trees
  - BST property
  - Search, insert, delete
  - Validation

□ Heaps
  - Min/max heap
  - Priority queue
  - K-th element problems
```

### Phase 4: Graphs (Week 7-8)
```
□ Graph Fundamentals
  - Representations
  - BFS and DFS
  - Connected components

□ Advanced Graphs
  - Topological sort
  - Shortest path (Dijkstra)
  - Union-Find
```

### Phase 5: Advanced Techniques (Week 9-12)
```
□ Recursion & Backtracking
  - Recursive thinking
  - Permutations, combinations
  - Constraint satisfaction

□ Dynamic Programming
  - Memoization vs tabulation
  - Common patterns
  - State definition

□ Sorting & Searching
  - Sorting algorithms
  - Binary search variants
  - Binary search on answer
```

### Practice Strategy

```
DAILY ROUTINE:
1. Solve 2-3 problems
2. If stuck > 20 min, look at solution
3. Implement solution yourself
4. Review and optimize
5. Document patterns learned

PROBLEM SOURCES:
- LeetCode (categorized by topic)
- HackerRank (fundamentals)
- Codeforces (competitive)
- AlgoExpert (guided learning)

DIFFICULTY PROGRESSION:
Week 1-4:  70% Easy, 30% Medium
Week 5-8:  30% Easy, 60% Medium, 10% Hard
Week 9-12: 10% Easy, 50% Medium, 40% Hard
```

---

## FINAL WISDOM

```
┌─────────────────────────────────────────────────────────────────┐
│                     THE DSA MINDSET                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. PATTERNS OVER PROBLEMS                                      │
│     Learn 15 patterns, solve 150 problems                       │
│     vs Learn 150 solutions, solve 15 variants                   │
│                                                                 │
│  2. BRUTE FORCE FIRST                                          │
│     Working solution > No solution                              │
│     Optimize after correctness                                  │
│                                                                 │
│  3. CONSTRAINTS ARE HINTS                                       │
│     n ≤ 20 → Exponential OK                                     │
│     n ≤ 10000 → O(n²) OK                                        │
│     n ≤ 10⁶ → Need O(n) or O(n log n)                          │
│                                                                 │
│  4. THINK BEFORE CODING                                         │
│     5 min planning saves 30 min debugging                       │
│                                                                 │
│  5. PRACTICE CONSISTENTLY                                       │
│     1 hour daily > 7 hours once a week                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

*"First, solve the problem. Then, write the code." - John Johnson*
