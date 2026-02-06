# Golang Mastery Concepts 2026

## Language Fundamentals
- Package Declaration & Imports
- Variables & Constants
- Type Declarations
- Zero Values
- Type Inference (:= operator)
- Blank Identifier (_)
- Visibility (Exported vs Unexported)
- Comments & Documentation

## Data Types
- Basic Types (int, float, bool, string, byte, rune)
- Numeric Types & Precision
- Strings & String Manipulation
- Arrays (Fixed Size)
- Slices (Dynamic Arrays)
- Slice Internals (length, capacity, backing array)
- Maps
- Structs
- Pointers
- Type Aliases vs Type Definitions

## Control Flow
- if/else Statements
- switch Statements (expression & type)
- for Loops (traditional, range, infinite)
- break, continue, goto
- Labels
- defer Statement & Stack
- panic & recover

## Functions
- Function Declaration & Signatures
- Multiple Return Values
- Named Return Values
- Variadic Functions
- First-Class Functions
- Anonymous Functions (Lambdas)
- Closures
- Function Types
- Recursion

## Methods & Interfaces
- Methods on Types
- Value vs Pointer Receivers
- Interfaces & Implicit Implementation
- Empty Interface (any)
- Type Assertions
- Type Switches
- Interface Composition
- Common Interfaces (io.Reader, io.Writer, error, Stringer)

## Generics (Go 1.18+)
- Type Parameters
- Type Constraints
- constraints Package
- Generic Functions
- Generic Types
- Type Inference with Generics
- comparable Constraint
- any Constraint

## Concurrency
- Goroutines
- Channels (buffered & unbuffered)
- Channel Directions (send-only, receive-only)
- select Statement
- Deadlock Detection
- sync.Mutex & sync.RWMutex
- sync.WaitGroup
- sync.Once
- sync.Map
- sync.Pool
- sync.Cond
- atomic Package
- Context Package (cancellation, timeouts, values)
- errgroup Package

## Concurrency Patterns
- Fan-In / Fan-Out
- Worker Pools
- Pipeline Pattern
- Semaphore Pattern
- Rate Limiting
- Graceful Shutdown
- Bounded Parallelism
- Generator Pattern
- Or-Done Channel
- Tee Channel

## Error Handling
- error Interface
- Creating Errors (errors.New, fmt.Errorf)
- Error Wrapping (%w verb)
- errors.Is & errors.As
- errors.Join (Go 1.20+)
- Sentinel Errors
- Custom Error Types
- Panic vs Error
- Handling Multiple Errors

## Memory Management
- Stack vs Heap Allocation
- Escape Analysis
- Garbage Collection
- Memory Profiling
- Reducing Allocations
- Object Pooling
- String Interning

## Packages & Modules
- Go Modules
- go.mod & go.sum
- Semantic Versioning
- Module Proxies
- Private Modules
- Vendoring
- Workspaces (Go 1.18+)
- Internal Packages
- Package Initialization (init functions)

## Testing
- Unit Tests (*_test.go)
- Table-Driven Tests
- Subtests (t.Run)
- Test Fixtures
- t.Parallel()
- Benchmarks
- Fuzz Testing (Go 1.18+)
- Example Tests
- Test Coverage
- Mocking & Interfaces
- testify Library
- httptest Package
- Testing Main (TestMain)

## Standard Library Essentials
- fmt (formatting)
- io & io/fs
- os & os/exec
- bufio
- strings & strconv
- bytes
- encoding/json
- encoding/xml
- net/http
- net/url
- html/template & text/template
- time
- regexp
- sort
- math & math/rand
- crypto
- path/filepath
- log & log/slog (Go 1.21+)
- flag
- embed

## HTTP & Web
- http.Handler & http.HandlerFunc
- ServeMux & Routing
- Enhanced ServeMux (Go 1.22+)
- Middleware Pattern
- Request Context
- HTTP Client & Transport
- Connection Pooling
- Timeouts & Cancellation
- JSON Encoding/Decoding
- Form Handling
- File Uploads
- WebSockets

## Database
- database/sql Package
- Connection Pooling
- Prepared Statements
- Transactions
- Null Types
- sqlx Library
- GORM ORM
- Query Builders
- Migrations

## Performance & Profiling
- pprof (CPU, Memory, Goroutine)
- trace Package
- Benchmarking
- Escape Analysis (-gcflags="-m")
- Compiler Optimizations
- Inlining
- Bounds Check Elimination
- Race Detector (-race)

## Build & Tooling
- go build & go install
- go run
- go fmt & gofmt
- go vet
- go mod tidy
- go generate
- Build Tags
- Cross-Compilation (GOOS, GOARCH)
- CGO
- Build Constraints
- ldflags (version injection)

## Code Organization Patterns
- Flat Structure
- Domain-Driven Design
- Clean Architecture
- Hexagonal Architecture
- Repository Pattern
- Dependency Injection
- Functional Options Pattern
- Builder Pattern
- Factory Pattern
- Singleton Pattern

## Advanced Patterns
- Reflection (reflect package)
- unsafe Package
- Runtime Package
- Plugin System
- Code Generation
- AST Manipulation
- Interface Satisfaction Checks
- Embedding (Composition over Inheritance)
- Method Expressions & Values

## Modern Go (1.21-1.23)
- log/slog (Structured Logging)
- slices Package
- maps Package
- cmp Package
- clear Builtin
- min/max Builtins
- Loop Variable Semantics Change
- Enhanced http.ServeMux Routing
- Range Over Integers
- Range Over Functions (Iterators)

## Popular Frameworks & Libraries
- Gin / Echo / Chi / Fiber (Web)
- gRPC & Protocol Buffers
- Cobra & Viper (CLI)
- Zap / Zerolog (Logging)
- Wire (Dependency Injection)
- sqlc (Type-Safe SQL)
- Ent (Entity Framework)
- go-kit / Kratos (Microservices)
- Watermill (Event-Driven)

## Cloud Native Go
- Docker Integration
- Kubernetes Client-go
- Operator SDK
- Controller Runtime
- Health Checks
- Graceful Shutdown
- Configuration Management
- Secrets Handling
- Observability (OpenTelemetry)
- Feature Flags

## Security
- Input Validation
- SQL Injection Prevention
- XSS Prevention
- CSRF Protection
- Secure Headers
- TLS Configuration
- Secret Management
- Cryptographic Best Practices
- Dependency Scanning (govulncheck)
