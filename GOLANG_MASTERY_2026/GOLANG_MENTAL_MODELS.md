# Go (Golang) Mental Models for Mastery

## 1. The Go Philosophy - Simplicity is Complicated

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         THE GO PHILOSOPHY                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   "Clear is better than clever"                                             │
│   "A little copying is better than a little dependency"                     │
│   "Don't communicate by sharing memory, share memory by communicating"      │
│                                                                              │
│   GO'S DESIGN PRINCIPLES:                                                   │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ ✓ Simplicity         - Less is more, one way to do things          │   │
│   │ ✓ Readability        - Code is read more than written              │   │
│   │ ✓ Fast compilation   - Quick feedback loop                         │   │
│   │ ✓ Concurrency first  - Goroutines and channels built-in           │   │
│   │ ✓ Strong typing      - Catch errors at compile time               │   │
│   │ ✓ Composition        - No inheritance, use interfaces             │   │
│   │ ✓ Explicit           - No hidden magic, what you see is what runs │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   WHAT GO INTENTIONALLY LACKS:                                              │
│   ✗ Classes/Inheritance  → Use composition and interfaces                   │
│   ✗ Exceptions           → Use explicit error returns                       │
│   ✗ Generics (until 1.18)→ Now has type parameters                         │
│   ✗ Operator overloading → Methods only                                     │
│   ✗ Default parameters   → Use functional options pattern                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. The Type System - Everything Has a Type

```
                    ┌──────────────────────────────────────┐
                    │         GO TYPE HIERARCHY            │
                    └──────────────────────────────────────┘

                              ALL TYPES
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        ▼                         ▼                         ▼
    BASIC TYPES            COMPOSITE TYPES            REFERENCE-LIKE
        │                         │                         │
   ┌────┴────┐            ┌───────┴───────┐          ┌──────┴──────┐
   ▼         ▼            ▼               ▼          ▼             ▼
 Numeric   Other        Value          Reference   Interfaces   Functions
   │         │          Types           Types
   │         │            │               │
   │    bool, string   struct         slice
   │                   array          map
 int, int8/16/32/64                   channel
 uint, uint8/16/32/64                 pointer
 float32/64
 complex64/128
 byte (uint8)
 rune (int32)


ZERO VALUES (Every type has one!):
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Type              Zero Value        Example                               │
│   ─────────────     ──────────        ───────                              │
│   int/float         0                 var x int      // x == 0             │
│   bool              false             var b bool     // b == false         │
│   string            ""                var s string   // s == ""            │
│   pointer           nil               var p *int     // p == nil           │
│   slice             nil               var s []int    // s == nil           │
│   map               nil               var m map[..]  // m == nil           │
│   channel           nil               var c chan int // c == nil           │
│   interface         nil               var i error    // i == nil           │
│   struct            {zero values}     var s MyStruct // fields zeroed      │
│   array             [zero values]     var a [3]int   // [0, 0, 0]         │
│                                                                             │
│   KEY INSIGHT: You can always use a zero value. No null pointer surprises  │
│   (except for nil maps and channels - they need make())                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


TYPE DECLARATIONS vs TYPE ALIASES:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   TYPE DEFINITION (New distinct type):                                     │
│   type UserID int                                                          │
│   var id UserID = 42                                                       │
│   // id is NOT interchangeable with int without explicit conversion        │
│   // Can add methods to UserID                                             │
│                                                                             │
│   TYPE ALIAS (Same type, different name):                                  │
│   type ID = int                                                            │
│   var id ID = 42                                                           │
│   // id IS interchangeable with int                                        │
│   // Cannot add methods (it's the same type)                               │
│                                                                             │
│   Common aliases:                                                          │
│   • byte = uint8                                                           │
│   • rune = int32                                                           │
│   • any  = interface{} (Go 1.18+)                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Slices - The Most Important Data Structure

```
                    ┌──────────────────────────────────────┐
                    │        SLICE MENTAL MODEL            │
                    └──────────────────────────────────────┘


SLICE INTERNALS:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   A slice is a WINDOW into a backing array                                 │
│                                                                             │
│   Slice Header (24 bytes on 64-bit):                                       │
│   ┌─────────────────────────────────────┐                                  │
│   │  ptr  │  len  │  cap  │                                                │
│   │  (8)  │  (8)  │  (8)  │                                                │
│   └───┬───┴───────┴───────┘                                                │
│       │                                                                     │
│       ▼ points to                                                          │
│   ┌───────────────────────────────────────────┐                            │
│   │ [0] │ [1] │ [2] │ [3] │ [4] │ [5] │ [6] │  ← Backing Array            │
│   └───────────────────────────────────────────┘                            │
│       └─────────────┘                                                       │
│            len=3                                                            │
│       └─────────────────────────────────┘                                   │
│                      cap=7                                                  │
│                                                                             │
│   s := []int{10, 20, 30, 40, 50}                                           │
│   s2 := s[1:3]  // s2 = [20, 30], len=2, cap=4                             │
│                 // s2 SHARES backing array with s!                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


SLICE OPERATIONS VISUALIZED:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   APPEND (may or may not reallocate):                                      │
│                                                                             │
│   Case 1: Capacity available (no reallocation)                             │
│   ┌─────────────────────────────────────────┐                              │
│   │ [1] │ [2] │ [3] │ [ ] │ [ ] │           │  cap=5, len=3                │
│   └─────────────────────────────────────────┘                              │
│   append(s, 4) →                                                           │
│   ┌─────────────────────────────────────────┐                              │
│   │ [1] │ [2] │ [3] │ [4] │ [ ] │           │  cap=5, len=4                │
│   └─────────────────────────────────────────┘                              │
│   Same backing array!                                                      │
│                                                                             │
│   Case 2: No capacity (reallocation)                                       │
│   ┌─────────────────────────────────────────┐                              │
│   │ [1] │ [2] │ [3] │                       │  cap=3, len=3                │
│   └─────────────────────────────────────────┘                              │
│   append(s, 4) →                                                           │
│   ┌─────────────────────────────────────────────────────────────┐          │
│   │ [1] │ [2] │ [3] │ [4] │ [ ] │ [ ] │     │  cap=6, len=4    │          │
│   └─────────────────────────────────────────────────────────────┘          │
│   NEW backing array! (typically 2x capacity)                               │
│                                                                             │
│   GOTCHA: Always use s = append(s, x) because append may return new slice  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


SLICE GOTCHA - SHARED BACKING ARRAY:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   original := []int{1, 2, 3, 4, 5}                                         │
│   slice := original[1:3]  // [2, 3]                                        │
│                                                                             │
│   slice[0] = 99  // MODIFIES original too!                                 │
│   // original is now [1, 99, 3, 4, 5]                                      │
│                                                                             │
│   SOLUTION - Make a copy:                                                  │
│   slice := make([]int, 2)                                                  │
│   copy(slice, original[1:3])                                               │
│   // Now slice is independent                                              │
│                                                                             │
│   Or use full slice expression:                                            │
│   slice := original[1:3:3]  // cap limited to 2                            │
│   // append to slice will reallocate, not affect original                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


SLICE PATTERNS:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   // Pre-allocate when size known (performance)                            │
│   s := make([]int, 0, 1000)  // len=0, cap=1000                            │
│                                                                             │
│   // Delete element at index i                                             │
│   s = append(s[:i], s[i+1:]...)                                            │
│                                                                             │
│   // Insert element at index i                                             │
│   s = append(s[:i], append([]T{x}, s[i:]...)...)                           │
│                                                                             │
│   // Filter in place                                                       │
│   n := 0                                                                   │
│   for _, x := range s {                                                    │
│       if keep(x) {                                                         │
│           s[n] = x                                                         │
│           n++                                                              │
│       }                                                                    │
│   }                                                                        │
│   s = s[:n]                                                                │
│                                                                             │
│   // Clear slice (reuse backing array)                                     │
│   s = s[:0]                                                                │
│   // Or with Go 1.21+: clear(s)                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Maps - The Hash Table

```
                    ┌──────────────────────────────────────┐
                    │          MAP MENTAL MODEL            │
                    └──────────────────────────────────────┘


MAP BASICS:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Declaration and Initialization:                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │ // Nil map (read OK, write PANIC!)                                 │  │
│   │ var m map[string]int         // m == nil                           │  │
│   │ _ = m["key"]                 // OK, returns 0                      │  │
│   │ m["key"] = 1                 // PANIC!                             │  │
│   │                                                                     │  │
│   │ // Initialized maps                                                │  │
│   │ m = make(map[string]int)    // Empty but usable                    │  │
│   │ m = make(map[string]int, 100) // With hint for 100 entries        │  │
│   │ m = map[string]int{"a": 1}  // Literal                             │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   The Comma-OK Idiom:                                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │ value := m["key"]           // Returns zero value if not found     │  │
│   │                                                                     │  │
│   │ value, ok := m["key"]       // ok=true if found, ok=false if not   │  │
│   │ if ok {                                                            │  │
│   │     // key exists                                                  │  │
│   │ }                                                                  │  │
│   │                                                                     │  │
│   │ // Idiomatic check                                                 │  │
│   │ if value, ok := m["key"]; ok {                                     │  │
│   │     // use value                                                   │  │
│   │ }                                                                  │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


MAP ITERATION ORDER:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   MAPS ARE UNORDERED!                                                      │
│                                                                             │
│   m := map[string]int{"a": 1, "b": 2, "c": 3}                              │
│                                                                             │
│   for k, v := range m {                                                    │
│       fmt.Println(k, v)                                                    │
│   }                                                                        │
│   // Run 1: a 1, c 3, b 2                                                  │
│   // Run 2: b 2, a 1, c 3                                                  │
│   // Order is RANDOMIZED intentionally!                                    │
│                                                                             │
│   NEED SORTED ORDER? Sort keys first:                                      │
│   keys := make([]string, 0, len(m))                                        │
│   for k := range m {                                                       │
│       keys = append(keys, k)                                               │
│   }                                                                        │
│   sort.Strings(keys)                                                       │
│   for _, k := range keys {                                                 │
│       fmt.Println(k, m[k])                                                 │
│   }                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


MAP CONCURRENCY:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   MAPS ARE NOT GOROUTINE-SAFE!                                             │
│                                                                             │
│   // WRONG - Data race                                                     │
│   m := make(map[string]int)                                                │
│   go func() { m["a"] = 1 }()                                               │
│   go func() { _ = m["a"] }()  // RACE!                                     │
│                                                                             │
│   SOLUTIONS:                                                               │
│                                                                             │
│   1. sync.Mutex (general purpose):                                         │
│   var mu sync.Mutex                                                        │
│   mu.Lock()                                                                │
│   m["a"] = 1                                                               │
│   mu.Unlock()                                                              │
│                                                                             │
│   2. sync.RWMutex (read-heavy):                                            │
│   var mu sync.RWMutex                                                      │
│   mu.RLock()    // Multiple readers OK                                     │
│   _ = m["a"]                                                               │
│   mu.RUnlock()                                                             │
│                                                                             │
│   3. sync.Map (built-in concurrent map):                                   │
│   var m sync.Map                                                           │
│   m.Store("a", 1)                                                          │
│   v, ok := m.Load("a")                                                     │
│   // Good for: write-once-read-many, disjoint key sets                     │
│   // Not good for: general concurrent access                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Interfaces - The Power of Go

```
                    ┌──────────────────────────────────────┐
                    │       INTERFACE MENTAL MODEL         │
                    └──────────────────────────────────────┘


INTERFACE SATISFACTION IS IMPLICIT:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   // Interface definition                                                  │
│   type Writer interface {                                                  │
│       Write([]byte) (int, error)                                           │
│   }                                                                        │
│                                                                             │
│   // Types automatically satisfy if they have the methods                  │
│   type File struct { ... }                                                 │
│   func (f *File) Write(b []byte) (int, error) { ... }  // File is Writer  │
│                                                                             │
│   type Buffer struct { ... }                                               │
│   func (b *Buffer) Write(p []byte) (int, error) { ... } // Buffer is Writer│
│                                                                             │
│   // No "implements" keyword needed!                                       │
│   var w Writer = &File{}   // Works!                                       │
│   var w Writer = &Buffer{} // Works!                                       │
│                                                                             │
│   KEY INSIGHT: Interfaces are satisfied implicitly                         │
│   • Types don't know about interfaces they satisfy                         │
│   • Decouples packages (no import needed)                                  │
│   • Add interfaces after the fact                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


INTERFACE INTERNALS:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Interface value = (type, value) pair                                     │
│                                                                             │
│   ┌─────────────────────────────────────────┐                              │
│   │   Interface Variable                    │                              │
│   │   ┌─────────────┬─────────────┐         │                              │
│   │   │    type     │    value    │         │                              │
│   │   │   (itable)  │  (pointer)  │         │                              │
│   │   └──────┬──────┴──────┬──────┘         │                              │
│   │          │             │                │                              │
│   │          ▼             ▼                │                              │
│   │    Type metadata   Actual data          │                              │
│   │    + method table                       │                              │
│   └─────────────────────────────────────────┘                              │
│                                                                             │
│   NIL INTERFACE vs INTERFACE WITH NIL VALUE:                               │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │ var w Writer           // Nil interface: (nil, nil)                │  │
│   │ w == nil               // true                                     │  │
│   │                                                                     │  │
│   │ var f *File = nil      // Nil pointer                              │  │
│   │ var w Writer = f       // Interface with nil value: (*File, nil)   │  │
│   │ w == nil               // FALSE! (type is not nil)                 │  │
│   │                                                                     │  │
│   │ GOTCHA: This catches many people!                                  │  │
│   │ func getWriter() Writer {                                          │  │
│   │     var f *File = nil                                              │  │
│   │     return f        // Returns (*File, nil), NOT nil interface!    │  │
│   │ }                                                                  │  │
│   │ getWriter() == nil  // false!                                      │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


TYPE ASSERTIONS AND TYPE SWITCHES:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   TYPE ASSERTION:                                                          │
│   var i interface{} = "hello"                                              │
│                                                                             │
│   // Risky (panics if wrong type)                                          │
│   s := i.(string)         // s = "hello"                                   │
│   n := i.(int)            // PANIC!                                        │
│                                                                             │
│   // Safe (use comma-ok)                                                   │
│   s, ok := i.(string)     // s = "hello", ok = true                        │
│   n, ok := i.(int)        // n = 0, ok = false                             │
│                                                                             │
│   TYPE SWITCH:                                                             │
│   func describe(i interface{}) {                                           │
│       switch v := i.(type) {                                               │
│       case int:                                                            │
│           fmt.Printf("int: %d\n", v)                                       │
│       case string:                                                         │
│           fmt.Printf("string: %s\n", v)                                    │
│       case nil:                                                            │
│           fmt.Println("nil")                                               │
│       default:                                                             │
│           fmt.Printf("unknown: %T\n", v)                                   │
│       }                                                                    │
│   }                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


INTERFACE DESIGN PRINCIPLES:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   "The bigger the interface, the weaker the abstraction"                   │
│                                                                             │
│   GOOD - Small interfaces:                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ type Reader interface { Read(p []byte) (n int, err error) }        │  │
│   │ type Writer interface { Write(p []byte) (n int, err error) }       │  │
│   │ type Closer interface { Close() error }                            │  │
│   │                                                                     │  │
│   │ // Compose when needed                                             │  │
│   │ type ReadWriter interface { Reader; Writer }                       │  │
│   │ type ReadWriteCloser interface { Reader; Writer; Closer }          │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   ACCEPT INTERFACES, RETURN STRUCTS:                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │ // Good: Accept interface (flexible for callers)                   │  │
│   │ func Process(r io.Reader) error { ... }                            │  │
│   │                                                                     │  │
│   │ // Good: Return concrete type (clear for users)                    │  │
│   │ func NewServer() *Server { ... }                                   │  │
│   │                                                                     │  │
│   │ // Avoid: Return interface (hides implementation)                  │  │
│   │ func NewServer() ServerInterface { ... }  // Usually wrong         │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   DEFINE INTERFACES WHERE THEY'RE USED:                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │ // Package db provides storage                                     │  │
│   │ type Store struct { ... }                                          │  │
│   │ func (s *Store) Get(key string) (string, error) { ... }            │  │
│   │ func (s *Store) Set(key, value string) error { ... }               │  │
│   │                                                                     │  │
│   │ // Package handler uses storage (defines its own interface)        │  │
│   │ type KeyGetter interface {                                         │  │
│   │     Get(key string) (string, error)                                │  │
│   │ }                                                                  │  │
│   │ func NewHandler(kg KeyGetter) *Handler { ... }                     │  │
│   │ // Can accept *db.Store or any mock!                               │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Goroutines & Channels - Concurrency Model

```
                    ┌──────────────────────────────────────┐
                    │        CONCURRENCY MENTAL MODEL      │
                    └──────────────────────────────────────┘


GOROUTINE = LIGHTWEIGHT THREAD:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   OS Thread (1-8 MB stack)          Goroutine (2 KB stack, grows)          │
│   ┌─────────────────────┐           ┌──────────┐                           │
│   │ █████████████████████│           │ ██ │ grows as needed               │
│   │ █████████████████████│           └──────────┘                           │
│   │ █████████████████████│                                                  │
│   │ █████████████████████│           Can run millions of goroutines!       │
│   └─────────────────────┘                                                  │
│                                                                             │
│   Go Scheduler (M:N scheduling):                                           │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │   Goroutines (G)     ─────►    Processors (P)    ─────►   OS Threads (M)
│   │   [G] [G] [G] [G]              [P] [P] [P]               [M] [M] [M] │  │
│   │   [G] [G] [G] [G]                  │                                │  │
│   │   [G] [G] [G] [G]              Run queues                           │  │
│   │                                                                     │  │
│   │   GOMAXPROCS = number of P's (default = num CPU cores)             │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   STARTING GOROUTINES:                                                     │
│   go doSomething()                    // Function call                     │
│   go func() { ... }()                 // Anonymous function                │
│   go func(x int) { ... }(value)       // With parameters (COPY value)     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


CHANNELS = COMMUNICATION:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   UNBUFFERED CHANNEL (Synchronous):                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │   ch := make(chan int)                                             │  │
│   │                                                                     │  │
│   │   Sender          Channel         Receiver                         │  │
│   │   ──────          ───────         ────────                         │  │
│   │   ch <- 42   →    [   ]    →      <-ch                             │  │
│   │   (blocks)        (empty)         (blocks)                         │  │
│   │                                                                     │  │
│   │   Both must be ready at the same time (rendezvous)                 │  │
│   │   • Send blocks until receiver ready                               │  │
│   │   • Receive blocks until sender ready                              │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   BUFFERED CHANNEL (Asynchronous up to capacity):                          │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │   ch := make(chan int, 3)    // Buffer size 3                      │  │
│   │                                                                     │  │
│   │   Sender          Channel              Receiver                    │  │
│   │   ──────          ───────              ────────                    │  │
│   │   ch <- 1    →    [ 1 |   |   ]                                    │  │
│   │   ch <- 2    →    [ 1 | 2 |   ]                                    │  │
│   │   ch <- 3    →    [ 1 | 2 | 3 ]   →    <-ch returns 1             │  │
│   │   ch <- 4    →    (BLOCKS - full)      <-ch returns 2             │  │
│   │                                                                     │  │
│   │   • Send blocks only when buffer full                              │  │
│   │   • Receive blocks only when buffer empty                          │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


CHANNEL DIRECTIONS:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   ch := make(chan int)        // Bidirectional                             │
│                                                                             │
│   func send(ch chan<- int)    // Send-only (can only ch <- x)             │
│   func recv(ch <-chan int)    // Receive-only (can only <-ch)             │
│                                                                             │
│   USE: Enforce correct usage in function signatures                        │
│                                                                             │
│   func producer(out chan<- int) {                                          │
│       out <- 42                                                            │
│       // <-out  // Compile error! Can't receive                            │
│   }                                                                        │
│                                                                             │
│   func consumer(in <-chan int) {                                           │
│       x := <-in                                                            │
│       // in <- 1  // Compile error! Can't send                             │
│   }                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


SELECT STATEMENT:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   select = switch for channels (blocks until one case ready)               │
│                                                                             │
│   select {                                                                 │
│   case v := <-ch1:                                                         │
│       // Received v from ch1                                               │
│   case ch2 <- x:                                                           │
│       // Sent x to ch2                                                     │
│   case <-time.After(1 * time.Second):                                      │
│       // Timeout after 1 second                                            │
│   default:                                                                 │
│       // No channel ready, don't block (non-blocking select)               │
│   }                                                                        │
│                                                                             │
│   COMMON PATTERNS:                                                         │
│                                                                             │
│   // Timeout                                                               │
│   select {                                                                 │
│   case result := <-ch:                                                     │
│       return result                                                        │
│   case <-time.After(5 * time.Second):                                      │
│       return errors.New("timeout")                                         │
│   }                                                                        │
│                                                                             │
│   // Cancellation with context                                             │
│   select {                                                                 │
│   case result := <-ch:                                                     │
│       return result, nil                                                   │
│   case <-ctx.Done():                                                       │
│       return nil, ctx.Err()                                                │
│   }                                                                        │
│                                                                             │
│   // Non-blocking send/receive                                             │
│   select {                                                                 │
│   case ch <- x:                                                            │
│       // Sent                                                              │
│   default:                                                                 │
│       // Channel full, drop or handle                                      │
│   }                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Concurrency Patterns

```
                    ┌──────────────────────────────────────┐
                    │       CONCURRENCY PATTERNS           │
                    └──────────────────────────────────────┘


WORKER POOL:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   ┌─────────┐     ┌─────────────────────────┐     ┌─────────┐             │
│   │  Jobs   │ ──► │  Worker  Worker  Worker │ ──► │ Results │             │
│   │ Channel │     │    1       2       3    │     │ Channel │             │
│   └─────────┘     └─────────────────────────┘     └─────────┘             │
│                                                                             │
│   func worker(id int, jobs <-chan Job, results chan<- Result) {            │
│       for job := range jobs {          // Range until channel closed       │
│           results <- process(job)                                          │
│       }                                                                    │
│   }                                                                        │
│                                                                             │
│   func main() {                                                            │
│       jobs := make(chan Job, 100)                                          │
│       results := make(chan Result, 100)                                    │
│                                                                             │
│       // Start workers                                                     │
│       for w := 1; w <= 3; w++ {                                            │
│           go worker(w, jobs, results)                                      │
│       }                                                                    │
│                                                                             │
│       // Send jobs                                                         │
│       for _, job := range jobList {                                        │
│           jobs <- job                                                      │
│       }                                                                    │
│       close(jobs)  // Signal no more jobs                                  │
│                                                                             │
│       // Collect results                                                   │
│       for range jobList {                                                  │
│           result := <-results                                              │
│       }                                                                    │
│   }                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


FAN-OUT / FAN-IN:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   FAN-OUT: One channel → Multiple goroutines                               │
│                                                                             │
│               ┌─────► [Worker 1] ───┐                                      │
│   [Input] ────┼─────► [Worker 2] ───┼────► [Output]                        │
│               └─────► [Worker 3] ───┘                                      │
│                                                                             │
│   FAN-IN: Multiple channels → One channel                                  │
│                                                                             │
│   func fanIn(channels ...<-chan int) <-chan int {                          │
│       out := make(chan int)                                                │
│       var wg sync.WaitGroup                                                │
│       wg.Add(len(channels))                                                │
│                                                                             │
│       for _, ch := range channels {                                        │
│           go func(c <-chan int) {                                          │
│               defer wg.Done()                                              │
│               for v := range c {                                           │
│                   out <- v                                                 │
│               }                                                            │
│           }(ch)                                                            │
│       }                                                                    │
│                                                                             │
│       go func() {                                                          │
│           wg.Wait()                                                        │
│           close(out)                                                       │
│       }()                                                                  │
│                                                                             │
│       return out                                                           │
│   }                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


PIPELINE:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   [Generate] ──► [Square] ──► [Filter] ──► [Print]                        │
│       │              │            │            │                           │
│   chan int      chan int     chan int     consume                          │
│                                                                             │
│   func generate(nums ...int) <-chan int {                                  │
│       out := make(chan int)                                                │
│       go func() {                                                          │
│           for _, n := range nums {                                         │
│               out <- n                                                     │
│           }                                                                │
│           close(out)                                                       │
│       }()                                                                  │
│       return out                                                           │
│   }                                                                        │
│                                                                             │
│   func square(in <-chan int) <-chan int {                                  │
│       out := make(chan int)                                                │
│       go func() {                                                          │
│           for n := range in {                                              │
│               out <- n * n                                                 │
│           }                                                                │
│           close(out)                                                       │
│       }()                                                                  │
│       return out                                                           │
│   }                                                                        │
│                                                                             │
│   // Usage                                                                 │
│   nums := generate(2, 3, 4)                                                │
│   squared := square(nums)                                                  │
│   for n := range squared {                                                 │
│       fmt.Println(n)  // 4, 9, 16                                          │
│   }                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


GRACEFUL SHUTDOWN:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   func main() {                                                            │
│       // Setup signal handling                                             │
│       ctx, cancel := context.WithCancel(context.Background())              │
│       defer cancel()                                                       │
│                                                                             │
│       sigCh := make(chan os.Signal, 1)                                     │
│       signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)                │
│                                                                             │
│       // Start workers                                                     │
│       var wg sync.WaitGroup                                                │
│       for i := 0; i < numWorkers; i++ {                                    │
│           wg.Add(1)                                                        │
│           go func() {                                                      │
│               defer wg.Done()                                              │
│               worker(ctx)                                                  │
│           }()                                                              │
│       }                                                                    │
│                                                                             │
│       // Wait for signal                                                   │
│       <-sigCh                                                              │
│       fmt.Println("Shutting down...")                                      │
│                                                                             │
│       // Signal workers to stop                                            │
│       cancel()                                                             │
│                                                                             │
│       // Wait for workers with timeout                                     │
│       done := make(chan struct{})                                          │
│       go func() { wg.Wait(); close(done) }()                               │
│                                                                             │
│       select {                                                             │
│       case <-done:                                                         │
│           fmt.Println("Clean shutdown")                                    │
│       case <-time.After(10 * time.Second):                                 │
│           fmt.Println("Forced shutdown")                                   │
│       }                                                                    │
│   }                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Error Handling - The Go Way

```
                    ┌──────────────────────────────────────┐
                    │       ERROR HANDLING MODEL           │
                    └──────────────────────────────────────┘


ERRORS ARE VALUES:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   type error interface {                                                   │
│       Error() string                                                       │
│   }                                                                        │
│                                                                             │
│   // Creating errors                                                       │
│   err := errors.New("something went wrong")                                │
│   err := fmt.Errorf("failed to open %s: %w", filename, originalErr)        │
│                                                                             │
│   // The canonical pattern                                                 │
│   result, err := doSomething()                                             │
│   if err != nil {                                                          │
│       return fmt.Errorf("doSomething failed: %w", err)                     │
│   }                                                                        │
│   // Use result...                                                         │
│                                                                             │
│   RULE: Check errors immediately, handle or return                         │
│   RULE: Never ignore errors (except explicitly with _)                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


ERROR WRAPPING (Go 1.13+):
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   WRAPPING ADDS CONTEXT:                                                   │
│                                                                             │
│   func readConfig(path string) (*Config, error) {                          │
│       data, err := os.ReadFile(path)                                       │
│       if err != nil {                                                      │
│           return nil, fmt.Errorf("reading config %s: %w", path, err)       │
│       }                                                                    │
│                                                                             │
│       var cfg Config                                                       │
│       if err := json.Unmarshal(data, &cfg); err != nil {                   │
│           return nil, fmt.Errorf("parsing config: %w", err)                │
│       }                                                                    │
│       return &cfg, nil                                                     │
│   }                                                                        │
│                                                                             │
│   Error chain:                                                             │
│   "reading config /etc/app.json: open /etc/app.json: no such file"        │
│         └─────── our context ──────┘ └───── original error ─────┘          │
│                                                                             │
│   UNWRAPPING:                                                              │
│   errors.Is(err, os.ErrNotExist)  // Check if error IS a specific error   │
│   errors.As(err, &pathError)       // Extract specific error type          │
│                                                                             │
│   // errors.Is example                                                     │
│   if errors.Is(err, os.ErrNotExist) {                                      │
│       // Handle file not found                                             │
│   }                                                                        │
│                                                                             │
│   // errors.As example                                                     │
│   var pathErr *os.PathError                                                │
│   if errors.As(err, &pathErr) {                                            │
│       fmt.Println("Path:", pathErr.Path)                                   │
│   }                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


SENTINEL ERRORS vs CUSTOM ERROR TYPES:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   SENTINEL ERRORS (predefined error values):                               │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │ var ErrNotFound = errors.New("not found")                          │  │
│   │ var ErrInvalidInput = errors.New("invalid input")                  │  │
│   │                                                                     │  │
│   │ func Get(id string) (*Item, error) {                               │  │
│   │     if id == "" {                                                  │  │
│   │         return nil, ErrInvalidInput                                │  │
│   │     }                                                              │  │
│   │     item := db.Find(id)                                            │  │
│   │     if item == nil {                                               │  │
│   │         return nil, ErrNotFound                                    │  │
│   │     }                                                              │  │
│   │     return item, nil                                               │  │
│   │ }                                                                  │  │
│   │                                                                     │  │
│   │ // Usage                                                           │  │
│   │ if errors.Is(err, ErrNotFound) { ... }                             │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   CUSTOM ERROR TYPES (when you need more info):                            │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                     │  │
│   │ type ValidationError struct {                                      │  │
│   │     Field   string                                                 │  │
│   │     Message string                                                 │  │
│   │ }                                                                  │  │
│   │                                                                     │  │
│   │ func (e *ValidationError) Error() string {                         │  │
│   │     return fmt.Sprintf("%s: %s", e.Field, e.Message)               │  │
│   │ }                                                                  │  │
│   │                                                                     │  │
│   │ // Usage                                                           │  │
│   │ var valErr *ValidationError                                        │  │
│   │ if errors.As(err, &valErr) {                                       │  │
│   │     fmt.Println("Invalid field:", valErr.Field)                    │  │
│   │ }                                                                  │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


PANIC vs ERROR:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   ERROR:  Expected failure conditions (file not found, network timeout)    │
│   PANIC:  Programmer errors, unrecoverable situations                      │
│                                                                             │
│   USE PANIC FOR:                                                           │
│   • Impossible conditions that indicate bugs                               │
│   • During initialization (if setup fails fatally)                         │
│   • Assertion-like checks in development                                   │
│                                                                             │
│   func MustParseTemplate(text string) *template.Template {                 │
│       t, err := template.New("").Parse(text)                               │
│       if err != nil {                                                      │
│           panic(err)  // Programmer error - bad template                   │
│       }                                                                    │
│       return t                                                             │
│   }                                                                        │
│                                                                             │
│   RECOVER FROM PANIC:                                                      │
│   func safeCall() (err error) {                                            │
│       defer func() {                                                       │
│           if r := recover(); r != nil {                                    │
│               err = fmt.Errorf("panic: %v", r)                             │
│           }                                                                │
│       }()                                                                  │
│       riskyFunction()                                                      │
│       return nil                                                           │
│   }                                                                        │
│                                                                             │
│   RULE: Don't panic for normal errors                                      │
│   RULE: recover() only works in deferred functions                         │
│   RULE: Don't use panic/recover for control flow                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Memory Model & Escape Analysis

```
                    ┌──────────────────────────────────────┐
                    │        MEMORY MENTAL MODEL           │
                    └──────────────────────────────────────┘


STACK vs HEAP:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   STACK (fast, automatic)              HEAP (slower, GC managed)           │
│   ┌─────────────────────┐              ┌─────────────────────┐             │
│   │ Local variables     │              │ Escaping variables  │             │
│   │ Function params     │              │ Large allocations   │             │
│   │ Return addresses    │              │ Shared data         │             │
│   │                     │              │                     │             │
│   │ Grows/shrinks auto  │              │ Managed by GC       │             │
│   │ Very fast alloc     │              │ Slower alloc        │             │
│   └─────────────────────┘              └─────────────────────┘             │
│                                                                             │
│   Go decides automatically! (escape analysis)                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


ESCAPE ANALYSIS:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   "Does this variable outlive the function?"                               │
│                                                                             │
│   // STAYS ON STACK (doesn't escape)                                       │
│   func noEscape() int {                                                    │
│       x := 42          // x on stack                                       │
│       return x         // value copied                                     │
│   }                                                                        │
│                                                                             │
│   // ESCAPES TO HEAP                                                       │
│   func escapes() *int {                                                    │
│       x := 42          // x must go to heap!                               │
│       return &x        // pointer returned, x outlives function            │
│   }                                                                        │
│                                                                             │
│   // ESCAPE TO HEAP (passed to interface)                                  │
│   func escapes2() {                                                        │
│       x := 42                                                              │
│       fmt.Println(x)   // fmt.Println takes interface{}, x escapes        │
│   }                                                                        │
│                                                                             │
│   CHECK ESCAPE ANALYSIS:                                                   │
│   go build -gcflags="-m" main.go                                           │
│   # Output shows what escapes and why                                      │
│                                                                             │
│   WHEN VARIABLES ESCAPE:                                                   │
│   • Returned as pointer                                                    │
│   • Stored in interface{}                                                  │
│   • Captured by closure                                                    │
│   • Too large for stack                                                    │
│   • Size unknown at compile time (variable-length slice)                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


REDUCING ALLOCATIONS:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   1. PRE-ALLOCATE SLICES:                                                  │
│   // Bad: Multiple reallocations                                           │
│   var s []int                                                              │
│   for i := 0; i < 1000; i++ {                                              │
│       s = append(s, i)                                                     │
│   }                                                                        │
│                                                                             │
│   // Good: Single allocation                                               │
│   s := make([]int, 0, 1000)                                                │
│   for i := 0; i < 1000; i++ {                                              │
│       s = append(s, i)                                                     │
│   }                                                                        │
│                                                                             │
│   2. SYNC.POOL FOR REUSE:                                                  │
│   var bufPool = sync.Pool{                                                 │
│       New: func() interface{} {                                            │
│           return new(bytes.Buffer)                                         │
│       },                                                                   │
│   }                                                                        │
│                                                                             │
│   buf := bufPool.Get().(*bytes.Buffer)                                     │
│   buf.Reset()                                                              │
│   // use buf...                                                            │
│   bufPool.Put(buf)                                                         │
│                                                                             │
│   3. PASS VALUES FOR SMALL STRUCTS:                                        │
│   // Small struct - pass by value (stays on stack)                         │
│   type Point struct { X, Y int }                                           │
│   func distance(p1, p2 Point) float64 { ... }                              │
│                                                                             │
│   4. USE STRINGS.BUILDER FOR CONCATENATION:                                │
│   // Bad                                                                   │
│   s := ""                                                                  │
│   for _, word := range words {                                             │
│       s += word + " "  // New string each iteration!                       │
│   }                                                                        │
│                                                                             │
│   // Good                                                                  │
│   var b strings.Builder                                                    │
│   for _, word := range words {                                             │
│       b.WriteString(word)                                                  │
│       b.WriteString(" ")                                                   │
│   }                                                                        │
│   s := b.String()                                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Context - Request-Scoped Data & Cancellation

```
                    ┌──────────────────────────────────────┐
                    │         CONTEXT MENTAL MODEL         │
                    └──────────────────────────────────────┘


CONTEXT IS FOR:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   1. CANCELLATION    - Stop work when no longer needed                     │
│   2. DEADLINES       - Timeout after duration                              │
│   3. REQUEST VALUES  - Carry request-scoped data (trace IDs, auth)         │
│                                                                             │
│   Context flows DOWN the call stack:                                       │
│                                                                             │
│   [HTTP Handler] ──ctx──► [Service] ──ctx──► [Repository] ──ctx──► [DB]   │
│        │                      │                    │                       │
│        └──────────────────────┴────────────────────┘                       │
│                 If handler cancelled, ALL work stops                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


CONTEXT TREE:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│                    context.Background()                                    │
│                           │                                                 │
│              ┌────────────┼────────────┐                                   │
│              ▼            ▼            ▼                                   │
│         WithCancel   WithTimeout  WithDeadline                             │
│              │            │            │                                   │
│         child ctx    child ctx    child ctx                                │
│              │                                                              │
│         WithValue                                                          │
│              │                                                              │
│         child ctx with value                                               │
│                                                                             │
│   RULE: Cancelling parent cancels ALL children                             │
│   RULE: Values only visible to children, not parents                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


CONTEXT PATTERNS:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   CANCELLATION:                                                            │
│   ctx, cancel := context.WithCancel(context.Background())                  │
│   defer cancel()  // Always call cancel to release resources               │
│                                                                             │
│   go func() {                                                              │
│       select {                                                             │
│       case <-ctx.Done():                                                   │
│           return  // Context cancelled                                     │
│       case result := <-doWork():                                           │
│           // Use result                                                    │
│       }                                                                    │
│   }()                                                                      │
│                                                                             │
│   TIMEOUT:                                                                 │
│   ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)  │
│   defer cancel()                                                           │
│                                                                             │
│   result, err := doWorkWithContext(ctx)                                    │
│   if err == context.DeadlineExceeded {                                     │
│       // Timed out                                                         │
│   }                                                                        │
│                                                                             │
│   REQUEST VALUES (use sparingly!):                                         │
│   type contextKey string                                                   │
│   const userIDKey contextKey = "userID"                                    │
│                                                                             │
│   ctx = context.WithValue(ctx, userIDKey, "user-123")                      │
│   userID := ctx.Value(userIDKey).(string)                                  │
│                                                                             │
│   RULES FOR VALUES:                                                        │
│   • Use custom types for keys (avoid collisions)                           │
│   • Only for request-scoped data, NOT for passing parameters               │
│   • Data should be immutable                                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


CONTEXT BEST PRACTICES:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   1. First parameter: func DoSomething(ctx context.Context, ...) error     │
│   2. Never store context in structs (pass as parameter)                    │
│   3. Never pass nil context (use context.TODO() if unsure)                 │
│   4. Always call cancel() (defer cancel())                                 │
│   5. Don't use context.Value for required parameters                       │
│   6. Keep value keys package-private (type contextKey string)              │
│                                                                             │
│   CHECKING CANCELLATION:                                                   │
│   func doWork(ctx context.Context) error {                                 │
│       for i := 0; i < 1000; i++ {                                          │
│           select {                                                         │
│           case <-ctx.Done():                                               │
│               return ctx.Err()  // Return early                            │
│           default:                                                         │
│           }                                                                │
│           // Do one unit of work                                           │
│       }                                                                    │
│       return nil                                                           │
│   }                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Testing Patterns

```
                    ┌──────────────────────────────────────┐
                    │         TESTING MENTAL MODEL         │
                    └──────────────────────────────────────┘


TABLE-DRIVEN TESTS (The Go Way):
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   func TestAdd(t *testing.T) {                                             │
│       tests := []struct {                                                  │
│           name     string                                                  │
│           a, b     int                                                     │
│           expected int                                                     │
│       }{                                                                   │
│           {"positive", 2, 3, 5},                                           │
│           {"negative", -1, -2, -3},                                        │
│           {"zero", 0, 0, 0},                                               │
│           {"mixed", -1, 5, 4},                                             │
│       }                                                                    │
│                                                                             │
│       for _, tt := range tests {                                           │
│           t.Run(tt.name, func(t *testing.T) {                              │
│               result := Add(tt.a, tt.b)                                    │
│               if result != tt.expected {                                   │
│                   t.Errorf("Add(%d, %d) = %d, want %d",                     │
│                       tt.a, tt.b, result, tt.expected)                     │
│               }                                                            │
│           })                                                               │
│       }                                                                    │
│   }                                                                        │
│                                                                             │
│   WHY TABLE-DRIVEN:                                                        │
│   • Easy to add new cases                                                  │
│   • DRY - test logic written once                                          │
│   • Clear what's being tested                                              │
│   • Subtests can run in parallel                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


TESTING WITH INTERFACES (Dependency Injection):
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   // Production code - define interface where used                         │
│   type UserStore interface {                                               │
│       GetUser(id string) (*User, error)                                    │
│   }                                                                        │
│                                                                             │
│   type UserService struct {                                                │
│       store UserStore                                                      │
│   }                                                                        │
│                                                                             │
│   func (s *UserService) GetUserName(id string) (string, error) {           │
│       user, err := s.store.GetUser(id)                                     │
│       if err != nil {                                                      │
│           return "", err                                                   │
│       }                                                                    │
│       return user.Name, nil                                                │
│   }                                                                        │
│                                                                             │
│   // Test code - mock implementation                                       │
│   type mockStore struct {                                                  │
│       user *User                                                           │
│       err  error                                                           │
│   }                                                                        │
│                                                                             │
│   func (m *mockStore) GetUser(id string) (*User, error) {                  │
│       return m.user, m.err                                                 │
│   }                                                                        │
│                                                                             │
│   func TestGetUserName(t *testing.T) {                                     │
│       mock := &mockStore{user: &User{Name: "Alice"}}                       │
│       svc := &UserService{store: mock}                                     │
│                                                                             │
│       name, err := svc.GetUserName("123")                                  │
│       if err != nil {                                                      │
│           t.Fatal(err)                                                     │
│       }                                                                    │
│       if name != "Alice" {                                                 │
│           t.Errorf("got %q, want %q", name, "Alice")                       │
│       }                                                                    │
│   }                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


HTTPTEST FOR HTTP HANDLERS:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   func TestHelloHandler(t *testing.T) {                                    │
│       // Create request                                                    │
│       req := httptest.NewRequest("GET", "/hello?name=Go", nil)             │
│       // Create response recorder                                          │
│       rec := httptest.NewRecorder()                                        │
│                                                                             │
│       // Call handler                                                      │
│       HelloHandler(rec, req)                                               │
│                                                                             │
│       // Check response                                                    │
│       if rec.Code != http.StatusOK {                                       │
│           t.Errorf("status = %d, want %d", rec.Code, http.StatusOK)        │
│       }                                                                    │
│       if rec.Body.String() != "Hello, Go!" {                               │
│           t.Errorf("body = %q", rec.Body.String())                         │
│       }                                                                    │
│   }                                                                        │
│                                                                             │
│   // Testing against a real server                                         │
│   func TestAPI(t *testing.T) {                                             │
│       srv := httptest.NewServer(http.HandlerFunc(MyHandler))               │
│       defer srv.Close()                                                    │
│                                                                             │
│       resp, err := http.Get(srv.URL + "/api/users")                        │
│       // Check resp...                                                     │
│   }                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


BENCHMARKS:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   func BenchmarkFibonacci(b *testing.B) {                                  │
│       for i := 0; i < b.N; i++ {                                           │
│           Fibonacci(20)                                                    │
│       }                                                                    │
│   }                                                                        │
│                                                                             │
│   // Run: go test -bench=. -benchmem                                       │
│   // Output:                                                               │
│   // BenchmarkFibonacci-8   5000000   250 ns/op   0 B/op   0 allocs/op    │
│   //                        ^ runs    ^ time      ^ memory  ^ allocations │
│                                                                             │
│   // Reset timer after setup                                               │
│   func BenchmarkWithSetup(b *testing.B) {                                  │
│       data := expensiveSetup()                                             │
│       b.ResetTimer()  // Don't count setup time                            │
│       for i := 0; i < b.N; i++ {                                           │
│           process(data)                                                    │
│       }                                                                    │
│   }                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 12. Common Patterns & Idioms

```
                    ┌──────────────────────────────────────┐
                    │          GO IDIOMS & PATTERNS        │
                    └──────────────────────────────────────┘


FUNCTIONAL OPTIONS PATTERN:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   type Server struct {                                                     │
│       host    string                                                       │
│       port    int                                                          │
│       timeout time.Duration                                                │
│   }                                                                        │
│                                                                             │
│   type Option func(*Server)                                                │
│                                                                             │
│   func WithHost(host string) Option {                                      │
│       return func(s *Server) { s.host = host }                             │
│   }                                                                        │
│                                                                             │
│   func WithPort(port int) Option {                                         │
│       return func(s *Server) { s.port = port }                             │
│   }                                                                        │
│                                                                             │
│   func WithTimeout(d time.Duration) Option {                               │
│       return func(s *Server) { s.timeout = d }                             │
│   }                                                                        │
│                                                                             │
│   func NewServer(opts ...Option) *Server {                                 │
│       s := &Server{                                                        │
│           host:    "localhost",  // defaults                               │
│           port:    8080,                                                   │
│           timeout: 30 * time.Second,                                       │
│       }                                                                    │
│       for _, opt := range opts {                                           │
│           opt(s)                                                           │
│       }                                                                    │
│       return s                                                             │
│   }                                                                        │
│                                                                             │
│   // Usage                                                                 │
│   srv := NewServer(                                                        │
│       WithHost("0.0.0.0"),                                                 │
│       WithPort(9000),                                                      │
│   )                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


EMBEDDING FOR COMPOSITION:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   type Logger struct{}                                                     │
│   func (l *Logger) Log(msg string) { fmt.Println(msg) }                    │
│                                                                             │
│   type Server struct {                                                     │
│       *Logger           // Embedded - Server "inherits" Log method         │
│       host string                                                          │
│   }                                                                        │
│                                                                             │
│   srv := &Server{Logger: &Logger{}, host: "localhost"}                     │
│   srv.Log("Starting...")  // Works! Promoted from embedded Logger          │
│                                                                             │
│   // Also works with interfaces                                            │
│   type ReadWriter struct {                                                 │
│       io.Reader                                                            │
│       io.Writer                                                            │
│   }                                                                        │
│                                                                             │
│   NOTE: This is composition, NOT inheritance                               │
│   • Embedded type's methods are "promoted"                                 │
│   • Outer type can override by defining same method                        │
│   • No "super" call - explicitly call embedded type                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


DEFER FOR CLEANUP:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   func ReadFile(path string) ([]byte, error) {                             │
│       f, err := os.Open(path)                                              │
│       if err != nil {                                                      │
│           return nil, err                                                  │
│       }                                                                    │
│       defer f.Close()  // Runs when function returns (LIFO order)          │
│                                                                             │
│       return io.ReadAll(f)                                                 │
│   }                                                                        │
│                                                                             │
│   DEFER GOTCHAS:                                                           │
│                                                                             │
│   // Arguments evaluated immediately!                                      │
│   x := 1                                                                   │
│   defer fmt.Println(x)  // Prints 1, not 2                                 │
│   x = 2                                                                    │
│                                                                             │
│   // Solution: use closure                                                 │
│   x := 1                                                                   │
│   defer func() { fmt.Println(x) }()  // Prints 2                           │
│   x = 2                                                                    │
│                                                                             │
│   // Defer in loop - be careful!                                           │
│   for _, f := range files {                                                │
│       file, _ := os.Open(f)                                                │
│       defer file.Close()  // ALL closes run at function end, not loop end │
│   }                                                                        │
│   // Better: wrap in function                                              │
│   for _, f := range files {                                                │
│       func() {                                                             │
│           file, _ := os.Open(f)                                            │
│           defer file.Close()  // Closes at end of this function            │
│           // process file                                                  │
│       }()                                                                  │
│   }                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


INIT FUNCTIONS:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   var config *Config                                                       │
│                                                                             │
│   func init() {                                                            │
│       // Runs automatically before main()                                  │
│       // Can have multiple init() in same file                             │
│       // Order: imported packages first, then this package                 │
│       config = loadConfig()                                                │
│   }                                                                        │
│                                                                             │
│   WHEN TO USE init():                                                      │
│   • Register drivers/plugins                                               │
│   • Initialize package-level state                                         │
│   • Verify program state                                                   │
│                                                                             │
│   WHEN NOT TO USE:                                                         │
│   • Complex initialization (use explicit New functions)                    │
│   • When initialization can fail (can't return error)                      │
│   • When order matters (hard to control)                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 13. Learning Path

```
                    ┌──────────────────────────────────────┐
                    │        GO MASTERY ROADMAP            │
                    └──────────────────────────────────────┘


PHASE 1: FOUNDATIONS (Week 1-2)
├── Variables, types, zero values
├── Control flow (if, for, switch)
├── Functions, multiple returns
├── Slices and maps (deeply!)
├── Structs and methods
└── Pointers (when and why)

PHASE 2: CORE GO (Week 3-4)
├── Interfaces (the Go way)
├── Error handling patterns
├── Packages and modules
├── Testing (table-driven tests)
├── Standard library (fmt, io, os)
└── JSON encoding/decoding

PHASE 3: CONCURRENCY (Week 5-6)
├── Goroutines
├── Channels (buffered/unbuffered)
├── select statement
├── sync package (Mutex, WaitGroup)
├── Context for cancellation
└── Common concurrency patterns

PHASE 4: PRODUCTION GO (Week 7-8)
├── HTTP servers and clients
├── Database access (database/sql)
├── Configuration management
├── Structured logging (slog)
├── Graceful shutdown
└── Docker and deployment

PHASE 5: ADVANCED (Ongoing)
├── Generics (Go 1.18+)
├── Performance profiling (pprof)
├── Memory optimization
├── Reflection (use sparingly)
├── Code generation
└── Popular frameworks (Gin, gRPC)


PROJECT IDEAS BY LEVEL:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│ Beginner:                                                                  │
│ • CLI todo app                                                             │
│ • File organizer                                                           │
│ • URL shortener (in-memory)                                                │
│                                                                             │
│ Intermediate:                                                              │
│ • REST API with database                                                   │
│ • Concurrent web scraper                                                   │
│ • Chat server (WebSocket)                                                  │
│                                                                             │
│ Advanced:                                                                  │
│ • Distributed cache                                                        │
│ • Custom load balancer                                                     │
│ • Kubernetes operator                                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        GO QUICK REFERENCE                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│ DECLARATION:     var x int   |   x := 42   |   const y = 10                │
│ SLICE:           s := make([]int, len, cap)  |  s = append(s, x)           │
│ MAP:             m := make(map[K]V)  |  v, ok := m[key]                    │
│ STRUCT:          type T struct { Field Type }                              │
│ INTERFACE:       type I interface { Method() }                             │
│ ERROR:           if err != nil { return fmt.Errorf("...: %w", err) }       │
│ GOROUTINE:       go func() { ... }()                                       │
│ CHANNEL:         ch := make(chan T, buffer)  |  ch <- v  |  v := <-ch     │
│ SELECT:          select { case <-ch: ... case <-ctx.Done(): ... }          │
│ CONTEXT:         ctx, cancel := context.WithTimeout(ctx, d); defer cancel()│
│ DEFER:           defer f.Close()  // LIFO, runs on return                  │
│ TEST:            func TestX(t *testing.T) { t.Run("case", func(t *T){}) } │
│                                                                             │
│ PROVERBS:                                                                  │
│ • Don't communicate by sharing memory; share memory by communicating       │
│ • Clear is better than clever                                              │
│ • Accept interfaces, return structs                                        │
│ • The bigger the interface, the weaker the abstraction                     │
│ • Make the zero value useful                                               │
│ • Errors are values                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```
