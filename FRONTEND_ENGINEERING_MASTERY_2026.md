# Frontend Engineering Mastery Concepts 2026

## Core Web Fundamentals

### HTML5 & Semantic Web
- Semantic Elements (article, section, nav, aside, header, footer)
- Document Outline Algorithm
- Microdata & Schema.org
- Open Graph Protocol
- Meta Tags & SEO
- Responsive Images (srcset, picture, sizes)
- HTML Accessibility Attributes
- Forms & Validation API
- Web Components (Custom Elements, Shadow DOM, Templates, Slots)
- HTML Parsing & DOM Construction

### CSS Mastery
- Box Model (content-box, border-box)
- Specificity & Cascade
- CSS Custom Properties (Variables)
- Flexbox Layout
- CSS Grid Layout
- Subgrid
- Container Queries
- CSS Nesting
- CSS Layers (@layer)
- CSS Scope (@scope)
- Logical Properties
- CSS Functions (calc, clamp, min, max)
- CSS Animations & Transitions
- CSS Transforms (2D, 3D)
- Media Queries (responsive, preference)
- Aspect Ratio
- CSS Containment
- Content Visibility
- CSS Scroll Snap
- CSS View Transitions API
- Anchor Positioning
- CSS Color Spaces (oklch, lab, lch)
- CSS Has Selector (:has())
- CSS Comparison Functions

### CSS Architecture
- BEM Methodology
- ITCSS
- SMACSS
- Atomic CSS / Utility-First CSS
- CSS Modules
- CSS-in-JS (Styled Components, Emotion)
- Zero-Runtime CSS-in-JS (vanilla-extract, Panda CSS)
- Tailwind CSS
- Design Tokens
- Theming Strategies

---

## JavaScript Deep Dive

### Core JavaScript
- Execution Context & Call Stack
- Scope Chain & Closures
- Hoisting (var, let, const, functions)
- this Binding (implicit, explicit, new, arrow)
- Prototypal Inheritance
- Object.create & Object.assign
- Property Descriptors
- Getters & Setters
- Symbols
- Iterators & Generators
- Proxy & Reflect
- WeakMap & WeakSet
- BigInt
- Regular Expressions (Advanced)

### ES6+ Features
- Destructuring (Objects, Arrays, Nested)
- Spread & Rest Operators
- Template Literals & Tagged Templates
- Arrow Functions
- Classes & Inheritance
- Static Properties & Methods
- Private Fields (#)
- Optional Chaining (?.)
- Nullish Coalescing (??)
- Logical Assignment Operators (&&=, ||=, ??=)
- Array Methods (map, filter, reduce, flat, flatMap, at)
- Object Methods (entries, fromEntries, hasOwn)
- String Methods (replaceAll, at, trimStart, trimEnd)
- Promise.allSettled, Promise.any
- Dynamic Import
- Top-Level Await
- Numeric Separators
- Array & Object Grouping (groupBy)
- Set Methods (union, intersection, difference)
- Temporal API
- Decorators (Stage 3)
- Record & Tuple (Stage 2)

### Asynchronous JavaScript
- Event Loop (Call Stack, Task Queue, Microtask Queue)
- Callbacks & Callback Hell
- Promises (then, catch, finally)
- Promise Chaining
- Promise.all, Promise.race, Promise.allSettled, Promise.any
- async/await
- Error Handling in Async Code
- AbortController & AbortSignal
- Concurrent vs Parallel Execution
- Async Iterators
- Async Generators
- Debouncing & Throttling
- requestAnimationFrame
- requestIdleCallback
- queueMicrotask

### Memory Management
- Garbage Collection (Mark & Sweep)
- Memory Leaks (Common Causes)
- WeakRef & FinalizationRegistry
- Memory Profiling
- Heap Snapshots
- Detached DOM Nodes

---

## TypeScript for Frontend

### Essential TypeScript
- Type Annotations
- Type Inference
- Union & Intersection Types
- Literal Types
- Type Guards & Narrowing
- Discriminated Unions
- Generics
- Utility Types (Partial, Required, Pick, Omit, Record)
- Mapped Types
- Conditional Types
- Template Literal Types
- keyof & typeof Operators
- Index Access Types
- satisfies Operator
- as const Assertions
- Function Overloads
- Declaration Files (.d.ts)

### React + TypeScript
- Component Props Typing
- Generic Components
- Event Handler Types
- Ref Types
- Context Types
- Custom Hook Types
- Higher-Order Component Types
- Render Props Types
- Polymorphic Components
- Strict Mode Considerations

---

## React Ecosystem

### React Fundamentals
- JSX & React Elements
- Components (Function vs Class)
- Props & PropTypes
- State Management
- Component Lifecycle
- React Strict Mode
- React DevTools

### React Hooks
- useState
- useEffect
- useContext
- useReducer
- useCallback
- useMemo
- useRef
- useImperativeHandle
- useLayoutEffect
- useDebugValue
- useId
- useSyncExternalStore
- useTransition
- useDeferredValue
- useOptimistic
- useFormStatus
- useActionState
- use Hook (Promise & Context)

### React Patterns
- Compound Components
- Render Props
- Higher-Order Components (HOC)
- Custom Hooks
- Provider Pattern
- Container/Presentational
- Controlled vs Uncontrolled Components
- State Initializer Pattern
- State Reducer Pattern
- Prop Getters Pattern
- Component Composition
- Slots Pattern
- Headless Components

### React Performance
- React.memo
- useMemo & useCallback Optimization
- Code Splitting (React.lazy, Suspense)
- Virtualization (react-window, react-virtual)
- Profiler API
- React DevTools Profiler
- Avoiding Re-renders
- Key Prop Optimization
- Context Performance Pitfalls
- Bundle Size Optimization

### React 19 Features
- React Compiler (automatic memoization)
- Actions & useActionState
- useOptimistic Hook
- use() Hook
- Document Metadata Support
- Stylesheet Precedence
- Async Script Support
- Resource Preloading
- Server Components
- Server Actions
- Improved Error Handling

### React Server Components (RSC)
- Server vs Client Components
- 'use client' Directive
- 'use server' Directive
- Streaming & Suspense
- Data Fetching Patterns
- Composition Patterns
- Serialization Boundaries
- Server Actions
- Progressive Enhancement

---

## State Management

### Local State
- useState Patterns
- useReducer for Complex State
- State Colocation
- Lifting State Up
- State Machines (XState)

### Global State Solutions
- React Context API
- Redux Toolkit
- Redux Saga
- Redux Thunk
- Zustand
- Jotai
- Recoil
- Valtio
- MobX
- Nanostores
- Legend State

### Server State
- TanStack Query (React Query)
- SWR
- RTK Query
- Apollo Client
- URQL
- Stale-While-Revalidate Pattern
- Optimistic Updates
- Cache Invalidation
- Background Refetching
- Infinite Queries
- Prefetching
- Mutations

### Form State
- React Hook Form
- Formik
- Zod Validation
- Yup Validation
- Form Field Arrays
- Form Validation Strategies
- Server-Side Validation Integration

---

## Routing & Navigation

### React Router
- Route Configuration
- Dynamic Routes
- Nested Routes
- Layout Routes
- Index Routes
- Route Parameters
- Search Parameters
- Programmatic Navigation
- Route Guards
- Data Loading (loaders)
- Actions (mutations)
- Error Boundaries
- Pending UI
- Scroll Restoration

### TanStack Router
- Type-Safe Routing
- File-Based Routing
- Search Parameter Validation
- Route Context
- Preloading
- Authenticated Routes

### Navigation Patterns
- Client-Side Navigation
- Shallow Routing
- Parallel Routes
- Intercepting Routes
- Modal Routes
- Breadcrumbs
- Active Link States

---

## Meta-Frameworks

### Next.js
- App Router
- Pages Router
- File-Based Routing
- Dynamic Routes
- Route Groups
- Parallel Routes
- Intercepting Routes
- Server Components
- Client Components
- Data Fetching (fetch, cache)
- Server Actions
- API Routes (Route Handlers)
- Middleware
- Static Site Generation (SSG)
- Incremental Static Regeneration (ISR)
- Server-Side Rendering (SSR)
- Partial Prerendering
- Image Optimization
- Font Optimization
- Script Optimization
- Metadata API
- Caching Strategies
- Deployment (Vercel, Self-hosted)

### Remix
- Nested Routing
- Loader Functions
- Action Functions
- Form Component
- Error Boundaries
- Catch Boundaries
- Resource Routes
- Streaming
- Progressive Enhancement
- Optimistic UI

### Astro
- Content Collections
- Island Architecture
- Zero JS by Default
- Framework Agnostic
- View Transitions
- Server-Side Rendering

---

## Build Tools & Development

### Package Managers
- npm
- yarn (Classic & Berry)
- pnpm
- Workspaces
- Monorepo Management
- Lock Files
- Dependency Resolution
- Peer Dependencies

### Build Tools
- Vite
- Webpack 5
- Turbopack
- esbuild
- Rollup
- Parcel
- SWC
- Babel

### Vite Deep Dive
- Dev Server (Native ESM)
- Hot Module Replacement (HMR)
- Build Optimization
- Plugin System
- CSS Handling
- Asset Handling
- Environment Variables
- Library Mode
- SSR Support

### Module Systems
- ES Modules (import/export)
- CommonJS
- Module Resolution
- Tree Shaking
- Side Effects
- Code Splitting
- Dynamic Imports
- Module Federation

### Monorepo Tools
- Turborepo
- Nx
- Lerna
- pnpm Workspaces
- Shared Dependencies
- Task Orchestration
- Caching Strategies
- Affected Detection

---

## Testing

### Testing Fundamentals
- Test Pyramid
- Unit Testing
- Integration Testing
- End-to-End Testing
- Test-Driven Development (TDD)
- Behavior-Driven Development (BDD)
- Test Coverage
- Mocking Strategies

### Unit Testing
- Vitest
- Jest
- Testing Library Philosophy
- React Testing Library
- User-Event
- Mock Functions
- Spies
- Snapshot Testing
- Test Fixtures
- Test Factories

### Component Testing
- Rendering Components
- Querying DOM
- User Interactions
- Async Testing
- Testing Hooks
- Testing Context
- Testing Forms
- Accessibility Testing
- Visual Regression Testing

### End-to-End Testing
- Playwright
- Cypress
- Page Object Model
- Test Isolation
- Network Mocking
- Authentication Flows
- Cross-Browser Testing
- Mobile Testing
- CI Integration

### API Mocking
- MSW (Mock Service Worker)
- Mirage JS
- API Mocking Strategies
- Fixture Management

---

## Performance Optimization

### Core Web Vitals
- Largest Contentful Paint (LCP)
- Interaction to Next Paint (INP)
- Cumulative Layout Shift (CLS)
- First Contentful Paint (FCP)
- Time to First Byte (TTFB)
- Total Blocking Time (TBT)

### Loading Performance
- Critical Rendering Path
- Render-Blocking Resources
- Code Splitting
- Route-Based Splitting
- Component-Based Splitting
- Lazy Loading
- Preloading & Prefetching
- Resource Hints (preconnect, dns-prefetch)
- Priority Hints
- Bundle Analysis
- Tree Shaking Optimization

### Runtime Performance
- JavaScript Execution Cost
- Long Tasks
- Main Thread Optimization
- Web Workers
- Offscreen Canvas
- WASM Integration
- Virtual DOM Diffing
- Reconciliation Optimization
- Avoiding Layout Thrashing
- Compositor Layers
- will-change Property
- contain Property

### Image Optimization
- Modern Formats (WebP, AVIF)
- Responsive Images
- Image CDN
- Lazy Loading Images
- Blur Placeholders
- Priority Loading
- Image Compression

### Caching Strategies
- HTTP Caching
- Cache-Control Headers
- ETag & If-None-Match
- Service Worker Caching
- Runtime Caching
- Stale-While-Revalidate
- Cache Busting

### Monitoring & Measurement
- Lighthouse
- WebPageTest
- Chrome DevTools Performance
- Performance API
- PerformanceObserver
- Real User Monitoring (RUM)
- Synthetic Monitoring
- Performance Budgets

---

## Browser Internals

### Rendering Pipeline
- DOM Construction
- CSSOM Construction
- Render Tree
- Layout (Reflow)
- Paint
- Composite
- Layer Promotion
- GPU Acceleration

### JavaScript Engine
- Parsing & AST
- Compilation (JIT)
- V8 Optimization
- Inline Caching
- Hidden Classes
- Deoptimization Bailouts

### Event Handling
- Event Propagation (Capture, Target, Bubble)
- Event Delegation
- Passive Event Listeners
- Event Loop
- Task Queue
- Microtask Queue
- requestAnimationFrame Queue
- Intersection Observer
- Resize Observer
- Mutation Observer
- Performance Observer

### Storage
- Cookies
- localStorage
- sessionStorage
- IndexedDB
- Cache API
- Storage Quotas
- Storage Persistence

---

## Web APIs

### Essential APIs
- Fetch API
- AbortController
- URL & URLSearchParams
- FormData
- Blob & File API
- Clipboard API
- Drag & Drop API
- History API
- Geolocation API
- Notifications API
- Fullscreen API
- Page Visibility API
- Screen Wake Lock API
- Web Share API
- Web Speech API

### Advanced APIs
- Web Workers
- Shared Workers
- Service Workers
- Worklets (CSS Houdini)
- WebSockets
- Server-Sent Events
- WebRTC
- Web Animations API
- Intersection Observer
- Resize Observer
- Mutation Observer
- Broadcast Channel API
- Channel Messaging API
- Streams API

### Emerging APIs
- View Transitions API
- Navigation API
- Popover API
- Dialog Element
- Compression Streams API
- File System Access API
- Web Serial API
- Web Bluetooth API
- WebGPU
- WebXR

---

## Progressive Web Apps (PWA)

### PWA Fundamentals
- Web App Manifest
- Service Worker Lifecycle
- Install Prompts
- Offline Support
- Background Sync
- Push Notifications
- App Shortcuts
- Maskable Icons

### Service Worker
- Registration
- Install Event
- Activate Event
- Fetch Event
- Cache Strategies
- Precaching
- Runtime Caching
- Cache Versioning
- Update Flow

### PWA Libraries
- Workbox
- Vite PWA Plugin
- Next PWA

---

## Accessibility (a11y)

### Fundamentals
- WCAG 2.2 Guidelines
- POUR Principles
- Conformance Levels (A, AA, AAA)
- Assistive Technologies
- Screen Readers
- Keyboard Navigation

### Semantic HTML
- Landmark Regions
- Heading Hierarchy
- Lists & Tables
- Form Labels
- Link vs Button
- Image Alt Text

### ARIA
- Roles
- States
- Properties
- Live Regions
- ARIA Patterns
- When Not to Use ARIA

### Accessible Components
- Focus Management
- Focus Trapping
- Skip Links
- Modal Dialogs
- Dropdown Menus
- Tabs & Tab Panels
- Accordions
- Carousels
- Form Validation
- Error Announcements

### Testing Accessibility
- Automated Testing (axe-core)
- Manual Testing
- Screen Reader Testing
- Keyboard Testing
- Color Contrast Testing
- Motion & Animation Preferences

---

## Security

### Common Vulnerabilities
- Cross-Site Scripting (XSS)
- Cross-Site Request Forgery (CSRF)
- Clickjacking
- Open Redirects
- Prototype Pollution
- Supply Chain Attacks
- Dependency Vulnerabilities

### Security Headers
- Content-Security-Policy (CSP)
- X-Frame-Options
- X-Content-Type-Options
- Strict-Transport-Security (HSTS)
- Referrer-Policy
- Permissions-Policy

### Authentication & Authorization
- JWT Handling
- Secure Cookie Attributes
- Token Storage
- OAuth 2.0 / OIDC
- Session Management
- PKCE Flow

### Input Sanitization
- XSS Prevention
- HTML Sanitization
- URL Validation
- File Upload Validation
- Content Type Validation

### Secure Development
- Dependency Auditing
- SAST Tools
- Subresource Integrity
- Trusted Types
- Secrets Management
- Environment Variables

---

## API Integration

### REST APIs
- HTTP Methods
- Status Codes
- Request/Response Headers
- Query Parameters
- Path Parameters
- Request Body Formats
- Error Handling
- Pagination
- Rate Limiting
- Retry Strategies

### GraphQL
- Queries & Mutations
- Fragments
- Variables
- Directives
- Subscriptions
- Schema Design
- Apollo Client
- URQL
- Code Generation
- Caching Strategies
- Optimistic Updates
- Error Handling

### Real-Time Communication
- WebSockets
- Socket.io
- Server-Sent Events
- Polling vs Push
- Reconnection Strategies
- Real-Time State Sync

### API Design Patterns
- BFF (Backend for Frontend)
- API Gateway
- GraphQL Federation
- tRPC
- REST vs GraphQL Selection

---

## Design Systems

### Foundations
- Design Tokens
- Color Systems
- Typography Scale
- Spacing Scale
- Breakpoints
- Motion/Animation Tokens
- Iconography
- Brand Guidelines

### Component Library
- Component API Design
- Prop Naming Conventions
- Composition Patterns
- Variant Patterns
- Polymorphic Components
- Compound Components
- Controlled vs Uncontrolled
- Default Props

### Styling Approaches
- CSS Variables for Theming
- Multi-Theme Support
- Dark Mode Implementation
- High Contrast Mode
- Reduced Motion Support
- RTL Support

### Documentation
- Storybook
- Component Documentation
- Interactive Examples
- Design Guidelines
- Usage Guidelines
- Accessibility Guidelines

### Distribution
- Package Structure
- Versioning (Semver)
- Changelog Management
- Tree-Shakeable Exports
- CSS Delivery
- TypeScript Definitions

---

## Architecture Patterns

### Component Architecture
- Atomic Design
- Feature-Based Structure
- Domain-Driven Design
- Layered Architecture
- Clean Architecture
- Hexagonal Architecture

### Code Organization
- Feature Folders
- Barrel Exports
- Absolute Imports
- Module Boundaries
- Dependency Rules
- Circular Dependency Prevention

### Micro-Frontends
- Module Federation
- Single-SPA
- Web Components Integration
- Iframe Isolation
- Shared Dependencies
- Cross-App Communication
- Deployment Strategies
- Routing Strategies

### Rendering Patterns
- Client-Side Rendering (CSR)
- Server-Side Rendering (SSR)
- Static Site Generation (SSG)
- Incremental Static Regeneration (ISR)
- Partial Hydration
- Progressive Hydration
- Island Architecture
- Streaming SSR
- Selective Hydration

---

## DevOps for Frontend

### CI/CD
- GitHub Actions
- GitLab CI
- Automated Testing
- Build Pipelines
- Preview Deployments
- Branch Deployments
- Rollback Strategies

### Deployment Platforms
- Vercel
- Netlify
- Cloudflare Pages
- AWS Amplify
- AWS S3 + CloudFront
- Docker Containers
- Kubernetes

### Infrastructure
- CDN Configuration
- Edge Functions
- Serverless Functions
- Environment Variables
- Feature Flags
- A/B Testing
- Canary Deployments
- Blue-Green Deployments

### Monitoring
- Error Tracking (Sentry)
- Performance Monitoring
- Log Aggregation
- User Analytics
- Session Replay
- Alerting

---

## Code Quality

### Linting & Formatting
- ESLint
- Prettier
- Stylelint
- TypeScript Strict Mode
- EditorConfig
- Husky & lint-staged
- Commitlint

### Code Review
- Pull Request Best Practices
- Code Review Checklist
- Review Automation
- CODEOWNERS

### Documentation
- JSDoc
- TypeDoc
- README Standards
- Architecture Decision Records (ADR)
- Runbooks

---

## Emerging Technologies 2026

### AI Integration
- AI-Powered Code Completion
- AI UI Components
- LLM API Integration
- Prompt Engineering for UI
- AI-Assisted Testing
- Generative UI

### Edge Computing
- Edge Functions
- Edge Rendering
- Edge Caching
- Geolocation-Based Routing
- Edge Databases

### New Standards
- Import Maps
- CSS Custom Highlight API
- Scroll-Driven Animations
- CSS Anchor Positioning
- Declarative Shadow DOM
- HTML Invokers
- CSS Toggles

### WebAssembly
- WASM Basics
- WASM + JavaScript Interop
- Rust to WASM
- WASM Component Model
- WASI

### Advanced Frameworks
- Solid.js
- Qwik
- Svelte 5 (Runes)
- Vue 3 Composition API
- Angular Signals
- HTMX
- Alpine.js

---

## Senior Engineer Skills

### Technical Leadership
- Code Review Excellence
- Technical Documentation
- Architecture Decisions
- Tech Debt Management
- Performance Culture
- Security Mindset
- Mentoring Developers

### System Thinking
- Cross-Functional Collaboration
- API Contract Design
- Frontend-Backend Communication
- Performance Budgeting
- Scalability Planning
- Disaster Recovery
- Incident Response

### Soft Skills
- Technical Communication
- Stakeholder Management
- Estimation & Planning
- Trade-off Analysis
- Risk Assessment
- Knowledge Sharing
- Team Empowerment

