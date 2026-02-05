---
name: hydration-patterns
description: Explains hydration, partial hydration, progressive hydration, and islands architecture. Use when discussing how server-rendered HTML becomes interactive, optimizing JavaScript loading, or choosing between full hydration and islands architecture.
---

# Hydration Patterns

## What is Hydration?

Hydration is the process of attaching JavaScript event handlers to server-rendered HTML, making static content interactive.

```
Server-Rendered HTML (static) → JavaScript Attaches → Interactive Application
        "Dry HTML"            →     Hydration      →    "Hydrated App"
```

## The Hydration Process

### Step by Step

```
1. Server renders HTML with content
2. Browser displays HTML immediately (fast FCP)
3. Browser downloads JavaScript bundle
4. JavaScript parses and executes
5. Framework "hydrates": 
   - Walks the DOM
   - Attaches event listeners
   - Initializes state
   - Makes components interactive
6. App is now fully interactive (TTI reached)
```

### The Hydration Gap

The time between content visible (FCP) and interactive (TTI):

```
Timeline:
[------- Server Render -------][-- HTML Sent --][---- JS Download ----][-- Hydration --]
                                                 ↓                                      ↓
                                            Content Visible                      Interactive
                                                 [========= Hydration Gap =========]
                                                 (Page looks ready but isn't)
```

## Hydration Challenges

### 1. Time to Interactive (TTI) Delay

Even though content is visible, buttons don't work until hydration completes.

```
User sees "Buy Now" button → Clicks → Nothing happens → Frustrated
                             (hydration not complete)
```

### 2. Hydration Mismatch

If client render differs from server render:

```javascript
// Server: <div>Today is Monday</div>
// Client (different timezone): <div>Today is Tuesday</div>
// Result: Error or UI flicker
```

### 3. Bundle Size

Full hydration requires downloading ALL component code:

```
Page uses: Header, Hero, ProductList, Footer, Cart, Reviews...
Bundle includes: ALL components (even if user never interacts)
```

### 4. CPU Cost

Hydration walks the entire DOM and attaches listeners:

```
Large page = More DOM nodes = Longer hydration = Worse TTI
```

## Full Hydration

**Traditional approach: hydrate the entire page.**

```javascript
// React example
import { hydrateRoot } from 'react-dom/client';
import App from './App';

// Hydrates entire app
hydrateRoot(document.getElementById('root'), <App />);
```

### Pros and Cons

| Pros | Cons |
|------|------|
| Simple mental model | Large JS bundles |
| Full interactivity everywhere | Slow TTI on large pages |
| Framework handles everything | Wasted JS for static content |

## Progressive Hydration

**Hydrate components in priority order, deferring non-critical parts.**

### Strategy

```
1. Hydrate above-the-fold content first
2. Hydrate below-the-fold on scroll or idle
3. Hydrate low-priority on user interaction
```

### Implementation Pattern

```javascript
// Conceptual - defer hydration until visible
function LazyHydrate({ children, whenVisible }) {
  const [hydrated, setHydrated] = useState(false);
  
  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        setHydrated(true);
      }
    });
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);
  
  if (!hydrated) return <div ref={ref}>{/* static HTML */}</div>;
  return children;
}
```

### Timeline

```
[Initial Hydration - Critical Components Only]
        ↓
[Scroll/Idle - Hydrate More Components]
        ↓
[Interaction - Hydrate On Demand]
```

## Selective Hydration

**Only hydrate components that need interactivity.**

React 18+ with Suspense enables this:

```jsx
<Suspense fallback={<Loading />}>
  <Comments /> {/* Hydrates independently when ready */}
</Suspense>
```

### How It Works

```
Server streams: Header (hydrated) → Content (hydrated) → Comments (pending)
Comments data arrives → Comments hydrate independently
User clicks comment before hydration → React prioritizes that subtree
```

## Partial Hydration

**Ship zero JavaScript for static components.**

### The Insight

Most pages have static and interactive parts:

```
┌─────────────────────────────────────┐
│  Header (static)                    │ ← No JS needed
├─────────────────────────────────────┤
│  Hero Banner (static)               │ ← No JS needed
├─────────────────────────────────────┤
│  Product List (static)              │ ← No JS needed
├─────────────────────────────────────┤
│  Add to Cart Button ★               │ ← NEEDS JS
├─────────────────────────────────────┤
│  Reviews (static)                   │ ← No JS needed
├─────────────────────────────────────┤
│  Newsletter Signup ★                │ ← NEEDS JS
├─────────────────────────────────────┤
│  Footer (static)                    │ ← No JS needed
└─────────────────────────────────────┘
```

**With partial hydration:** Only "Add to Cart" and "Newsletter" components ship JS.

## Islands Architecture

**Independent interactive components in a sea of static HTML.**

### Concept

```
┌─────────────────────────────────────────────────┐
│                                                 │
│   Static HTML (server-rendered, no JS)          │
│                                                 │
│   ┌─────────────┐          ┌─────────────┐      │
│   │   Island    │          │   Island    │      │
│   │  (Counter)  │          │  (Search)   │      │
│   │   [JS: 2KB] │          │   [JS: 5KB] │      │
│   └─────────────┘          └─────────────┘      │
│                                                 │
│   Static content continues...                   │
│                                                 │
│   ┌─────────────────────────────────────┐      │
│   │           Island (Comments)          │      │
│   │              [JS: 8KB]               │      │
│   └─────────────────────────────────────┘      │
│                                                 │
└─────────────────────────────────────────────────┘

Total JS: 15KB (vs potentially 200KB+ for full hydration)
```

### Key Characteristics

| Aspect | Full Hydration | Islands |
|--------|----------------|---------|
| JS shipped | All components | Only interactive |
| Hydration | Entire page | Per island |
| Bundle size | Large | Minimal |
| TTI | Slow | Fast |
| Coupling | Components connected | Islands independent |

### Frameworks Using Islands

- **Astro:** First-class islands support
- **Fresh (Deno):** Islands by default
- **Îles:** Islands for Vite
- **Qwik:** Resumability (related concept)

### Island Definition Example (Astro)

```astro
---
// Static by default
import Header from './Header.astro';
import Counter from './Counter.jsx';  // React component
---

<Header />  <!-- No JS shipped -->

<Counter client:load />  <!-- JS shipped, hydrates on load -->

<Counter client:visible />  <!-- JS shipped, hydrates when visible -->

<Counter client:idle />  <!-- JS shipped, hydrates on browser idle -->
```

## Resumability (Qwik's Approach)

**Skip hydration entirely by serializing application state.**

### Traditional Hydration

```
Server renders → Client downloads JS → Re-executes to rebuild state → Attaches listeners
```

### Resumability

```
Server renders + serializes state → Client downloads minimal JS → Resumes exactly where server left off
```

### Key Difference

```javascript
// Hydration: Re-run component code to figure out handlers
// Resumability: Handlers serialized in HTML, just attach them

// Qwik serializes even closures:
<button on:click="./chunk.js#handleClick[0]">Click me</button>
```

## Choosing a Hydration Strategy

| Scenario | Recommended Pattern |
|----------|---------------------|
| SPA/Dashboard | Full hydration |
| Content site with some interactivity | Islands |
| E-commerce product page | Partial/Progressive |
| Marketing landing page | Islands or SSG (no hydration) |
| Highly interactive app | Full hydration with code splitting |
| Performance critical, varied interactivity | Islands or Resumability |

## Hydration Anti-Patterns

### 1. Hydration Mismatch

```javascript
// Bad: Different output on server vs client
function Greeting() {
  return <p>Hello at {new Date().toLocaleTimeString()}</p>;
}

// Good: Defer client-only values
function Greeting() {
  const [time, setTime] = useState(null);
  useEffect(() => setTime(new Date().toLocaleTimeString()), []);
  return <p>Hello{time && ` at ${time}`}</p>;
}
```

### 2. Blocking Hydration on Data

```javascript
// Bad: Hydration waits for fetch
useEffect(() => {
  fetchData().then(setData);  // Delays hydration
}, []);

// Good: Use server-provided data or streaming
```

### 3. Huge Hydration Bundles

```javascript
// Bad: Import everything at top level
import { FullCalendar } from 'massive-calendar-lib';

// Good: Dynamic import for heavy components
const Calendar = lazy(() => import('./Calendar'));
```

---

## Deep Dive: Understanding Hydration From First Principles

### Why Hydration Exists: The Core Problem

Hydration exists because of a fundamental mismatch between two goals:

```
GOAL 1: Fast First Paint (show content quickly)
        → Server rendering gives HTML immediately
        → Browser displays without JavaScript
        
GOAL 2: Rich Interactivity (React/Vue/Svelte apps)
        → Requires JavaScript
        → Event handlers, state, reactivity
        
THE CONFLICT:
        Server-rendered HTML is STATIC
        Your framework needs to MANAGE that HTML
        
THE SOLUTION:
        "Hydration" - attach framework to existing HTML
```

### What Hydration Actually Does Internally

Let's trace through React's hydration process step by step:

```javascript
// 1. Server rendered this HTML:
`<button id="counter">Count: 0</button>`

// 2. Client receives HTML and displays it (fast!)

// 3. JavaScript bundle loads and executes

// 4. React's hydrateRoot runs:
hydrateRoot(document.getElementById('root'), <App />);

// 5. React executes your component to get expected virtual DOM:
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>;
}

// Expected virtual DOM:
// { type: 'button', props: { onClick: fn }, children: ['Count: ', 0] }

// 6. React walks EXISTING DOM and COMPARES to virtual DOM:
const existingButton = document.querySelector('button');
// Does it match? Yes, same structure

// 7. React ATTACHES the onClick handler to existing button:
existingButton.addEventListener('click', () => setCount(c => c + 1));

// 8. React connects state management:
// - useState now tracks count
// - setCount will trigger re-renders
// - Component is now "live"
```

**The key insight:** Hydration doesn't recreate the DOM. It walks the existing DOM and "wires up" the JavaScript.

### The Hydration Mismatch Problem in Detail

When server HTML doesn't match what client would render:

```javascript
// Server renders at 11:59:59 PM December 31:
function NewYearCountdown() {
  const now = new Date();
  return <p>Current time: 11:59:59 PM</p>;
}

// Client hydrates at 12:00:01 AM January 1:
function NewYearCountdown() {
  const now = new Date();  // Different time!
  return <p>Current time: 12:00:01 AM</p>;
}

// React compares:
// Server HTML: "Current time: 11:59:59 PM"
// Client expected: "Current time: 12:00:01 AM"
// MISMATCH!
```

**What happens on mismatch:**

```
DEVELOPMENT MODE:
- Console warning: "Text content did not match"
- React shows the client version (visual jump)

PRODUCTION MODE:
- React silently patches to client version
- Can cause visual flicker
- SEO mismatch if crawled during gap

SEVERE MISMATCH:
- Different DOM structure, not just text
- React may throw errors
- App may break completely
```

**The fix pattern:**

```javascript
function NewYearCountdown() {
  const [time, setTime] = useState(null);  // null on server
  
  useEffect(() => {
    // Only runs on client
    setTime(new Date().toLocaleTimeString());
    const interval = setInterval(() => {
      setTime(new Date().toLocaleTimeString());
    }, 1000);
    return () => clearInterval(interval);
  }, []);
  
  return <p>Current time: {time ?? 'Loading...'}</p>;
}

// Server renders: "Current time: Loading..."
// Client hydrates to same: "Current time: Loading..."
// useEffect updates to real time (no mismatch)
```

### Understanding the Hydration Performance Cost

Hydration is expensive. Let's understand why:

```javascript
// For a page with 1000 DOM nodes:

HYDRATION WORK:
1. Download JS bundle (network time)
2. Parse JavaScript (CPU time)
3. Execute all component code (CPU time)
4. Build virtual DOM tree (memory + CPU)
5. Walk real DOM tree (CPU time)
6. Compare virtual vs real (CPU time)
7. Attach all event listeners (memory)
8. Initialize all state (memory)

// For 1000 nodes, this might take 200-500ms on mobile
// During this time, clicks don't work!
```

**Measuring hydration cost:**

```javascript
// React 18 provides timing
const startMark = performance.now();
hydrateRoot(container, <App />);
const endMark = performance.now();
console.log(`Hydration took ${endMark - startMark}ms`);

// More detailed with React Profiler
<Profiler id="App" onRender={(id, phase, duration) => {
  if (phase === 'mount') {
    // This includes hydration time
    console.log(`${id} hydration: ${duration}ms`);
  }
}}>
  <App />
</Profiler>
```

### Islands Architecture: The Mental Model

Traditional hydration thinks in pages. Islands thinks in components:

```
TRADITIONAL (Page-Level Hydration):
┌──────────────────────────────────────────────┐
│  ┌──────────────────────────────────────┐    │
│  │           Page Component             │    │
│  │  ┌──────┐ ┌──────┐ ┌──────────────┐ │    │
│  │  │Header│ │ Nav  │ │   Content    │ │    │
│  │  └──────┘ └──────┘ └──────────────┘ │    │
│  │  ┌──────┐ ┌──────────────────────┐  │    │
│  │  │Button│ │      Comments        │  │    │
│  │  └──────┘ └──────────────────────┘  │    │
│  └──────────────────────────────────────┘    │
│                                              │
│  ENTIRE TREE must be hydrated                │
│  One root, all components connected          │
└──────────────────────────────────────────────┘


ISLANDS (Component-Level Hydration):
┌──────────────────────────────────────────────┐
│                                              │
│  Static HTML (no JS required)                │
│                                              │
│  ┌──────────────┐      ┌──────────────────┐ │
│  │ Island: Nav  │      │ Island: Search   │ │
│  │ (own JS, own │      │ (own JS, own     │ │
│  │  hydration)  │      │  hydration)      │ │
│  └──────────────┘      └──────────────────┘ │
│                                              │
│  More static HTML...                         │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │      Island: Comments Widget          │   │
│  │      (loads only when visible)        │   │
│  └──────────────────────────────────────┘   │
│                                              │
│  Each island is INDEPENDENT                  │
│  Hydrate separately, fail separately         │
└──────────────────────────────────────────────┘
```

**Islands technical implementation:**

```html
<!-- Astro compiles to something like this -->
<html>
<body>
  <header>Static header content</header>
  
  <!-- Island marker -->
  <astro-island 
    component-url="/components/Counter.js"
    client="visible"
  >
    <button>Count: 0</button>
  </astro-island>
  
  <main>Static main content</main>
  
  <!-- Another island -->
  <astro-island
    component-url="/components/Comments.js" 
    client="idle"
    props='{"postId":123}'
  >
    <div>Loading comments...</div>
  </astro-island>
  
  <footer>Static footer</footer>
</body>
</html>

<!-- The astro-island web component handles loading and hydration -->
```

### Resumability: How Qwik Eliminates Hydration

Qwik takes a radically different approach. Instead of re-running code to figure out what event handlers should do, it serializes everything:

```javascript
// TRADITIONAL HYDRATION:
// Server runs component, produces HTML
// Client runs SAME component code again to attach handlers

// Server:
function Counter() {
  const [count, setCount] = useState(0);
  const increment = () => setCount(c => c + 1);
  return <button onClick={increment}>Count: {count}</button>;
}
// Output: <button>Count: 0</button>
// Client must run Counter() to know what onClick does

// QWIK RESUMABILITY:
// Server runs component, serializes EVERYTHING including handlers

// Server:
function Counter() {
  const count = useSignal(0);
  return <button onClick$={() => count.value++}>Count: {count.value}</button>;
}

// Output:
<button 
  on:click="counter_onclick_abc123.js#s0" 
  q:obj="0"
>Count: 0</button>
<script type="qwik/json">{"signals":{"0":0}}</script>

// Client sees: "When clicked, load counter_onclick_abc123.js and run s0"
// No need to run Counter() at all!
// Handler code downloaded ONLY when button is clicked
```

**Why this matters:**

```
HYDRATION TIMELINE:
[Page Loads] → [Download ALL JS] → [Execute ALL code] → [Interactive]
               [====== Blocking, nothing works ======]

RESUMABILITY TIMELINE:
[Page Loads] → [Interactive immediately]
               [Download handler only on first interaction]
               
// 100KB app with hydration: 500ms until interactive
// 100KB app with resumability: 0ms until interactive (JS loads on demand)
```

### Partial Hydration: The Compiler's Role

Frameworks like Astro use compilers to determine what needs JavaScript:

```javascript
// Source code
---
import Header from './Header.astro';  // Astro component (static)
import Counter from './Counter.jsx'; // React component (interactive)
---

<Header />
<Counter client:load />

// COMPILER ANALYSIS:
// Header.astro: 
//   - No useState, no useEffect, no event handlers
//   - Only template logic
//   - RESULT: Compile to pure HTML, ship NO JavaScript

// Counter.jsx:
//   - Has useState, has onClick
//   - Needs interactivity
//   - RESULT: Ship JavaScript, hydrate this component only

// BUILD OUTPUT:
// index.html: Full HTML with Header content + Counter placeholder
// counter.[hash].js: Only Counter component code (~2KB)
// NOT included: React runtime for static components
```

### The Event Listener Memory Model

Understanding how event listeners work clarifies hydration cost:

```javascript
// Without framework (vanilla JS):
button.addEventListener('click', () => console.log('clicked'));
// Browser stores: {element: button, event: 'click', handler: fn}
// Memory: ~100 bytes per listener

// With React (synthetic events):
<button onClick={() => console.log('clicked')}>
// React stores in its own system:
// - Element reference
// - Event type
// - Handler function
// - Handler closure scope (variables it captures)
// - Fiber node reference
// - Event priority
// Memory: ~500 bytes per listener + closure scope

// A page with 100 interactive elements:
// Vanilla: ~10KB in event system
// React: ~50KB+ in React's event system + component tree
```

This is why hydration is expensive - it's rebuilding all these data structures.

### Streaming Hydration: Out of Order

React 18's streaming SSR allows components to hydrate out of order:

```jsx
<Layout>
  <Header />  {/* Hydrates first */}
  
  <Suspense fallback={<Spinner />}>
    <SlowComments />  {/* Hydrates when ready, even if Footer is waiting */}
  </Suspense>
  
  <Suspense fallback={<Spinner />}>
    <SlowSidebar />  {/* Can hydrate before or after Comments */}
  </Suspense>
  
  <Footer />  {/* Hydrates independently */}
</Layout>
```

**The streaming mechanism:**

```
1. Server streams shell immediately:
   <html><body><div id="header">...</div><div id="comments">Loading...</div>

2. As data arrives, server streams script tags:
   <script>
     $RC('comments', '<div class="comments">Real comments...</div>');
   </script>
   
3. $RC (React's internal function) swaps content and triggers hydration
   for just that subtree

4. User can interact with Header while Comments still loading
```

### Real-World Hydration Optimization Strategies

**1. Code Splitting by Route:**

```javascript
// Don't bundle everything together
const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));

// Only Dashboard code loads on /dashboard
// Settings code only loads when navigating to /settings
```

**2. Deferred Hydration:**

```javascript
// Hydrate critical UI first
import { startTransition } from 'react';

// Critical path - hydrate immediately
hydrateRoot(headerContainer, <Header />);

// Non-critical - defer
startTransition(() => {
  hydrateRoot(commentsContainer, <Comments />);
});
```

**3. Intersection Observer Pattern:**

```javascript
// Only hydrate when scrolled into view
function useHydrateOnVisible(ref) {
  const [shouldHydrate, setShouldHydrate] = useState(false);
  
  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setShouldHydrate(true);
          observer.disconnect();
        }
      },
      { rootMargin: '200px' }  // Start 200px before visible
    );
    
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);
  
  return shouldHydrate;
}
```

**4. Static Extraction:**

```javascript
// Identify components that need no JS
function ProductDescription({ text }) {
  // No state, no effects, no handlers
  // This could be static HTML
  return <p className="description">{text}</p>;
}

// vs
function AddToCartButton({ productId }) {
  const [loading, setLoading] = useState(false);
  const handleClick = () => { /* ... */ };
  // This NEEDS hydration
  return <button onClick={handleClick}>Add to Cart</button>;
}
```

## Related Skills

- See [rendering-patterns](../rendering-patterns/SKILL.md) for SSR/SSG context
- See [web-app-architectures](../web-app-architectures/SKILL.md) for SPA vs MPA
- See [meta-frameworks-overview](../meta-frameworks-overview/SKILL.md) for framework hydration strategies
