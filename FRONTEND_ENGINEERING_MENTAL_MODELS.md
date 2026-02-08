# Frontend Engineering Mental Models 2026

## Mental Model 1: The Frontend Architecture Stack

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MODERN FRONTEND APPLICATION STACK                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  LAYER 7: USER INTERFACE                                            │   │
│  │  Components, Styling, Animations, Interactions                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  LAYER 6: STATE MANAGEMENT                                          │   │
│  │  Local State, Global State, Server State, Form State                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  LAYER 5: ROUTING & NAVIGATION                                      │   │
│  │  Client-Side Routing, Deep Linking, History Management              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  LAYER 4: DATA FETCHING                                             │   │
│  │  REST, GraphQL, WebSockets, Caching, Optimistic Updates             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  LAYER 3: FRAMEWORK / LIBRARY                                       │   │
│  │  React, Vue, Svelte, Solid, Angular                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  LAYER 2: BUILD TOOLING                                             │   │
│  │  Vite, Webpack, TypeScript, Bundling, Transpilation                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  LAYER 1: WEB PLATFORM                                              │   │
│  │  HTML, CSS, JavaScript, Browser APIs, Web Standards                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Framework Selection Decision Tree

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FRAMEWORK / META-FRAMEWORK SELECTION                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  What type of application?                                                 │
│           │                                                                 │
│  ┌────────┴────────────────────────────────────┐                           │
│  │                    │                        │                           │
│  STATIC CONTENT    DYNAMIC APP            HYBRID                           │
│  (Blog, Docs)      (Dashboard, SaaS)      (Marketing + App)                │
│  │                    │                        │                           │
│  ▼                    ▼                        ▼                           │
│  ┌─────────┐      Do you need SEO?         ┌─────────┐                     │
│  │ Astro   │           │                   │ Next.js │                     │
│  │ 11ty    │      YES  │  NO               │ (Best   │                     │
│  │ Hugo    │      ▼    ▼                   │ of both)│                     │
│  └─────────┘  ┌─────────┐ ┌─────────┐      └─────────┘                     │
│               │ Next.js │ │ Vite +  │                                      │
│               │ Remix   │ │ React   │                                      │
│               │ Nuxt    │ │ (SPA)   │                                      │
│               └─────────┘ └─────────┘                                      │
│                                                                             │
│  META-FRAMEWORK COMPARISON:                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  Next.js      │ Full-featured, Vercel ecosystem, most popular      │   │
│  │  Remix        │ Web standards focused, progressive enhancement     │   │
│  │  Astro        │ Content-focused, partial hydration, any framework  │   │
│  │  Nuxt         │ Vue ecosystem, similar to Next.js                  │   │
│  │  SvelteKit    │ Svelte ecosystem, excellent DX                     │   │
│  │  Qwik City    │ Resumability, instant loading                      │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 2: JavaScript Event Loop Visualization

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    JAVASCRIPT EVENT LOOP                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                          CALL STACK                                  │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  Currently executing function                               │    │   │
│  │  ├─────────────────────────────────────────────────────────────┤    │   │
│  │  │  Parent function                                            │    │   │
│  │  ├─────────────────────────────────────────────────────────────┤    │   │
│  │  │  Global execution context                                   │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                          │                                          │   │
│  │                          │ When empty, check queues                 │   │
│  │                          ▼                                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────────────┐    ┌──────────────────────────────────────┐  │
│  │     MICROTASK QUEUE      │    │         TASK QUEUE (Macrotasks)     │  │
│  │     (Highest Priority)   │    │         (Lower Priority)             │  │
│  ├──────────────────────────┤    ├──────────────────────────────────────┤  │
│  │ • Promise.then()         │    │ • setTimeout callbacks               │  │
│  │ • queueMicrotask()       │    │ • setInterval callbacks              │  │
│  │ • MutationObserver       │    │ • I/O callbacks                      │  │
│  │ • process.nextTick (Node)│    │ • UI rendering                       │  │
│  │                          │    │ • requestAnimationFrame              │  │
│  │  ↑ ALL microtasks run    │    │ • Event handlers                     │  │
│  │    before next macrotask │    │                                      │  │
│  └──────────────────────────┘    └──────────────────────────────────────┘  │
│                                                                             │
│  EVENT LOOP CYCLE:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  1. Execute sync code on call stack                                │   │
│  │          │                                                          │   │
│  │          ▼                                                          │   │
│  │  2. Call stack empty?                                              │   │
│  │          │                                                          │   │
│  │          ▼                                                          │   │
│  │  3. Process ALL microtasks ◄──────────────────────────┐            │   │
│  │          │                                            │            │   │
│  │          ▼                                            │            │   │
│  │  4. Render if needed (requestAnimationFrame)          │            │   │
│  │          │                                            │            │   │
│  │          ▼                                            │            │   │
│  │  5. Process ONE macrotask ────────────────────────────┘            │   │
│  │          │                     (Then back to microtasks)           │   │
│  │          ▼                                                          │   │
│  │  6. Repeat                                                         │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  EXAMPLE:                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  console.log('1');                    // Sync - runs first         │   │
│  │  setTimeout(() => console.log('2'), 0); // Macrotask               │   │
│  │  Promise.resolve().then(() => console.log('3')); // Microtask      │   │
│  │  console.log('4');                    // Sync - runs second        │   │
│  │                                                                     │   │
│  │  OUTPUT: 1, 4, 3, 2                                                │   │
│  │          ───  ─  ─                                                 │   │
│  │          sync │  └─ Macrotask (after microtasks)                   │   │
│  │               └─ Microtask (before any macrotask)                  │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 3: React Component Mental Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    REACT COMPONENT LIFECYCLE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  FUNCTION COMPONENT WITH HOOKS:                                            │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  function MyComponent({ prop }) {                                  │   │
│  │    // ═══════════════════════════════════════════════════════════  │   │
│  │    // PHASE 1: RENDER (Pure, no side effects!)                     │   │
│  │    // ═══════════════════════════════════════════════════════════  │   │
│  │                                                                     │   │
│  │    const [state, setState] = useState(initial);  // Initialize     │   │
│  │    const computed = useMemo(() => expensive(state), [state]);      │   │
│  │    const handler = useCallback(() => {}, [dep]);                   │   │
│  │    const ref = useRef(null);                                       │   │
│  │                                                                     │   │
│  │    // ═══════════════════════════════════════════════════════════  │   │
│  │    // PHASE 2: EFFECTS (After render, for side effects)            │   │
│  │    // ═══════════════════════════════════════════════════════════  │   │
│  │                                                                     │   │
│  │    useEffect(() => {                                               │   │
│  │      // Runs AFTER paint (non-blocking)                           │   │
│  │      subscribeToAPI();                                             │   │
│  │      return () => cleanup();  // Cleanup before next effect       │   │
│  │    }, [dependencies]);                                             │   │
│  │                                                                     │   │
│  │    useLayoutEffect(() => {                                         │   │
│  │      // Runs BEFORE paint (blocking) - for DOM measurements       │   │
│  │    }, []);                                                         │   │
│  │                                                                     │   │
│  │    // ═══════════════════════════════════════════════════════════  │   │
│  │    // PHASE 3: RETURN JSX                                          │   │
│  │    // ═══════════════════════════════════════════════════════════  │   │
│  │                                                                     │   │
│  │    return <div>{computed}</div>;                                   │   │
│  │  }                                                                  │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  RENDER CYCLE VISUALIZATION:                                               │
│                                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐  │
│  │ Trigger │───▶│ Render  │───▶│ Commit  │───▶│ Browser │───▶│ Effects │  │
│  │         │    │ Phase   │    │ Phase   │    │ Paint   │    │  Run    │  │
│  └─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘  │
│                                                                             │
│  setState()     Calculate      Update DOM      Screen         useEffect    │
│  props change   new VDOM       (if changed)    updates        callbacks    │
│  parent render  (pure)                                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### React Hooks Mental Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         REACT HOOKS DECISION TREE                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  What do you need?                                                         │
│           │                                                                 │
│  ┌────────┴────────────────────────────────────────────────────┐           │
│  │              │                │                │            │           │
│  STATE        EFFECTS        PERFORMANCE      REFS        CONTEXT          │
│  │              │                │                │            │           │
│  │              │                │                │            │           │
│  ▼              ▼                ▼                ▼            ▼           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │useState  │  │useEffect │  │useMemo   │  │useRef    │  │useContext│     │
│  │Simple    │  │Side      │  │Cache     │  │DOM refs  │  │Read      │     │
│  │state     │  │effects   │  │computed  │  │Mutable   │  │context   │     │
│  │          │  │          │  │values    │  │values    │  │          │     │
│  ├──────────┤  ├──────────┤  ├──────────┤  └──────────┘  └──────────┘     │
│  │useReducer│  │useLayout │  │useCallbck│                                  │
│  │Complex   │  │Effect    │  │Cache     │                                  │
│  │state     │  │Before    │  │functions │                                  │
│  │          │  │paint     │  │          │                                  │
│  └──────────┘  └──────────┘  └──────────┘                                  │
│                                                                             │
│  REACT 19 NEW HOOKS:                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  use()           │ Read promises/context directly (suspense)       │   │
│  │  useActionState  │ Form actions with state (server actions)        │   │
│  │  useFormStatus   │ Pending state in forms                          │   │
│  │  useOptimistic   │ Optimistic UI updates                           │   │
│  │  useTransition   │ Non-blocking state updates                      │   │
│  │  useDeferredVal  │ Defer non-urgent updates                        │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  HOOK RULES:                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ✓ Always call at top level (not in loops/conditions)               │   │
│  │ ✓ Only call in React functions (components, custom hooks)          │   │
│  │ ✓ Name custom hooks with 'use' prefix                              │   │
│  │ ✗ Don't call conditionally: if (x) useState() // WRONG!           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 4: State Management Spectrum

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      STATE MANAGEMENT SPECTRUM                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  LOCAL ◄──────────────────────────────────────────────────────► GLOBAL     │
│                                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │ useState │  │useReducer│  │ Context  │  │ Zustand  │  │  Redux   │     │
│  │          │  │          │  │          │  │ Jotai    │  │  Toolkit │     │
│  │ Component│  │ Component│  │ Subtree  │  │ Global   │  │  Global  │     │
│  │ only     │  │ complex  │  │ sharing  │  │ simple   │  │  complex │     │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘     │
│       │              │              │              │              │        │
│       │              │              │              │              │        │
│   Form input     Shopping      Theme,         App-wide       Large apps   │
│   Toggle         cart logic    Auth           preferences    with many    │
│   Modal open                   Locale                        developers   │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  STATE TYPES & SOLUTIONS:                                                  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  UI STATE              │ useState, useReducer                      │   │
│  │  (Modals, tabs, form)  │ Keep close to usage                       │   │
│  │                        │                                            │   │
│  │  ─────────────────────────────────────────────────────────────────│   │
│  │                        │                                            │   │
│  │  SERVER STATE          │ TanStack Query, SWR, RTK Query            │   │
│  │  (API data)            │ Handles caching, refetching, sync         │   │
│  │                        │                                            │   │
│  │  ─────────────────────────────────────────────────────────────────│   │
│  │                        │                                            │   │
│  │  FORM STATE            │ React Hook Form, Formik                   │   │
│  │  (Input values, errors)│ Specialized for forms                     │   │
│  │                        │                                            │   │
│  │  ─────────────────────────────────────────────────────────────────│   │
│  │                        │                                            │   │
│  │  URL STATE             │ Router (search params)                    │   │
│  │  (Filters, pagination) │ Shareable, bookmarkable                   │   │
│  │                        │                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  GOLDEN RULE: Start local, lift when needed, use server state tools        │
│               for API data (don't put fetched data in Redux!)              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Server State Pattern (TanStack Query)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SERVER STATE WITH TANSTACK QUERY                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  const { data, isLoading, error, refetch } = useQuery({            │   │
│  │    queryKey: ['todos', filters],  // Unique cache key             │   │
│  │    queryFn: () => fetchTodos(filters),                            │   │
│  │    staleTime: 5 * 60 * 1000,     // Fresh for 5 minutes           │   │
│  │    gcTime: 30 * 60 * 1000,       // Keep in cache 30 min          │   │
│  │  });                                                               │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CACHE LIFECYCLE:                                                          │
│                                                                             │
│  ┌─────────┐      ┌─────────┐      ┌─────────┐      ┌─────────┐           │
│  │  FRESH  │─────▶│  STALE  │─────▶│REFETCH  │─────▶│ UPDATED │           │
│  │         │      │         │      │(background)    │         │           │
│  └─────────┘      └─────────┘      └─────────┘      └─────────┘           │
│   Data just       staleTime        On window        New data              │
│   fetched         exceeded         focus, mount     in cache              │
│                                    reconnect                               │
│                                                                             │
│  STALE-WHILE-REVALIDATE PATTERN:                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  1. Return cached (stale) data immediately → Fast UI               │   │
│  │  2. Refetch in background → Fresh data coming                      │   │
│  │  3. Update UI when new data arrives → Seamless update              │   │
│  │                                                                     │   │
│  │  User sees:  [Old Data] ──────────────▶ [New Data]                 │   │
│  │              Instant!                    Smooth transition          │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MUTATION PATTERN:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  const mutation = useMutation({                                    │   │
│  │    mutationFn: createTodo,                                         │   │
│  │    onSuccess: () => {                                              │   │
│  │      queryClient.invalidateQueries({ queryKey: ['todos'] });       │   │
│  │    },                                                              │   │
│  │    // Optimistic update:                                           │   │
│  │    onMutate: async (newTodo) => {                                  │   │
│  │      await queryClient.cancelQueries({ queryKey: ['todos'] });     │   │
│  │      const previous = queryClient.getQueryData(['todos']);         │   │
│  │      queryClient.setQueryData(['todos'], old => [...old, newTodo]);│   │
│  │      return { previous };                                          │   │
│  │    },                                                              │   │
│  │    onError: (err, newTodo, context) => {                           │   │
│  │      queryClient.setQueryData(['todos'], context.previous);        │   │
│  │    },                                                              │   │
│  │  });                                                               │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 5: Rendering Patterns

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      RENDERING PATTERNS COMPARISON                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  CSR (Client-Side Rendering)                                        │   │
│  │  ───────────────────────────                                        │   │
│  │                                                                     │   │
│  │  Server:  HTML (empty shell) ─────────────────────▶ Browser        │   │
│  │           <div id="root"></div>     JS bundle      Render app      │   │
│  │                                                                     │   │
│  │  Timeline:                                                          │   │
│  │  [Download HTML] → [Download JS] → [Execute JS] → [Render] → [Interactive]
│  │  ────────────────────────────────────────────────────────────▶     │   │
│  │                    BLANK SCREEN                    Content!        │   │
│  │                                                                     │   │
│  │  PROS: Simple hosting, full interactivity                          │   │
│  │  CONS: Slow initial load, bad SEO, blank flash                     │   │
│  │  USE: Dashboards, internal tools, authenticated apps               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  SSR (Server-Side Rendering)                                        │   │
│  │  ───────────────────────────                                        │   │
│  │                                                                     │   │
│  │  Server:  Render HTML ──────────────────────────▶ Browser          │   │
│  │           (full page)           + JS bundle        Hydrate          │   │
│  │                                                                     │   │
│  │  Timeline:                                                          │   │
│  │  [Server renders] → [Send HTML] → [Display] → [Download JS] → [Hydrate]
│  │  ─────────────────────────────────────────────────────────────▶    │   │
│  │                              Content!            Interactive!      │   │
│  │                                                                     │   │
│  │  PROS: Good SEO, fast first paint, social previews                 │   │
│  │  CONS: Server cost, TTFB depends on server, hydration cost         │   │
│  │  USE: E-commerce, content sites, public pages                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  SSG (Static Site Generation)                                       │   │
│  │  ────────────────────────────                                       │   │
│  │                                                                     │   │
│  │  Build time: Pre-render all pages ───▶ CDN ──────▶ Browser         │   │
│  │              (HTML files)              Cache       Instant!         │   │
│  │                                                                     │   │
│  │  PROS: Fastest possible, CDN cached, cheap hosting                 │   │
│  │  CONS: Build time scales with pages, stale content                 │   │
│  │  USE: Blogs, docs, marketing sites                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ISR (Incremental Static Regeneration)                              │   │
│  │  ─────────────────────────────────────                              │   │
│  │                                                                     │   │
│  │  [Stale page served] → [Background regeneration] → [New page cached]│   │
│  │                                                                     │   │
│  │  revalidate: 60  // Regenerate every 60 seconds if requested       │   │
│  │                                                                     │   │
│  │  PROS: SSG benefits + fresh content, scales well                   │   │
│  │  CONS: First request after stale gets old content                  │   │
│  │  USE: Product pages, news, frequently updated content              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  RSC (React Server Components) - NEW PARADIGM                       │   │
│  │  ──────────────────────────────────────────                         │   │
│  │                                                                     │   │
│  │  Server Components:     Client Components:                          │   │
│  │  ┌────────────────┐     ┌────────────────┐                         │   │
│  │  │ async function │     │ 'use client'   │                         │   │
│  │  │ Page() {       │     │                │                         │   │
│  │  │   const data = │     │ function       │                         │   │
│  │  │     await db() │     │ Button() {     │                         │   │
│  │  │   return (     │     │   const [x] =  │                         │   │
│  │  │     <Article   │     │     useState() │                         │   │
│  │  │       data />  │     │   ...          │                         │   │
│  │  │   )            │     │ }              │                         │   │
│  │  │ }              │     │                │                         │   │
│  │  └────────────────┘     └────────────────┘                         │   │
│  │  Run on server only     Ship to client                             │   │
│  │  No JS bundle           Includes JS                                │   │
│  │  Direct DB access       Interactivity                              │   │
│  │                                                                     │   │
│  │  BENEFIT: Zero JS for server components, smaller bundles           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Rendering Pattern Decision Tree

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RENDERING PATTERN SELECTION                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Does page need SEO / Social sharing?                                      │
│           │                                                                 │
│       NO  │   YES                                                          │
│       ▼   │                                                                │
│      CSR  │                                                                │
│      (SPA)│                                                                │
│           ▼                                                                 │
│     Is content static / changes infrequently?                              │
│           │                                                                 │
│       YES │   NO                                                           │
│       ▼   │                                                                │
│      SSG  │                                                                │
│           ▼                                                                 │
│     Can stale content be shown briefly?                                    │
│           │                                                                 │
│       YES │   NO                                                           │
│       ▼   │                                                                │
│      ISR  │                                                                │
│           ▼                                                                 │
│      SSR (Always fresh)                                                    │
│                                                                             │
│  MIX AND MATCH: Most real apps use multiple patterns!                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  Homepage:        SSG (fast, rarely changes)                       │   │
│  │  Product pages:   ISR (fresh prices, stale-ok)                     │   │
│  │  User dashboard:  CSR (authenticated, personal)                    │   │
│  │  Checkout:        SSR (needs latest cart)                          │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 6: CSS Layout Systems

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CSS LAYOUT MENTAL MODEL                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  FLEXBOX (1-Dimensional)                                                   │
│  ──────────────────────                                                    │
│  "Items in a row or column"                                                │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  MAIN AXIS ───────────────────────────────────────────────────────▶│   │
│  │  ┌──────┐  ┌──────┐  ┌──────┐     justify-content: space-between   │   │
│  │  │  1   │  │  2   │  │  3   │                                      │   │
│  │  └──────┘  └──────┘  └──────┘                                      │   │
│  │  │                                                                  │   │
│  │  │ CROSS AXIS                      align-items: center             │   │
│  │  ▼                                                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  KEY PROPERTIES:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Container:                  │ Items:                               │   │
│  │ display: flex               │ flex-grow: 1   (take extra space)    │   │
│  │ flex-direction: row/column  │ flex-shrink: 0 (don't shrink)        │   │
│  │ justify-content: center     │ flex-basis: 200px (base size)        │   │
│  │ align-items: stretch        │ align-self: flex-end                 │   │
│  │ gap: 16px                   │                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  GRID (2-Dimensional)                                                      │
│  ────────────────────                                                      │
│  "Items in rows AND columns"                                               │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ┌──────────────────┬──────────┬──────────┐                        │   │
│  │  │                  │          │          │                        │   │
│  │  │     Header       │          │  Sidebar │                        │   │
│  │  │     (span 2)     │          │  (span 3 │                        │   │
│  │  ├──────────────────┤  Main    │   rows)  │                        │   │
│  │  │                  │  Content │          │                        │   │
│  │  │     Article      │          │          │                        │   │
│  │  │                  │          │          │                        │   │
│  │  ├──────────────────┴──────────┤          │                        │   │
│  │  │        Footer               │          │                        │   │
│  │  └─────────────────────────────┴──────────┘                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  KEY PROPERTIES:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  display: grid;                                                    │   │
│  │  grid-template-columns: 1fr 2fr 1fr;    /* 3 columns */           │   │
│  │  grid-template-rows: auto 1fr auto;      /* 3 rows */             │   │
│  │  gap: 16px;                                                        │   │
│  │                                                                     │   │
│  │  /* Item placement */                                              │   │
│  │  grid-column: 1 / 3;    /* Span columns 1-2 */                    │   │
│  │  grid-row: 1 / -1;      /* All rows */                            │   │
│  │  grid-area: header;     /* Named areas */                         │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  WHEN TO USE WHAT:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  FLEXBOX                        GRID                               │   │
│  │  ───────                        ────                               │   │
│  │  Navigation bars                Page layouts                       │   │
│  │  Card layouts (single row)      Card grids                         │   │
│  │  Centering content              Dashboard layouts                  │   │
│  │  Spacing items evenly           Complex 2D layouts                 │   │
│  │  Unknown number of items        Known grid structure               │   │
│  │                                                                     │   │
│  │  TIP: They work great together! Grid for page, Flex for components │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CONTAINER QUERIES (2024+)                                                 │
│  ────────────────────────                                                  │
│  "Component responds to ITS container, not viewport"                       │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  .card-container {                                                  │   │
│  │    container-type: inline-size;                                    │   │
│  │    container-name: card;                                           │   │
│  │  }                                                                  │   │
│  │                                                                     │   │
│  │  @container card (min-width: 400px) {                              │   │
│  │    .card { flex-direction: row; }  /* Horizontal layout */         │   │
│  │  }                                                                  │   │
│  │                                                                     │   │
│  │  Same component adapts whether in sidebar (narrow) or main (wide)  │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 7: Browser Rendering Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    BROWSER RENDERING PIPELINE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  HTML + CSS + JS ──────▶ Pixels on Screen                                  │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐         │   │
│  │  │  Parse  │───▶│  Style  │───▶│ Layout  │───▶│  Paint  │         │   │
│  │  │   HTML  │    │  Calc   │    │(Reflow) │    │         │         │   │
│  │  └─────────┘    └─────────┘    └─────────┘    └─────────┘         │   │
│  │       │              │              │              │               │   │
│  │       ▼              ▼              ▼              ▼               │   │
│  │     DOM           CSSOM         Render        Paint           ┌─────────┐
│  │     Tree          Tree          Tree          Commands        │Composite│
│  │                                                               └─────────┘
│  │                                                                    │   │
│  │                                                                    ▼   │
│  │                                                              GPU Layers │
│  │                                                              (Screen)   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PERFORMANCE IMPACT OF CHANGES:                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  CHANGE TYPE      │ TRIGGERS           │ COST    │ EXAMPLE         │   │
│  │  ─────────────────┼────────────────────┼─────────┼─────────────────│   │
│  │  Geometry change  │ Layout→Paint→Comp  │ HIGH    │ width, height   │   │
│  │  (affects layout) │                    │         │ margin, padding │   │
│  │                   │                    │         │ font-size       │   │
│  │  ─────────────────┼────────────────────┼─────────┼─────────────────│   │
│  │  Paint-only       │ Paint→Composite    │ MEDIUM  │ color, background│   │
│  │  (visual change)  │                    │         │ border-color    │   │
│  │                   │                    │         │ box-shadow      │   │
│  │  ─────────────────┼────────────────────┼─────────┼─────────────────│   │
│  │  Composite-only   │ Composite only     │ LOW     │ transform       │   │
│  │  (GPU accelerated)│ (runs on GPU!)     │ BEST!   │ opacity         │   │
│  │                   │                    │         │ will-change     │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  OPTIMIZATION TIPS:                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  ✓ Animate transform/opacity (GPU accelerated)                     │   │
│  │  ✓ Use will-change sparingly to promote layers                     │   │
│  │  ✓ Batch DOM reads, then batch DOM writes (avoid layout thrashing) │   │
│  │  ✓ Use requestAnimationFrame for visual updates                    │   │
│  │  ✗ Don't animate width/height/top/left (triggers layout)           │   │
│  │  ✗ Don't read layout props in animation loop                       │   │
│  │                                                                     │   │
│  │  LAYOUT THRASHING (BAD):          BATCHED (GOOD):                  │   │
│  │  ──────────────────────           ──────────────                   │   │
│  │  read → write → read → write      read, read, read                 │   │
│  │  (4 layouts!)                     write, write, write              │   │
│  │                                   (2 layouts)                       │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 8: Core Web Vitals

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CORE WEB VITALS                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  THREE METRICS THAT MATTER:                                                │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  LCP (Largest Contentful Paint) - LOADING                          │   │
│  │  ────────────────────────────────────────                          │   │
│  │  "When does the main content appear?"                              │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                                                             │   │   │
│  │  │  ▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │   │   │
│  │  │  0      1.0s    1.5s    2.0s    2.5s    3.0s    3.5s  4.0s │   │   │
│  │  │         │        │        │                                │   │   │
│  │  │       GOOD    NEEDS     POOR                               │   │   │
│  │  │      <2.5s   IMPROVE   >4.0s                               │   │   │
│  │  │                                                             │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                                                                     │   │
│  │  IMPROVE: Optimize images, preload critical resources,             │   │
│  │           use CDN, SSR/SSG for critical content                    │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  INP (Interaction to Next Paint) - INTERACTIVITY                   │   │
│  │  ───────────────────────────────────────────────                   │   │
│  │  "How quickly does the page respond to user input?"                │   │
│  │                                                                     │   │
│  │  User clicks ──▶ [Processing] ──▶ Visual feedback                  │   │
│  │                   └─ INP ─────┘                                    │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                                                             │   │   │
│  │  │  ▓▓▓▓▓▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │   │   │
│  │  │  0    100ms  200ms  300ms  400ms  500ms                    │   │   │
│  │  │         │       │        │                                 │   │   │
│  │  │       GOOD   NEEDS     POOR                                │   │   │
│  │  │      <200ms  IMPROVE   >500ms                              │   │   │
│  │  │                                                             │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                                                                     │   │
│  │  IMPROVE: Break up long tasks, use web workers,                    │   │
│  │           yield to main thread, avoid heavy JS on interaction      │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  CLS (Cumulative Layout Shift) - VISUAL STABILITY                  │   │
│  │  ────────────────────────────────────────────────                  │   │
│  │  "Does content jump around unexpectedly?"                          │   │
│  │                                                                     │   │
│  │  BEFORE (Bad):                  AFTER (Good):                      │   │
│  │  ┌─────────────────┐           ┌─────────────────┐                 │   │
│  │  │ Text here      │           │ [placeholder]    │                 │   │
│  │  │ ↓ IMAGE LOADS! │           │ ↓ Image loads    │                 │   │
│  │  │ Text pushed    │           │ Text stays       │                 │   │
│  │  │ down suddenly  │           │ in place         │                 │   │
│  │  └─────────────────┘           └─────────────────┘                 │   │
│  │                                                                     │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │                                                             │   │   │
│  │  │  ▓▓▓▓▓▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │   │   │
│  │  │  0    0.1    0.15   0.2    0.25   0.3                      │   │   │
│  │  │         │       │       │                                  │   │   │
│  │  │       GOOD   NEEDS    POOR                                 │   │   │
│  │  │      <0.1   IMPROVE  >0.25                                 │   │   │
│  │  │                                                             │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                                                                     │   │
│  │  IMPROVE: Set dimensions on images/videos, reserve space for ads,  │   │
│  │           avoid inserting content above existing content           │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 9: Testing Pyramid

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TESTING PYRAMID                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                              /\                                            │
│                             /  \                                           │
│                            / E2E\     SLOW, EXPENSIVE, FEW                 │
│                           /──────\    Playwright, Cypress                  │
│                          /        \   Real browser, full flow              │
│                         / Integr.  \                                       │
│                        /────────────\ MEDIUM                               │
│                       / Component    \ React Testing Library               │
│                      /  Tests         \ Multiple units together            │
│                     /──────────────────\                                   │
│                    /                    \                                  │
│                   /      Unit Tests      \  FAST, CHEAP, MANY              │
│                  /────────────────────────\ Vitest, Jest                   │
│                 /                          \ Single functions              │
│                /____________________________\                              │
│                                                                             │
│  WHAT TO TEST AT EACH LEVEL:                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  UNIT TESTS (70%)                                                  │   │
│  │  • Pure functions (utils, helpers)                                 │   │
│  │  • Business logic                                                  │   │
│  │  • State reducers                                                  │   │
│  │  • Custom hooks (in isolation)                                     │   │
│  │                                                                     │   │
│  │  INTEGRATION / COMPONENT TESTS (20%)                               │   │
│  │  • Component rendering                                             │   │
│  │  • User interactions                                               │   │
│  │  • Component + hook integration                                    │   │
│  │  • API mocking with MSW                                            │   │
│  │                                                                     │   │
│  │  E2E TESTS (10%)                                                   │   │
│  │  • Critical user flows (signup, checkout)                          │   │
│  │  • Happy paths                                                     │   │
│  │  • Cross-page navigation                                           │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  REACT TESTING LIBRARY PHILOSOPHY:                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  "Test the way users use your app, not implementation details"     │   │
│  │                                                                     │   │
│  │  ✗ BAD: Testing internal state values                              │   │
│  │  ✗ BAD: Testing component instance methods                         │   │
│  │  ✓ GOOD: Testing what user sees                                    │   │
│  │  ✓ GOOD: Testing user interactions                                 │   │
│  │                                                                     │   │
│  │  QUERY PRIORITY:                                                   │   │
│  │  1. getByRole       ← Best! Accessible to everyone                 │   │
│  │  2. getByLabelText  ← Forms                                        │   │
│  │  3. getByPlaceholder← If no label                                  │   │
│  │  4. getByText       ← Non-interactive content                      │   │
│  │  5. getByTestId     ← Last resort                                  │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  EXAMPLE TEST:                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  test('user can submit form', async () => {                        │   │
│  │    render(<ContactForm />);                                        │   │
│  │                                                                     │   │
│  │    // Find by role (accessible!)                                   │   │
│  │    const nameInput = screen.getByRole('textbox', { name: /name/i });│  │
│  │    const submitBtn = screen.getByRole('button', { name: /submit/i });│  │
│  │                                                                     │   │
│  │    // Interact like a user                                         │   │
│  │    await userEvent.type(nameInput, 'John');                        │   │
│  │    await userEvent.click(submitBtn);                               │   │
│  │                                                                     │   │
│  │    // Assert on what user sees                                     │   │
│  │    expect(screen.getByText(/thanks john/i)).toBeInTheDocument();   │   │
│  │  });                                                               │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 10: Accessibility (a11y) Checklist

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ACCESSIBILITY MENTAL MODEL                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  POUR PRINCIPLES (WCAG Foundation):                                        │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  P - PERCEIVABLE     "Can users perceive the content?"             │   │
│  │      • Text alternatives for images (alt text)                     │   │
│  │      • Captions for videos                                         │   │
│  │      • Sufficient color contrast                                   │   │
│  │      • Content works without color alone                           │   │
│  │                                                                     │   │
│  │  O - OPERABLE        "Can users operate the interface?"            │   │
│  │      • Keyboard accessible (Tab, Enter, Escape)                    │   │
│  │      • No keyboard traps                                           │   │
│  │      • Skip links for repetitive content                           │   │
│  │      • Enough time to complete tasks                               │   │
│  │                                                                     │   │
│  │  U - UNDERSTANDABLE  "Can users understand the content?"           │   │
│  │      • Readable text (language declared)                           │   │
│  │      • Predictable navigation                                      │   │
│  │      • Input assistance (labels, error messages)                   │   │
│  │      • Consistent interactions                                     │   │
│  │                                                                     │   │
│  │  R - ROBUST          "Does it work with assistive tech?"           │   │
│  │      • Valid HTML                                                  │   │
│  │      • ARIA used correctly                                         │   │
│  │      • Works across browsers/devices                               │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SEMANTIC HTML (The Foundation):                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  ✗ BAD                          ✓ GOOD                             │   │
│  │  ─────                          ──────                             │   │
│  │  <div onclick>Click me</div>    <button>Click me</button>          │   │
│  │  <div class="header">           <header>                           │   │
│  │  <span class="link">            <a href="...">                     │   │
│  │  <div class="list">             <ul><li>                           │   │
│  │  <b>Important</b>               <strong>Important</strong>         │   │
│  │                                                                     │   │
│  │  LANDMARK REGIONS:                                                 │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │ <header>        → banner                                     │  │   │
│  │  │ <nav>           → navigation                                 │  │   │
│  │  │ <main>          → main (only one!)                          │  │   │
│  │  │ <aside>         → complementary                              │  │   │
│  │  │ <footer>        → contentinfo                                │  │   │
│  │  │ <section>       → region (with aria-label)                  │  │   │
│  │  └──────────────────────────────────────────────────────────────┘  │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ARIA WHEN NEEDED:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  FIRST RULE OF ARIA: Don't use ARIA if HTML can do it!            │   │
│  │                                                                     │   │
│  │  COMMON PATTERNS:                                                  │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │                                                              │  │   │
│  │  │  // Button with icon only                                    │  │   │
│  │  │  <button aria-label="Close dialog">                          │  │   │
│  │  │    <IconX />                                                 │  │   │
│  │  │  </button>                                                   │  │   │
│  │  │                                                              │  │   │
│  │  │  // Loading state                                            │  │   │
│  │  │  <div aria-live="polite" aria-busy="true">                   │  │   │
│  │  │    Loading...                                                │  │   │
│  │  │  </div>                                                      │  │   │
│  │  │                                                              │  │   │
│  │  │  // Expanded/collapsed                                       │  │   │
│  │  │  <button aria-expanded="false" aria-controls="menu">         │  │   │
│  │  │    Menu                                                      │  │   │
│  │  │  </button>                                                   │  │   │
│  │  │                                                              │  │   │
│  │  │  // Tabs                                                     │  │   │
│  │  │  <div role="tablist">                                        │  │   │
│  │  │    <button role="tab" aria-selected="true">Tab 1</button>    │  │   │
│  │  │  </div>                                                      │  │   │
│  │  │                                                              │  │   │
│  │  └──────────────────────────────────────────────────────────────┘  │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  KEYBOARD NAVIGATION CHECKLIST:                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Tab moves through interactive elements in logical order          │   │
│  │ □ Enter/Space activates buttons and links                          │   │
│  │ □ Escape closes modals/dropdowns                                   │   │
│  │ □ Arrow keys navigate within components (tabs, menus)              │   │
│  │ □ Focus is visible (never outline: none without alternative)       │   │
│  │ □ Focus is trapped in modals                                       │   │
│  │ □ Focus returns to trigger after modal closes                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 11: Frontend Security

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FRONTEND SECURITY ESSENTIALS                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  XSS (Cross-Site Scripting) PREVENTION:                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  ATTACK: Attacker injects malicious script                         │   │
│  │  <script>stealCookies()</script> in user input                     │   │
│  │                                                                     │   │
│  │  REACT IS SAFE BY DEFAULT:                                         │   │
│  │  ┌──────────────────────────────────────────────────────────────┐  │   │
│  │  │                                                              │  │   │
│  │  │  // Safe - React escapes this                                │  │   │
│  │  │  <div>{userInput}</div>                                      │  │   │
│  │  │  // Output: &lt;script&gt;...                                │  │   │
│  │  │                                                              │  │   │
│  │  │  // DANGEROUS - bypasses escaping!                           │  │   │
│  │  │  <div dangerouslySetInnerHTML={{ __html: userInput }} />     │  │   │
│  │  │                                                              │  │   │
│  │  │  // If you MUST use dangerouslySetInnerHTML:                 │  │   │
│  │  │  import DOMPurify from 'dompurify';                          │  │   │
│  │  │  <div dangerouslySetInnerHTML={{                             │  │   │
│  │  │    __html: DOMPurify.sanitize(userInput)                     │  │   │
│  │  │  }} />                                                       │  │   │
│  │  │                                                              │  │   │
│  │  └──────────────────────────────────────────────────────────────┘  │   │
│  │                                                                     │   │
│  │  OTHER XSS VECTORS TO WATCH:                                       │   │
│  │  • href="javascript:..." - validate URLs                           │   │
│  │  • eval(), new Function() - never with user input                  │   │
│  │  • document.write() - avoid entirely                               │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  AUTHENTICATION TOKEN STORAGE:                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  OPTION              │ XSS SAFE │ CSRF SAFE │ RECOMMENDATION        │   │
│  │  ────────────────────┼──────────┼───────────┼──────────────────────│   │
│  │  localStorage        │    ✗     │    ✓      │ Avoid for tokens     │   │
│  │  sessionStorage      │    ✗     │    ✓      │ Avoid for tokens     │   │
│  │  HttpOnly Cookie     │    ✓     │    ✗*     │ Best with CSRF token │   │
│  │  Memory (JS variable)│    ✓     │    ✓      │ Lost on refresh      │   │
│  │                                                                     │   │
│  │  * Add CSRF protection when using cookies                          │   │
│  │                                                                     │   │
│  │  BEST PRACTICE:                                                    │   │
│  │  • Access tokens: Short-lived, in memory                          │   │
│  │  • Refresh tokens: HttpOnly, Secure, SameSite=Strict cookie        │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CONTENT SECURITY POLICY (CSP):                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  // HTTP Header                                                    │   │
│  │  Content-Security-Policy:                                          │   │
│  │    default-src 'self';                // Only same origin          │   │
│  │    script-src 'self' 'nonce-abc123';  // Scripts need nonce       │   │
│  │    style-src 'self' 'unsafe-inline';  // Allow inline styles      │   │
│  │    img-src 'self' data: https:;       // Images from HTTPS        │   │
│  │    connect-src 'self' api.example.com; // API calls               │   │
│  │                                                                     │   │
│  │  // In HTML                                                        │   │
│  │  <script nonce="abc123">                                           │   │
│  │    // This script runs because nonce matches                       │   │
│  │  </script>                                                         │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SECURITY CHECKLIST:                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Never trust user input (sanitize, validate)                      │   │
│  │ □ Use HttpOnly cookies for sensitive tokens                        │   │
│  │ □ Implement CSP headers                                            │   │
│  │ □ Enable HTTPS everywhere                                          │   │
│  │ □ Validate URLs before navigation (open redirect prevention)       │   │
│  │ □ Audit dependencies regularly (npm audit)                         │   │
│  │ □ Use Subresource Integrity for CDN scripts                        │   │
│  │ □ Implement proper CORS on your API                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 12: TypeScript for React

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TYPESCRIPT + REACT PATTERNS                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  COMPONENT PROPS PATTERNS:                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  // Basic props                                                    │   │
│  │  interface ButtonProps {                                           │   │
│  │    label: string;                                                  │   │
│  │    onClick: () => void;                                            │   │
│  │    disabled?: boolean;  // Optional                                │   │
│  │  }                                                                 │   │
│  │                                                                     │   │
│  │  // With children                                                  │   │
│  │  interface CardProps {                                             │   │
│  │    children: React.ReactNode;  // Any renderable content          │   │
│  │    title?: string;                                                 │   │
│  │  }                                                                 │   │
│  │                                                                     │   │
│  │  // Extending HTML elements                                        │   │
│  │  interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
│  │    label: string;                                                  │   │
│  │    error?: string;                                                 │   │
│  │  }                                                                 │   │
│  │  // Now InputProps has all <input> props + custom ones             │   │
│  │                                                                     │   │
│  │  // Discriminated union (conditional props)                        │   │
│  │  type AlertProps =                                                 │   │
│  │    | { type: 'success'; message: string }                          │   │
│  │    | { type: 'error'; message: string; retry: () => void };        │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  COMMON HOOK TYPES:                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  // useState with explicit type                                    │   │
│  │  const [user, setUser] = useState<User | null>(null);              │   │
│  │                                                                     │   │
│  │  // useRef with DOM element                                        │   │
│  │  const inputRef = useRef<HTMLInputElement>(null);                  │   │
│  │  // Usage: <input ref={inputRef} />                                │   │
│  │  // Access: inputRef.current?.focus()                              │   │
│  │                                                                     │   │
│  │  // useRef with mutable value                                      │   │
│  │  const timerRef = useRef<number | null>(null);                     │   │
│  │                                                                     │   │
│  │  // Custom hook with return type                                   │   │
│  │  function useToggle(initial = false): [boolean, () => void] {      │   │
│  │    const [value, setValue] = useState(initial);                    │   │
│  │    const toggle = useCallback(() => setValue(v => !v), []);        │   │
│  │    return [value, toggle];                                         │   │
│  │  }                                                                 │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  EVENT HANDLER TYPES:                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  // Click handler                                                  │   │
│  │  const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {};│   │
│  │                                                                     │   │
│  │  // Change handler                                                 │   │
│  │  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {│   │
│  │    console.log(e.target.value);                                    │   │
│  │  };                                                                │   │
│  │                                                                     │   │
│  │  // Form submit                                                    │   │
│  │  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {   │   │
│  │    e.preventDefault();                                             │   │
│  │  };                                                                │   │
│  │                                                                     │   │
│  │  // Keyboard                                                       │   │
│  │  const handleKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
│  │    if (e.key === 'Enter') { ... }                                  │   │
│  │  };                                                                │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  GENERIC COMPONENTS:                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  // Generic list component                                         │   │
│  │  interface ListProps<T> {                                          │   │
│  │    items: T[];                                                     │   │
│  │    renderItem: (item: T) => React.ReactNode;                       │   │
│  │    keyExtractor: (item: T) => string;                              │   │
│  │  }                                                                 │   │
│  │                                                                     │   │
│  │  function List<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
│  │    return (                                                        │   │
│  │      <ul>                                                          │   │
│  │        {items.map(item => (                                        │   │
│  │          <li key={keyExtractor(item)}>{renderItem(item)}</li>      │   │
│  │        ))}                                                         │   │
│  │      </ul>                                                         │   │
│  │    );                                                              │   │
│  │  }                                                                 │   │
│  │                                                                     │   │
│  │  // Usage - TypeScript infers T as User                            │   │
│  │  <List                                                             │   │
│  │    items={users}                                                   │   │
│  │    renderItem={(user) => <span>{user.name}</span>}                 │   │
│  │    keyExtractor={(user) => user.id}                                │   │
│  │  />                                                                │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 13: Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FRONTEND QUICK REFERENCE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  REACT PERFORMANCE CHECKLIST:                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Memoize expensive calculations (useMemo)                         │   │
│  │ □ Memoize callbacks passed to children (useCallback)               │   │
│  │ □ Use React.memo for pure components                               │   │
│  │ □ Virtualize long lists (react-window)                             │   │
│  │ □ Code split routes (lazy loading)                                 │   │
│  │ □ Use proper keys (not index for dynamic lists)                    │   │
│  │ □ Avoid inline object/array literals in JSX                        │   │
│  │ □ Move state down (colocate with usage)                            │   │
│  │ □ Don't put everything in Context                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  COMMON PATTERNS:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                     │   │
│  │  // Debounce (wait for user to stop typing)                        │   │
│  │  useEffect(() => {                                                 │   │
│  │    const timer = setTimeout(() => search(query), 300);             │   │
│  │    return () => clearTimeout(timer);                               │   │
│  │  }, [query]);                                                      │   │
│  │                                                                     │   │
│  │  // Throttle (max once per interval)                               │   │
│  │  const throttledFn = useMemo(                                      │   │
│  │    () => throttle(handleScroll, 100),                              │   │
│  │    []                                                              │   │
│  │  );                                                                │   │
│  │                                                                     │   │
│  │  // Previous value                                                 │   │
│  │  function usePrevious<T>(value: T): T | undefined {                │   │
│  │    const ref = useRef<T>();                                        │   │
│  │    useEffect(() => { ref.current = value; });                      │   │
│  │    return ref.current;                                             │   │
│  │  }                                                                 │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CSS UNITS GUIDE:                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ px    │ Fixed size, use for borders, small details                 │   │
│  │ rem   │ Relative to root font, use for font-size, spacing          │   │
│  │ em    │ Relative to parent font, use sparingly                     │   │
│  │ %     │ Relative to parent, use for widths                         │   │
│  │ vw/vh │ Viewport units, use for full-screen layouts                │   │
│  │ fr    │ Grid fractions, use in grid-template                       │   │
│  │ ch    │ Character width, use for max-width of text                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  KEY TOOLS BY CATEGORY:                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Build:        Vite (dev), Turbopack (Next.js)                      │   │
│  │ State:        Zustand (simple), TanStack Query (server state)      │   │
│  │ Forms:        React Hook Form + Zod                                │   │
│  │ Styling:      Tailwind CSS or CSS Modules                          │   │
│  │ Testing:      Vitest + React Testing Library + Playwright          │   │
│  │ Animation:    Framer Motion                                        │   │
│  │ Date:         date-fns or dayjs                                    │   │
│  │ Icons:        Lucide React or Heroicons                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Mental Model 14: Learning Path

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    FRONTEND ENGINEER LEARNING PATH                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PHASE 1: FOUNDATIONS (Week 1-3)                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Master HTML5 semantic elements & accessibility                   │   │
│  │ □ Deep dive into CSS (Flexbox, Grid, Container Queries)            │   │
│  │ □ JavaScript fundamentals (closures, prototypes, async)            │   │
│  │ □ Understand the event loop                                        │   │
│  │ □ Learn TypeScript basics (types, interfaces, generics)            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 2: REACT MASTERY (Week 4-6)                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Component patterns (composition, compound components)            │   │
│  │ □ All React hooks (including React 19 hooks)                       │   │
│  │ □ State management (useState → Context → Zustand)                  │   │
│  │ □ Server state with TanStack Query                                 │   │
│  │ □ Forms with React Hook Form + Zod                                 │   │
│  │ □ Performance optimization (memo, profiler)                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 3: META-FRAMEWORKS (Week 7-9)                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Next.js App Router                                               │   │
│  │ □ Server Components vs Client Components                           │   │
│  │ □ Data fetching patterns (SSR, SSG, ISR)                           │   │
│  │ □ Server Actions                                                   │   │
│  │ □ Caching strategies                                               │   │
│  │ □ Deployment to Vercel                                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 4: TESTING & QUALITY (Week 10-11)                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Unit testing with Vitest                                         │   │
│  │ □ Component testing with React Testing Library                     │   │
│  │ □ E2E testing with Playwright                                      │   │
│  │ □ API mocking with MSW                                             │   │
│  │ □ Accessibility testing (axe-core)                                 │   │
│  │ □ Set up CI/CD pipeline                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 5: PERFORMANCE & PRODUCTION (Week 12-14)                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ Master Core Web Vitals                                           │   │
│  │ □ Bundle analysis and optimization                                 │   │
│  │ □ Image optimization strategies                                    │   │
│  │ □ Caching strategies (HTTP, Service Worker)                        │   │
│  │ □ Error monitoring (Sentry)                                        │   │
│  │ □ Performance monitoring (RUM)                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PHASE 6: SENIOR SKILLS (Ongoing)                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ □ System design for frontend                                       │   │
│  │ □ Design systems & component libraries                             │   │
│  │ □ Micro-frontends architecture                                     │   │
│  │ □ Monorepo management (Turborepo)                                  │   │
│  │ □ Security best practices                                          │   │
│  │ □ Technical leadership & code review                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PROJECTS TO BUILD:                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. Component Library - Design system with Storybook                │   │
│  │ 2. Dashboard App - Complex state, data viz, real-time              │   │
│  │ 3. E-commerce - Next.js, payments, auth, SEO                       │   │
│  │ 4. Collaborative App - WebSockets, optimistic updates              │   │
│  │ 5. PWA - Offline support, push notifications                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary

This mental models guide covers:

1. **Frontend Stack Architecture** - Layer model, framework selection
2. **JavaScript Event Loop** - Call stack, queues, execution order
3. **React Component Model** - Lifecycle, hooks, render phases
4. **State Management** - Local to global spectrum, server state
5. **Rendering Patterns** - CSR, SSR, SSG, ISR, RSC comparison
6. **CSS Layout** - Flexbox, Grid, Container Queries
7. **Browser Rendering** - Pipeline, performance implications
8. **Core Web Vitals** - LCP, INP, CLS optimization
9. **Testing Pyramid** - Unit, integration, E2E strategies
10. **Accessibility** - POUR principles, ARIA, keyboard nav
11. **Security** - XSS, token storage, CSP
12. **TypeScript + React** - Props, hooks, events typing
13. **Quick Reference** - Checklists, tools, patterns
14. **Learning Path** - 14-week progression to senior
