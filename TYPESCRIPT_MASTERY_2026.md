# TypeScript Mastery Concepts 2026

## Fundamentals
- TypeScript Overview
- TypeScript vs JavaScript
- Static Typing
- Type Inference
- Type Annotations
- Structural Typing
- Duck Typing
- Type Soundness
- Strict Mode
- TypeScript Compiler (tsc)
- tsconfig.json
- TypeScript Playground

## Basic Types
- string
- number
- boolean
- null
- undefined
- void
- never
- any
- unknown
- object
- symbol
- bigint
- Array Types
- Tuple Types
- Enum Types
- Numeric Enums
- String Enums
- Heterogeneous Enums
- Const Enums
- Ambient Enums
- Literal Types
- String Literal Types
- Numeric Literal Types
- Boolean Literal Types
- Template Literal Types

## Type Annotations & Inference
- Variable Type Annotations
- Function Parameter Annotations
- Return Type Annotations
- Contextual Typing
- Type Inference Rules
- Best Common Type
- Type Widening
- Type Narrowing
- Literal Inference
- const Assertions
- as const
- satisfies Operator

## Arrays & Tuples
- Array Type Syntax (T[])
- Generic Array Syntax (Array<T>)
- Readonly Arrays
- ReadonlyArray<T>
- Tuple Types
- Named Tuple Elements
- Optional Tuple Elements
- Rest Elements in Tuples
- Readonly Tuples
- Tuple Spread
- Variadic Tuple Types

## Objects & Interfaces
- Object Type Literals
- Interface Declaration
- Optional Properties (?)
- Readonly Properties
- Index Signatures
- String Index Signatures
- Number Index Signatures
- Excess Property Checks
- Interface Extension
- Multiple Interface Extension
- Interface Merging
- Declaration Merging
- Hybrid Types
- Callable Interfaces
- Constructable Interfaces
- Interface vs Type Alias

## Type Aliases
- Type Alias Declaration
- Primitive Type Aliases
- Object Type Aliases
- Union Type Aliases
- Intersection Type Aliases
- Function Type Aliases
- Generic Type Aliases
- Recursive Type Aliases
- Conditional Type Aliases
- Type Alias vs Interface

## Union & Intersection Types
- Union Types (|)
- Discriminated Unions
- Tagged Unions
- Discriminant Property
- Exhaustive Checks
- Intersection Types (&)
- Type Merging
- Combining Object Types
- Union of Intersections
- Intersection of Unions

## Type Narrowing
- typeof Type Guards
- instanceof Type Guards
- Truthiness Narrowing
- Equality Narrowing
- in Operator Narrowing
- Assignment Narrowing
- Control Flow Analysis
- Type Predicates (is)
- Custom Type Guards
- Assertion Functions (asserts)
- Discriminated Union Narrowing
- never Type & Exhaustiveness
- Non-Null Assertion (!)

## Functions
- Function Type Expressions
- Call Signatures
- Construct Signatures
- Function Overloads
- Overload Signatures
- Implementation Signature
- Optional Parameters
- Default Parameters
- Rest Parameters
- Parameter Destructuring
- Return Type Inference
- void Return Type
- never Return Type
- this Parameter
- this Type in Methods
- Function Generic Parameters
- Contextual This Typing

## Generics
- Generic Functions
- Generic Type Parameters
- Generic Interfaces
- Generic Classes
- Generic Type Aliases
- Generic Constraints (extends)
- Multiple Type Parameters
- Default Type Parameters
- Generic Parameter Defaults
- Constraining with keyof
- Constraining with Conditional Types
- Generic Inference
- Generic Spread
- Generic Rest Parameters
- Higher-Order Type Functions

## Classes
- Class Declaration
- Constructor
- Properties
- Public Properties
- Private Properties (#)
- Protected Properties
- Readonly Properties
- Static Properties
- Static Blocks
- Methods
- Getters & Setters
- Index Signatures in Classes
- Class Expressions
- Abstract Classes
- Abstract Methods
- Abstract Properties
- Class Inheritance (extends)
- Interface Implementation (implements)
- Multiple Interface Implementation
- Constructor Parameter Properties
- Method Overriding
- super Keyword
- this Type in Classes
- Class Type Parameters (Generics)
- Static Members in Generics
- Class Decorators
- Member Decorators

## Utility Types

### Built-in Utility Types
- Partial<T>
- Required<T>
- Readonly<T>
- Record<K, T>
- Pick<T, K>
- Omit<T, K>
- Exclude<T, U>
- Extract<T, U>
- NonNullable<T>
- Parameters<T>
- ConstructorParameters<T>
- ReturnType<T>
- InstanceType<T>
- ThisParameterType<T>
- OmitThisParameter<T>
- ThisType<T>
- Awaited<T>
- Uppercase<S>
- Lowercase<S>
- Capitalize<S>
- Uncapitalize<S>
- NoInfer<T>

### Custom Utility Types
- DeepPartial<T>
- DeepReadonly<T>
- DeepRequired<T>
- Mutable<T>
- Nullable<T>
- NonNullableKeys<T>
- RequiredKeys<T>
- OptionalKeys<T>
- FunctionKeys<T>
- UnionToIntersection<T>
- Prettify<T>

## Advanced Types

### Conditional Types
- Basic Conditional Types
- T extends U ? X : Y
- Distributive Conditional Types
- Conditional Type Inference
- infer Keyword
- Inferring in Conditional Types
- Nested Conditional Types
- Recursive Conditional Types

### Mapped Types
- Basic Mapped Types
- Key Remapping (as)
- Property Modifiers (+, -)
- Making Properties Optional
- Making Properties Required
- Making Properties Readonly
- Removing Readonly
- Filtering Keys
- Transforming Value Types
- Template Literal Key Remapping

### Template Literal Types
- Basic Template Literals
- String Interpolation in Types
- Union in Template Literals
- Uppercase/Lowercase Modifiers
- Capitalize/Uncapitalize
- Pattern Matching
- Inferring Parts of Strings
- Recursive Template Literals

### Index Access Types
- Indexed Access (T[K])
- Accessing Array Elements
- Accessing Tuple Elements
- typeof with Index Access
- Nested Index Access
- Union Index Access

### keyof Operator
- keyof Object Types
- keyof with Index Signatures
- keyof with Mapped Types
- keyof Generic Constraints
- keyof Union Types
- keyof Intersection Types

### typeof Operator
- typeof Variables
- typeof Functions
- typeof Classes
- typeof Imported Modules
- ReturnType with typeof

## Type Manipulation
- Type Queries
- Lookup Types
- Type Relationships
- Type Compatibility
- Subtyping
- Supertyping
- Assignability
- Covariance
- Contravariance
- Invariance
- Bivariance
- Strictness Flags & Variance

## Modules
- ES Modules Syntax
- import Statement
- Named Imports
- Default Imports
- Namespace Imports
- import type
- Type-Only Imports
- Type-Only Exports
- export Statement
- Named Exports
- Default Exports
- Re-exports
- export type
- Dynamic Imports
- import()
- Module Resolution
- Node Module Resolution
- Classic Module Resolution
- Bundler Module Resolution
- Path Mapping
- baseUrl
- paths Configuration
- Module Augmentation
- Global Augmentation

## Namespaces
- Namespace Declaration
- Nested Namespaces
- Namespace Merging
- Namespace vs Modules
- Triple-Slash Directives
- /// <reference path="" />
- /// <reference types="" />
- /// <reference lib="" />
- Ambient Namespaces

## Declaration Files
- .d.ts Files
- Ambient Declarations
- declare Keyword
- declare var
- declare function
- declare class
- declare module
- declare namespace
- declare global
- declare enum
- Module Declaration
- Global Declaration
- Augmenting Existing Types
- Augmenting Global Types
- Augmenting Module Types
- DefinitelyTyped (@types)
- Publishing Declaration Files
- types Field in package.json
- typesVersions
- typeRoots Configuration

## Decorators
- Decorator Syntax (@)
- Decorator Factories
- Class Decorators
- Method Decorators
- Accessor Decorators
- Property Decorators
- Parameter Decorators
- Decorator Composition
- Decorator Evaluation Order
- Metadata Reflection
- reflect-metadata
- Stage 3 Decorators (TC39)
- Legacy Decorators
- experimentalDecorators

## Compiler Options (tsconfig.json)

### Type Checking
- strict
- strictNullChecks
- strictFunctionTypes
- strictBindCallApply
- strictPropertyInitialization
- noImplicitAny
- noImplicitThis
- noImplicitReturns
- noImplicitOverride
- noUnusedLocals
- noUnusedParameters
- noFallthroughCasesInSwitch
- noUncheckedIndexedAccess
- noPropertyAccessFromIndexSignature
- exactOptionalPropertyTypes
- useUnknownInCatchVariables
- allowUnreachableCode
- allowUnusedLabels

### Modules
- module
- moduleResolution
- baseUrl
- paths
- rootDirs
- typeRoots
- types
- resolveJsonModule
- resolvePackageJsonExports
- resolvePackageJsonImports
- allowImportingTsExtensions
- allowArbitraryExtensions

### Emit
- target
- lib
- outDir
- outFile
- rootDir
- declaration
- declarationDir
- declarationMap
- emitDeclarationOnly
- sourceMap
- inlineSourceMap
- inlineSources
- removeComments
- noEmit
- noEmitOnError
- importHelpers
- downlevelIteration
- preserveConstEnums
- stripInternal

### JavaScript Support
- allowJs
- checkJs
- maxNodeModuleJsDepth

### Interop
- esModuleInterop
- allowSyntheticDefaultImports
- forceConsistentCasingInFileNames
- isolatedModules
- verbatimModuleSyntax
- erasableSyntaxOnly

### Project References
- composite
- incremental
- tsBuildInfoFile
- references

### Watch Options
- watch
- watchFile
- watchDirectory
- fallbackPolling
- synchronousWatchDirectory
- excludeDirectories
- excludeFiles

## Error Handling
- Error Types
- Error Class Extension
- Typed Error Handling
- Result Type Pattern
- Either Type Pattern
- Optional Chaining (?.)
- Nullish Coalescing (??)
- Non-Null Assertion (!)
- Error Boundaries (React)
- Try-Catch Type Narrowing
- unknown in Catch Clauses

## Asynchronous TypeScript
- Promise<T>
- async Functions
- await Expressions
- Async Return Types
- Promise.all Types
- Promise.race Types
- Promise.allSettled Types
- Promise.any Types
- Async Iterators
- AsyncIterable<T>
- AsyncIterator<T>
- AsyncGenerator<T>
- for await...of
- Top-Level Await

## TypeScript with React

### Component Types
- React.FC / React.FunctionComponent
- React.Component<P, S>
- React.PureComponent<P, S>
- React.ComponentType<P>
- React.ComponentProps<T>
- React.ComponentPropsWithRef<T>
- React.ComponentPropsWithoutRef<T>
- React.PropsWithChildren<P>
- React.PropsWithRef<P>

### Hook Types
- useState<T>
- useReducer<R, A>
- useContext<T>
- useRef<T>
- useMemo<T>
- useCallback<T>
- useEffect Types
- useLayoutEffect Types
- useImperativeHandle<T, R>
- Custom Hook Types

### Event Types
- React.MouseEvent<T>
- React.KeyboardEvent<T>
- React.ChangeEvent<T>
- React.FormEvent<T>
- React.FocusEvent<T>
- React.DragEvent<T>
- React.TouchEvent<T>
- React.WheelEvent<T>
- React.ClipboardEvent<T>
- React.SyntheticEvent<T>

### Element Types
- React.ReactNode
- React.ReactElement
- React.ReactChild
- React.ReactFragment
- React.ReactPortal
- JSX.Element
- JSX.IntrinsicElements

### Ref Types
- React.Ref<T>
- React.RefObject<T>
- React.MutableRefObject<T>
- React.ForwardedRef<T>
- React.forwardRef<T, P>

### Context Types
- React.Context<T>
- React.Provider<T>
- React.Consumer<T>
- createContext<T>

### Other React Types
- React.CSSProperties
- React.HTMLAttributes<T>
- React.SVGAttributes<T>
- React.AriaAttributes
- React.DOMAttributes<T>
- React.InputHTMLAttributes<T>
- React.ButtonHTMLAttributes<T>
- React.FormHTMLAttributes<T>
- React.Key
- React.Suspense
- React.lazy Types
- React.memo Types

## TypeScript with Node.js
- @types/node
- Buffer Types
- Stream Types
- EventEmitter Types
- fs Module Types
- path Module Types
- http/https Types
- process Types
- NodeJS.ProcessEnv
- NodeJS.Timeout
- NodeJS.Immediate
- Express Types
- Request/Response Types
- Middleware Types
- Fastify Types
- NestJS Types

## TypeScript with APIs
- Fetch API Types
- Response Types
- Request Types
- Headers Types
- Axios Types
- AxiosResponse<T>
- AxiosRequestConfig
- API Response Typing
- JSON Type Safety
- Zod Schema Validation
- io-ts Validation
- Type-Safe API Clients
- OpenAPI Types
- GraphQL Types
- tRPC

## Testing TypeScript
- Jest with TypeScript
- ts-jest
- @types/jest
- Vitest
- Mocha with TypeScript
- Testing Library Types
- @testing-library/react
- Mock Types
- jest.Mock<T>
- jest.SpyInstance
- Type-Safe Mocking
- Test Utilities Typing
- Assertion Types
- expect Types
- Custom Matchers

## Build Tools & Bundlers
- tsc (TypeScript Compiler)
- ts-node
- tsx
- esbuild
- swc
- Babel with TypeScript
- @babel/preset-typescript
- webpack with TypeScript
- ts-loader
- Vite with TypeScript
- Rollup with TypeScript
- Parcel with TypeScript
- Turbopack

## Linting & Formatting
- ESLint with TypeScript
- @typescript-eslint/parser
- @typescript-eslint/eslint-plugin
- typescript-eslint Rules
- Type-Aware Linting
- Prettier with TypeScript
- EditorConfig

## TypeScript 5.x Features

### TypeScript 5.0
- const Type Parameters
- extends Constraints on infer
- All Enums Are Union Enums
- Decorators (Stage 3)
- --moduleResolution bundler
- Resolution Customization
- --verbatimModuleSyntax
- export type *
- @satisfies in JSDoc
- @overload in JSDoc
- Bundler Resolution Mode

### TypeScript 5.1
- Easier Implicit Returns for undefined
- Unrelated Types for Getters/Setters
- Decoupled Type-Checking Between JSX Elements
- Linked Cursors for JSX Tags
- @param JSDoc Tag Snippets
- Optimizations

### TypeScript 5.2
- using Declarations (Explicit Resource Management)
- await using
- Disposable Interface
- AsyncDisposable Interface
- Decorator Metadata
- Named/Anonymous Tuple Elements
- Easier Method Usage for Unions of Arrays
- Copying Array Methods

### TypeScript 5.3
- Import Attributes
- import { type } from
- Resolution Mode in Import Types
- switch(true) Narrowing
- Narrowing On Comparisons to Booleans
- instanceof Narrowing Through Symbol.hasInstance
- Checks for super Accesses on Instance Fields
- Interactive Inlay Hints

### TypeScript 5.4
- NoInfer<T> Utility Type
- Preserved Narrowing in Closures
- Object.groupBy & Map.groupBy
- Support require() Calls in --moduleResolution bundler
- Checked Import Attributes
- Quick Fix for Missing Parameters

### TypeScript 5.5
- Inferred Type Predicates
- Control Flow Narrowing for Constant Indexed Accesses
- Type Imports in JSDoc
- Regular Expression Syntax Checking
- Support for New ECMAScript Set Methods
- Isolated Declarations

### TypeScript 5.6+
- Disallowed Nullish and Truthy Checks
- Iterator Helper Methods
- Strict Builtin Iterator Checks
- Support for --module nodenext in Node 22
- Region Folding in Editor
- Enhanced Editor Features

## Advanced Patterns

### Type-Level Programming
- Type Arithmetic
- Recursive Types
- Type-Level State Machines
- Type-Level Parsers
- Variadic Generics
- Higher-Kinded Types Simulation
- Phantom Types
- Branded Types
- Opaque Types
- Nominal Typing Simulation

### Design Patterns
- Factory Pattern Typing
- Singleton Pattern Typing
- Builder Pattern Typing
- Strategy Pattern Typing
- Observer Pattern Typing
- Dependency Injection Typing
- Repository Pattern Typing
- Adapter Pattern Typing

### Functional Programming
- Function Composition Types
- Pipe/Flow Types
- Curry Types
- Partial Application Types
- Monad Types
- Functor Types
- Either/Result Types
- Option/Maybe Types
- Task/IO Types

### Object-Oriented Patterns
- Interface Segregation
- Dependency Inversion
- Liskov Substitution
- Open/Closed Principle
- Single Responsibility
- Mixins
- Composition over Inheritance

## Performance & Optimization
- Type Complexity Management
- Avoiding Deep Nesting
- Type Caching
- Project References
- Incremental Compilation
- Composite Projects
- Build Mode (--build)
- Watch Mode Performance
- skipLibCheck
- Declaration Maps
- Isolating Type Checking

## Best Practices
- Prefer Interfaces for Object Shapes
- Use Type Aliases for Unions/Intersections
- Avoid any (Use unknown)
- Enable Strict Mode
- Use Readonly Where Possible
- Prefer const Assertions
- Use Type Guards for Narrowing
- Leverage Discriminated Unions
- Keep Types Close to Usage
- Export Types with Values
- Document Complex Types
- Use Utility Types
- Avoid Type Assertions
- Prefer satisfies over as
- Use NonNullable
- Handle null/undefined Explicitly
- Avoid Overloads When Possible
- Use Generic Constraints
- Keep Generics Simple
- Test Type Definitions

## Ecosystem & Tools
- DefinitelyTyped
- TypeScript Playground
- typedoc
- ts-morph
- Type Coverage Tools
- typescript-eslint
- ts-prune
- ts-unused-exports
- type-fest
- utility-types
- ts-toolbelt
- type-challenges
- TypeScript Error Translator
- Pretty TypeScript Errors (VS Code)
- Total TypeScript Resources
