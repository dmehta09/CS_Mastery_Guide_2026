# JavaScript Mastery Concepts 2026

## Fundamentals
- JavaScript Engine
- V8 Engine
- SpiderMonkey
- JavaScriptCore
- Execution Context
- Global Execution Context
- Function Execution Context
- Eval Execution Context
- Call Stack
- Memory Heap
- Stack Overflow
- Single-Threaded Nature
- Synchronous Execution
- Lexical Environment
- Variable Environment
- Scope Chain
- Hoisting
- Variable Hoisting
- Function Hoisting
- Temporal Dead Zone (TDZ)
- Strict Mode ("use strict")

## Variables & Declarations
- var Keyword
- let Keyword
- const Keyword
- var vs let vs const
- Block Scope
- Function Scope
- Global Scope
- Lexical Scope
- Variable Shadowing
- Re-declaration Rules
- Immutability with const
- Naming Conventions
- Reserved Words
- Identifier Rules

## Data Types

### Primitive Types
- String
- Number
- BigInt
- Boolean
- undefined
- null
- Symbol
- typeof Operator
- Primitive Coercion
- Primitive Wrapper Objects

### Reference Types
- Object
- Array
- Function
- Date
- RegExp
- Map
- Set
- WeakMap
- WeakSet
- ArrayBuffer
- SharedArrayBuffer
- DataView
- TypedArrays

### Type Checking
- typeof Operator
- instanceof Operator
- Object.prototype.toString
- Array.isArray()
- Number.isNaN()
- Number.isFinite()
- Number.isInteger()

### Type Coercion
- Implicit Coercion
- Explicit Coercion
- String Coercion
- Number Coercion
- Boolean Coercion
- ToNumber Rules
- ToString Rules
- ToBoolean Rules
- ToPrimitive
- Truthy Values
- Falsy Values
- Equality Coercion (==)
- Strict Equality (===)
- Object.is()

## Operators

### Arithmetic Operators
- Addition (+)
- Subtraction (-)
- Multiplication (*)
- Division (/)
- Remainder (%)
- Exponentiation (**)
- Increment (++)
- Decrement (--)
- Unary Plus (+)
- Unary Negation (-)

### Assignment Operators
- Basic Assignment (=)
- Addition Assignment (+=)
- Subtraction Assignment (-=)
- Multiplication Assignment (*=)
- Division Assignment (/=)
- Remainder Assignment (%=)
- Exponentiation Assignment (**=)
- Logical AND Assignment (&&=)
- Logical OR Assignment (||=)
- Nullish Coalescing Assignment (??=)
- Bitwise Assignment Operators

### Comparison Operators
- Equal (==)
- Strict Equal (===)
- Not Equal (!=)
- Strict Not Equal (!==)
- Greater Than (>)
- Less Than (<)
- Greater Than or Equal (>=)
- Less Than or Equal (<=)

### Logical Operators
- Logical AND (&&)
- Logical OR (||)
- Logical NOT (!)
- Nullish Coalescing (??)
- Short-Circuit Evaluation
- Double NOT (!!)

### Bitwise Operators
- Bitwise AND (&)
- Bitwise OR (|)
- Bitwise XOR (^)
- Bitwise NOT (~)
- Left Shift (<<)
- Right Shift (>>)
- Unsigned Right Shift (>>>)

### Other Operators
- Ternary Operator (?:)
- Comma Operator (,)
- delete Operator
- in Operator
- typeof Operator
- void Operator
- new Operator
- Spread Operator (...)
- Optional Chaining (?.)
- Grouping Operator ()

## Control Flow

### Conditional Statements
- if Statement
- if...else Statement
- else if Statement
- Nested if Statements
- switch Statement
- case Clause
- default Clause
- break Statement
- Fall-Through Behavior

### Loops
- for Loop
- while Loop
- do...while Loop
- for...in Loop
- for...of Loop
- for await...of Loop
- Loop Labels
- break Statement
- continue Statement
- Nested Loops
- Infinite Loops

### Exception Handling
- try Statement
- catch Statement
- finally Statement
- throw Statement
- Error Object
- Error Types
- EvalError
- RangeError
- ReferenceError
- SyntaxError
- TypeError
- URIError
- AggregateError
- Custom Errors
- Error Stack Trace
- Error Propagation
- Re-throwing Errors

## Functions

### Function Basics
- Function Declaration
- Function Expression
- Named Function Expression
- Anonymous Functions
- Arrow Functions
- Arrow Function Syntax
- Implicit Return
- Concise Body
- Block Body
- IIFE (Immediately Invoked Function Expression)
- Function Constructor
- Generator Functions
- Async Functions

### Parameters & Arguments
- Function Parameters
- Default Parameters
- Rest Parameters (...)
- Spread Syntax in Calls
- arguments Object
- arguments.length
- arguments.callee (deprecated)
- Named Parameters Pattern
- Parameter Destructuring
- Optional Parameters

### Return Values
- return Statement
- Implicit undefined Return
- Returning Objects
- Returning Functions
- Multiple Return Values (Destructuring)
- Early Return Pattern

### Function Properties & Methods
- function.name
- function.length
- function.prototype
- Function.prototype.call()
- Function.prototype.apply()
- Function.prototype.bind()
- Function.prototype.toString()

### Advanced Function Concepts
- First-Class Functions
- Higher-Order Functions
- Callback Functions
- Pure Functions
- Impure Functions
- Side Effects
- Function Composition
- Function Currying
- Partial Application
- Memoization
- Recursion
- Tail Recursion
- Trampolining
- Thunks

## Scope & Closures

### Scope
- Global Scope
- Function Scope
- Block Scope
- Module Scope
- Lexical Scope
- Dynamic Scope
- Scope Chain
- Scope Lookup
- Variable Resolution

### Closures
- Closure Definition
- Closure Creation
- Lexical Environment Capture
- Closure Use Cases
- Data Privacy
- Function Factories
- Partial Application
- Module Pattern
- Closure Memory Considerations
- Closure in Loops
- IIFE for Closure

## this Keyword
- this Binding Rules
- Default Binding
- Implicit Binding
- Explicit Binding (call, apply, bind)
- new Binding
- Arrow Function this
- Lexical this
- this Precedence
- this in Event Handlers
- this in Callbacks
- this in Methods
- this in Constructors
- this in Classes
- Losing this Context
- Binding this Solutions
- globalThis

## Objects

### Object Basics
- Object Literal Syntax
- Object Constructor
- Object.create()
- Property Access (Dot Notation)
- Property Access (Bracket Notation)
- Computed Property Names
- Shorthand Property Names
- Shorthand Method Names
- Property Descriptors
- Data Properties
- Accessor Properties
- Getters
- Setters

### Object Methods
- Object.keys()
- Object.values()
- Object.entries()
- Object.fromEntries()
- Object.assign()
- Object.create()
- Object.defineProperty()
- Object.defineProperties()
- Object.getOwnPropertyDescriptor()
- Object.getOwnPropertyDescriptors()
- Object.getOwnPropertyNames()
- Object.getOwnPropertySymbols()
- Object.getPrototypeOf()
- Object.setPrototypeOf()
- Object.is()
- Object.freeze()
- Object.isFrozen()
- Object.seal()
- Object.isSealed()
- Object.preventExtensions()
- Object.isExtensible()
- Object.hasOwn()
- Object.groupBy()

### Property Operations
- Adding Properties
- Deleting Properties (delete)
- Checking Properties (in)
- hasOwnProperty()
- Object.hasOwn()
- Property Enumeration
- Enumerable Properties
- Non-Enumerable Properties
- Property Order

### Object Patterns
- Object Destructuring
- Nested Destructuring
- Default Values in Destructuring
- Rest in Destructuring
- Object Spread
- Object Shallow Copy
- Object Deep Copy
- structuredClone()
- Object Merging
- Object Comparison

## Prototypes & Inheritance

### Prototype Chain
- [[Prototype]]
- __proto__ (deprecated)
- Object.getPrototypeOf()
- Object.setPrototypeOf()
- prototype Property
- Constructor Functions
- new Operator Behavior
- Prototype Chain Lookup
- Property Shadowing
- Prototype Pollution

### Inheritance Patterns
- Prototypal Inheritance
- Constructor Inheritance
- Object.create() Inheritance
- Parasitic Inheritance
- Combination Inheritance
- Prototypal Pattern
- Class-Based Inheritance
- Mixins
- Multiple Inheritance Simulation
- Composition over Inheritance

## Arrays

### Array Creation
- Array Literal
- Array Constructor
- Array.of()
- Array.from()
- Array.fromAsync()
- Spread into Array

### Array Properties
- length Property
- Sparse Arrays
- Dense Arrays
- Array Holes

### Array Methods - Mutating
- push()
- pop()
- shift()
- unshift()
- splice()
- sort()
- reverse()
- fill()
- copyWithin()

### Array Methods - Non-Mutating
- concat()
- slice()
- join()
- toString()
- toLocaleString()
- indexOf()
- lastIndexOf()
- includes()
- find()
- findIndex()
- findLast()
- findLastIndex()
- filter()
- map()
- reduce()
- reduceRight()
- every()
- some()
- flat()
- flatMap()
- at()
- with()
- toSorted()
- toReversed()
- toSpliced()

### Array Iteration
- forEach()
- for...of Loop
- for...in Loop (not recommended)
- entries()
- keys()
- values()

### Array Static Methods
- Array.isArray()
- Array.from()
- Array.of()
- Array.fromAsync()

### Typed Arrays
- Int8Array
- Uint8Array
- Uint8ClampedArray
- Int16Array
- Uint16Array
- Int32Array
- Uint32Array
- Float32Array
- Float64Array
- BigInt64Array
- BigUint64Array

## Strings

### String Creation
- String Literals
- String Constructor
- Template Literals
- Tagged Templates
- Raw Strings (String.raw)

### String Properties
- length Property
- Character Access (index)

### String Methods
- charAt()
- charCodeAt()
- codePointAt()
- at()
- concat()
- includes()
- startsWith()
- endsWith()
- indexOf()
- lastIndexOf()
- search()
- match()
- matchAll()
- replace()
- replaceAll()
- slice()
- substring()
- substr() (deprecated)
- split()
- toLowerCase()
- toUpperCase()
- toLocaleLowerCase()
- toLocaleUpperCase()
- trim()
- trimStart()
- trimEnd()
- padStart()
- padEnd()
- repeat()
- normalize()
- localeCompare()
- isWellFormed()
- toWellFormed()

### String Static Methods
- String.fromCharCode()
- String.fromCodePoint()
- String.raw()

### Template Literals
- String Interpolation
- Multi-line Strings
- Expression Embedding
- Nested Templates
- Tagged Templates
- Tag Functions

## Numbers & Math

### Number Properties
- Number.MAX_VALUE
- Number.MIN_VALUE
- Number.MAX_SAFE_INTEGER
- Number.MIN_SAFE_INTEGER
- Number.POSITIVE_INFINITY
- Number.NEGATIVE_INFINITY
- Number.NaN
- Number.EPSILON

### Number Methods
- toFixed()
- toPrecision()
- toExponential()
- toString()
- toLocaleString()
- valueOf()

### Number Static Methods
- Number.isNaN()
- Number.isFinite()
- Number.isInteger()
- Number.isSafeInteger()
- Number.parseFloat()
- Number.parseInt()

### Math Object
- Math.PI
- Math.E
- Math.abs()
- Math.ceil()
- Math.floor()
- Math.round()
- Math.trunc()
- Math.sign()
- Math.max()
- Math.min()
- Math.pow()
- Math.sqrt()
- Math.cbrt()
- Math.hypot()
- Math.random()
- Math.sin()
- Math.cos()
- Math.tan()
- Math.log()
- Math.log10()
- Math.log2()
- Math.exp()
- Math.clz32()
- Math.fround()
- Math.imul()

### BigInt
- BigInt Literals (n suffix)
- BigInt Constructor
- BigInt Operations
- BigInt Comparisons
- BigInt Limitations
- BigInt with TypedArrays

## Dates & Times
- Date Object
- Date Constructor
- Date.now()
- Date.parse()
- Date.UTC()
- Date Get Methods
- getFullYear()
- getMonth()
- getDate()
- getDay()
- getHours()
- getMinutes()
- getSeconds()
- getMilliseconds()
- getTime()
- getTimezoneOffset()
- Date Set Methods
- Date to String Methods
- toISOString()
- toJSON()
- toDateString()
- toTimeString()
- toLocaleString()
- toLocaleDateString()
- toLocaleTimeString()
- Temporal API (Upcoming)

## Regular Expressions

### RegExp Basics
- RegExp Literal
- RegExp Constructor
- Pattern Syntax
- Flags

### RegExp Flags
- g (global)
- i (case-insensitive)
- m (multiline)
- s (dotAll)
- u (unicode)
- v (unicodeSets)
- y (sticky)
- d (hasIndices)

### Pattern Elements
- Literal Characters
- Metacharacters
- Character Classes
- Negated Character Classes
- Shorthand Character Classes (\d, \w, \s)
- Anchors (^ $)
- Word Boundaries (\b)
- Quantifiers (*, +, ?, {n}, {n,}, {n,m})
- Greedy vs Lazy Quantifiers
- Alternation (|)
- Grouping ()
- Capturing Groups
- Non-Capturing Groups (?:)
- Named Capturing Groups (?<name>)
- Backreferences
- Lookahead (?=) (?!)
- Lookbehind (?<=) (?<!)
- Unicode Property Escapes

### RegExp Methods
- test()
- exec()
- toString()

### String Methods with RegExp
- match()
- matchAll()
- search()
- replace()
- replaceAll()
- split()

## Collections

### Map
- Map Constructor
- Map.prototype.set()
- Map.prototype.get()
- Map.prototype.has()
- Map.prototype.delete()
- Map.prototype.clear()
- Map.prototype.size
- Map.prototype.keys()
- Map.prototype.values()
- Map.prototype.entries()
- Map.prototype.forEach()
- Map vs Object
- Map Iteration Order

### Set
- Set Constructor
- Set.prototype.add()
- Set.prototype.has()
- Set.prototype.delete()
- Set.prototype.clear()
- Set.prototype.size
- Set.prototype.keys()
- Set.prototype.values()
- Set.prototype.entries()
- Set.prototype.forEach()
- Set Operations
- Set.prototype.union()
- Set.prototype.intersection()
- Set.prototype.difference()
- Set.prototype.symmetricDifference()
- Set.prototype.isSubsetOf()
- Set.prototype.isSupersetOf()
- Set.prototype.isDisjointFrom()

### WeakMap
- WeakMap Constructor
- WeakMap.prototype.set()
- WeakMap.prototype.get()
- WeakMap.prototype.has()
- WeakMap.prototype.delete()
- WeakMap Use Cases
- WeakMap vs Map

### WeakSet
- WeakSet Constructor
- WeakSet.prototype.add()
- WeakSet.prototype.has()
- WeakSet.prototype.delete()
- WeakSet Use Cases

### WeakRef
- WeakRef Constructor
- WeakRef.prototype.deref()
- Use Cases

### FinalizationRegistry
- FinalizationRegistry Constructor
- register()
- unregister()
- Cleanup Callbacks

## Iterators & Generators

### Iterators
- Iterator Protocol
- Iterable Protocol
- Symbol.iterator
- Iterator Object
- next() Method
- IteratorResult Object
- done Property
- value Property
- Custom Iterables
- Built-in Iterables

### Generators
- Generator Functions
- function* Syntax
- yield Keyword
- yield* Expression
- Generator Object
- next() Method
- return() Method
- throw() Method
- Generator as Iterator
- Two-Way Communication
- Generator Use Cases
- Lazy Evaluation
- Infinite Sequences

### Async Iteration
- Async Iterators
- Symbol.asyncIterator
- Async Generators
- async function* Syntax
- for await...of Loop

## Asynchronous JavaScript

### Event Loop
- Call Stack
- Callback Queue (Task Queue)
- Microtask Queue
- Event Loop Mechanism
- Macrotasks
- Microtasks
- requestAnimationFrame
- queueMicrotask()

### Callbacks
- Callback Functions
- Callback Pattern
- Error-First Callbacks
- Callback Hell
- Pyramid of Doom

### Promises
- Promise States (pending, fulfilled, rejected)
- Promise Constructor
- Executor Function
- resolve()
- reject()
- Promise.prototype.then()
- Promise.prototype.catch()
- Promise.prototype.finally()
- Promise Chaining
- Returning Promises
- Promise.resolve()
- Promise.reject()
- Promise.all()
- Promise.allSettled()
- Promise.race()
- Promise.any()
- Promise.withResolvers()
- Promisification
- Promise Anti-Patterns

### Async/Await
- async Functions
- await Expression
- Async Function Return Value
- Error Handling with try/catch
- await in Loops
- Parallel Execution
- Sequential Execution
- Promise.all with async/await
- Top-Level Await
- Async Iterators

### Timers
- setTimeout()
- setInterval()
- clearTimeout()
- clearInterval()
- setImmediate() (Node.js)
- process.nextTick() (Node.js)
- requestAnimationFrame()
- cancelAnimationFrame()
- requestIdleCallback()
- cancelIdleCallback()

## Modules

### ES Modules
- import Statement
- Named Imports
- Default Imports
- Namespace Imports
- import * as
- Dynamic import()
- export Statement
- Named Exports
- Default Exports
- Re-exports
- export * from
- Module Scope
- Strict Mode in Modules
- Module Caching
- Circular Dependencies
- Import Assertions
- Import Attributes

### CommonJS (Node.js)
- require()
- module.exports
- exports Object
- __dirname
- __filename
- CommonJS vs ES Modules

### Module Patterns
- Revealing Module Pattern
- Singleton Module
- Module Augmentation
- Barrel Exports

## Classes

### Class Basics
- Class Declaration
- Class Expression
- Constructor Method
- Instance Methods
- Static Methods
- Static Properties
- Instance Properties
- Public Fields
- Private Fields (#)
- Private Methods (#)
- Static Private Fields
- Static Private Methods
- Static Initialization Blocks
- Getter Methods
- Setter Methods
- Computed Method Names

### Inheritance
- extends Keyword
- super Keyword
- super() Constructor Call
- super.method() Calls
- Method Overriding
- Inheriting Static Members
- Extending Built-ins
- Mixins with Classes

### Class Features
- new.target
- instanceof Operator
- Class Hoisting
- Class Strict Mode
- Class toString()
- Symbol.hasInstance
- Symbol.species

## Symbols
- Symbol() Function
- Symbol Description
- Symbol Uniqueness
- Symbol.for()
- Symbol.keyFor()
- Global Symbol Registry
- Well-Known Symbols
- Symbol.iterator
- Symbol.asyncIterator
- Symbol.toStringTag
- Symbol.toPrimitive
- Symbol.hasInstance
- Symbol.species
- Symbol.match
- Symbol.replace
- Symbol.search
- Symbol.split
- Symbol.isConcatSpreadable
- Symbol.unscopables
- Symbols as Property Keys
- Symbols and Enumeration

## Proxy & Reflect

### Proxy
- Proxy Constructor
- Target Object
- Handler Object
- Proxy Traps
- get Trap
- set Trap
- has Trap
- deleteProperty Trap
- ownKeys Trap
- getOwnPropertyDescriptor Trap
- defineProperty Trap
- getPrototypeOf Trap
- setPrototypeOf Trap
- isExtensible Trap
- preventExtensions Trap
- apply Trap
- construct Trap
- Revocable Proxies
- Proxy.revocable()

### Reflect
- Reflect.get()
- Reflect.set()
- Reflect.has()
- Reflect.deleteProperty()
- Reflect.ownKeys()
- Reflect.getOwnPropertyDescriptor()
- Reflect.defineProperty()
- Reflect.getPrototypeOf()
- Reflect.setPrototypeOf()
- Reflect.isExtensible()
- Reflect.preventExtensions()
- Reflect.apply()
- Reflect.construct()

## JSON
- JSON.parse()
- JSON.stringify()
- Replacer Function
- Reviver Function
- Space Parameter
- toJSON() Method
- JSON5
- Circular Reference Handling
- BigInt Serialization

## Web APIs (Browser)

### DOM
- Document Object Model
- document Object
- Element Selection
- getElementById()
- getElementsByClassName()
- getElementsByTagName()
- querySelector()
- querySelectorAll()
- Element Creation
- createElement()
- createTextNode()
- createDocumentFragment()
- Element Manipulation
- appendChild()
- removeChild()
- replaceChild()
- insertBefore()
- cloneNode()
- append()
- prepend()
- after()
- before()
- remove()
- replaceWith()
- Element Properties
- innerHTML
- outerHTML
- textContent
- innerText
- Attributes
- getAttribute()
- setAttribute()
- removeAttribute()
- hasAttribute()
- dataset
- classList
- add()
- remove()
- toggle()
- contains()
- replace()

### Events
- Event Model
- Event Types
- Mouse Events
- Keyboard Events
- Form Events
- Focus Events
- Window Events
- Touch Events
- Pointer Events
- Drag Events
- Clipboard Events
- Event Listeners
- addEventListener()
- removeEventListener()
- Event Object
- event.target
- event.currentTarget
- event.type
- event.preventDefault()
- event.stopPropagation()
- event.stopImmediatePropagation()
- Event Bubbling
- Event Capturing
- Event Delegation
- Custom Events
- CustomEvent()
- dispatchEvent()

### Browser APIs
- Window Object
- Location Object
- History API
- pushState()
- replaceState()
- popstate Event
- Navigator Object
- Screen Object
- Console API
- LocalStorage
- SessionStorage
- IndexedDB
- Cookies
- Fetch API
- XMLHttpRequest
- WebSocket
- Server-Sent Events
- Web Workers
- Service Workers
- Broadcast Channel
- Intersection Observer
- Mutation Observer
- Resize Observer
- Performance API
- Geolocation API
- Notification API
- Clipboard API
- File API
- Drag and Drop API
- Canvas API
- Web Audio API
- WebGL
- WebGPU
- WebRTC

## Memory Management
- Garbage Collection
- Mark-and-Sweep Algorithm
- Reference Counting
- Memory Leaks
- Common Leak Patterns
- Detached DOM Nodes
- Closure Leaks
- Event Listener Leaks
- Timer Leaks
- Memory Profiling
- Heap Snapshots
- WeakMap/WeakSet for Memory
- WeakRef

## Error Handling Patterns
- Try-Catch-Finally
- Error Boundaries
- Global Error Handlers
- window.onerror
- window.onunhandledrejection
- Error Logging
- Error Monitoring
- Graceful Degradation
- Fail-Fast Pattern
- Circuit Breaker Pattern
- Retry Pattern

## Design Patterns

### Creational Patterns
- Constructor Pattern
- Factory Pattern
- Abstract Factory
- Singleton Pattern
- Builder Pattern
- Prototype Pattern
- Module Pattern

### Structural Patterns
- Adapter Pattern
- Decorator Pattern
- Facade Pattern
- Flyweight Pattern
- Proxy Pattern
- Composite Pattern
- Bridge Pattern

### Behavioral Patterns
- Observer Pattern
- Pub/Sub Pattern
- Mediator Pattern
- Command Pattern
- Strategy Pattern
- Iterator Pattern
- State Pattern
- Template Method
- Chain of Responsibility
- Visitor Pattern

### JavaScript-Specific Patterns
- Revealing Module Pattern
- Mixin Pattern
- IIFE Pattern
- Currying Pattern
- Partial Application
- Middleware Pattern
- Chaining Pattern
- Lazy Initialization

## ES2022+ Features

### ES2022
- Class Fields (Public/Private)
- Private Methods
- Static Class Fields
- Static Private Methods
- Static Initialization Blocks
- Top-Level Await
- .at() Method
- Object.hasOwn()
- RegExp Match Indices (d flag)
- Error Cause

### ES2023
- Array findLast()
- Array findLastIndex()
- Array toReversed()
- Array toSorted()
- Array toSpliced()
- Array with()
- Hashbang Grammar
- Symbols as WeakMap Keys

### ES2024
- Promise.withResolvers()
- Object.groupBy()
- Map.groupBy()
- ArrayBuffer.prototype.resize()
- ArrayBuffer.prototype.transfer()
- String.prototype.isWellFormed()
- String.prototype.toWellFormed()
- Atomics.waitAsync()
- RegExp v Flag

### ES2025/2026
- Set Methods (union, intersection, etc.)
- Iterator Helpers
- Decorators
- Record & Tuple (Proposed)
- Pattern Matching (Proposed)
- Pipeline Operator (Proposed)
- Temporal API (Proposed)

## Performance Optimization
- DOM Manipulation Optimization
- Document Fragments
- Virtual DOM Concepts
- Event Delegation
- Debouncing
- Throttling
- Lazy Loading
- Code Splitting
- Tree Shaking
- Minification
- Compression
- Caching Strategies
- Web Workers for Heavy Tasks
- requestAnimationFrame
- requestIdleCallback
- Memory Optimization
- Loop Optimization
- String Concatenation
- Object Property Access

## Testing
- Unit Testing
- Integration Testing
- E2E Testing
- Test Runners
- Jest
- Mocha
- Vitest
- Jasmine
- Assertion Libraries
- Mocking
- Spies
- Stubs
- Test Coverage
- TDD (Test-Driven Development)
- BDD (Behavior-Driven Development)

## Node.js Specifics
- Node.js Runtime
- Global Objects
- process Object
- Buffer
- Streams
- Readable Streams
- Writable Streams
- Duplex Streams
- Transform Streams
- File System (fs)
- Path Module
- HTTP/HTTPS Modules
- Events Module
- EventEmitter
- Child Processes
- Cluster Module
- Worker Threads
- npm/yarn/pnpm
- package.json
- node_modules
- CommonJS vs ESM in Node
- Environment Variables

## Security
- XSS (Cross-Site Scripting)
- CSRF (Cross-Site Request Forgery)
- Injection Attacks
- Prototype Pollution
- eval() Dangers
- Function Constructor Risks
- Content Security Policy
- CORS
- Input Validation
- Output Encoding
- Secure Cookies
- HTTPS
- Subresource Integrity

## Best Practices
- Use const by Default
- Prefer let over var
- Use Strict Mode
- Avoid Global Variables
- Use Descriptive Names
- Keep Functions Small
- Single Responsibility
- DRY (Don't Repeat Yourself)
- KISS (Keep It Simple)
- Avoid Deep Nesting
- Handle Errors Properly
- Use Promises/Async-Await
- Avoid Callback Hell
- Comment Complex Logic
- Use ESLint
- Use Prettier
- Write Tests
- Performance Profiling
- Code Reviews
- Documentation
