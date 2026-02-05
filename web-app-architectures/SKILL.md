---
name: web-app-architectures
description: Explains Single Page Applications (SPA), Multi Page Applications (MPA), and hybrid architectures. Use when discussing app architecture decisions, comparing SPA vs MPA, explaining how traditional vs modern web apps work, or when building a new web application.
---

# Web Application Architectures

## Overview

Web applications fall into three main architectural patterns, each with distinct characteristics for navigation, state management, and user experience.

## Multi Page Application (MPA)

**The traditional web model** - each navigation triggers a full page request to the server.

### How It Works

```
User clicks link → Browser requests new HTML → Server renders full page → Browser loads entire page
```

### Characteristics

| Aspect | Behavior |
|--------|----------|
| Navigation | Full page reload on each route change |
| Initial Load | Fast - only current page HTML |
| Subsequent Navigation | Slower - full round trip |
| State | Lost on navigation (unless stored in cookies/sessions) |
| SEO | Excellent - each page is a complete HTML document |
| Server | Handles routing and rendering |

### When to Use MPA

- Content-heavy sites (blogs, documentation, news)
- SEO is critical
- Users have slow connections (less JS to download)
- Simple interactivity requirements
- Progressive enhancement is important

### Example Flow

```
/home → Server renders home.html
/about → Server renders about.html (full page reload)
/products/123 → Server renders product.html (full page reload)
```

## Single Page Application (SPA)

**The modern app model** - one HTML shell, JavaScript handles all routing and rendering.

### How It Works

```
Initial: Browser loads shell HTML + JS bundle
Navigation: JS intercepts clicks → Updates URL → Renders new view (no server request for HTML)
Data: Fetch API calls for JSON data only
```

### Characteristics

| Aspect | Behavior |
|--------|----------|
| Navigation | Instant (no page reload) |
| Initial Load | Slower - must download JS bundle |
| Subsequent Navigation | Fast - only fetch data, render client-side |
| State | Preserved across navigation |
| SEO | Challenging - requires additional strategies |
| Server | API endpoints only (JSON responses) |

### The SPA Tradeoffs

**Advantages:**
- App-like user experience
- Smooth transitions and animations
- Persistent UI state (music players, forms, etc.)
- Reduced server load after initial load

**Disadvantages:**
- Large initial JavaScript bundle
- SEO requires workarounds (SSR, prerendering)
- Memory management complexity
- Back button/deep linking need explicit handling
- Time to Interactive (TTI) can be slow

### Example Flow

```
/home → JS renders Home component
/about → JS renders About component (no server request)
/products/123 → JS fetches product data → Renders Product component
```

## Hybrid Architectures

Modern meta-frameworks blur the line between SPA and MPA.

### Multi Page App with Islands

MPA foundation with interactive "islands" of JavaScript.

```
Server renders full HTML → JS hydrates only interactive components
```

**Examples:** Astro, Fresh (Deno)

### SPA with Server-Side Rendering

SPA that pre-renders on server for initial load.

```
First request: Server renders full HTML + hydrates to SPA
Subsequent: Client-side navigation (SPA behavior)
```

**Examples:** Next.js, Nuxt, SvelteKit, Remix

### Streaming/Progressive Rendering

Server streams HTML as it becomes available.

```
Server starts sending HTML → Browser renders progressively → JS hydrates as content arrives
```

## Architecture Decision Matrix

| Requirement | Recommended |
|-------------|-------------|
| Content site, SEO critical | MPA or Hybrid with SSR |
| Dashboard, authenticated app | SPA or Hybrid |
| E-commerce (SEO + interactivity) | Hybrid with SSR |
| Minimal JS, fast initial load | MPA with Islands |
| Rich interactions, app-like UX | SPA or Hybrid |
| Limited team, simple stack | MPA |
| Offline support needed | SPA with Service Workers |

## Key Concepts to Understand

### Client-Side Routing

SPAs intercept navigation using the History API:

```javascript
// Instead of browser navigation
window.history.pushState({}, '', '/new-route');
// App renders new view without page reload
```

### Code Splitting

Breaking JS bundle into smaller chunks loaded on demand:

```
Initial: core.js (router, framework)
/dashboard: dashboard.chunk.js (loaded when needed)
/settings: settings.chunk.js (loaded when needed)
```

### State Persistence

SPAs maintain state in memory; MPAs must serialize state:

```
SPA: Component state survives navigation
MPA: State stored in URL params, cookies, localStorage, or server sessions
```

---

## Deep Dive: Understanding the Fundamentals

### The Browser's Request-Response Cycle

To truly understand SPA vs MPA, you must understand how browsers work at a fundamental level.

**Traditional Web (MPA) - How the browser was designed:**

```
1. USER ACTION: Click a link <a href="/about">
   
2. BROWSER BEHAVIOR:
   - Stops current page execution
   - Clears current DOM
   - Sends HTTP GET request to server
   - Shows loading indicator
   
3. SERVER RESPONSE:
   - Server receives request for "/about"
   - Server executes backend code (PHP, Ruby, Python, Node)
   - Server queries database if needed
   - Server generates complete HTML document
   - Server sends HTML back with Content-Type: text/html
   
4. BROWSER RENDERING:
   - Browser receives HTML
   - Parses HTML, builds DOM tree
   - Discovers CSS/JS, fetches them
   - Renders page to screen
   - Page is interactive
```

This is how the web worked from 1991 until ~2010. Every navigation = full cycle.

**SPA - Hijacking the browser's default behavior:**

SPAs work by *preventing* the browser's natural behavior:

```javascript
// The core SPA trick: prevent default browser navigation
document.addEventListener('click', (event) => {
  const link = event.target.closest('a');
  
  if (link && link.href.startsWith(window.location.origin)) {
    // STOP the browser from doing its normal thing
    event.preventDefault();
    
    // Instead, WE handle navigation with JavaScript
    const path = new URL(link.href).pathname;
    
    // Update the URL bar (without page reload)
    window.history.pushState({}, '', path);
    
    // Render the new "page" ourselves
    renderRoute(path);
  }
});
```

This is why SPAs feel "app-like" - the page never actually reloads.

### Why Does SPA Navigation Feel Instant?

**MPA Navigation Time Breakdown:**

```
DNS Lookup:           ~20-100ms (if not cached)
TCP Connection:       ~50-200ms (round trip)
TLS Handshake:        ~50-150ms (HTTPS)
Server Processing:    ~50-500ms (database, rendering)
Response Transfer:    ~50-200ms (HTML size dependent)
Browser Parsing:      ~50-100ms
CSS/JS Fetch:         ~100-300ms (even if cached, verification)
Render:               ~50-100ms
─────────────────────────────────
TOTAL:                ~400-1600ms minimum
```

**SPA Navigation Time Breakdown:**

```
JavaScript Execution: ~5-50ms (route matching, component rendering)
DOM Update:           ~5-20ms (virtual DOM diff, real DOM update)
─────────────────────────────────
TOTAL:                ~10-70ms

Data Fetch (if needed): +100-500ms (but can show skeleton immediately)
```

The SPA is **10-100x faster** for navigation because it skips the entire HTTP round-trip.

### The Cost of SPA Speed: Initial Load

But there's a tradeoff. Before ANY navigation can happen, the SPA must:

```
1. Download the HTML shell (small, ~5KB)
2. Download the JavaScript bundle (often 200KB-2MB+)
3. Parse the JavaScript (CPU intensive)
4. Execute the JavaScript (initialize framework, router, stores)
5. Render the initial route

MPA First Page:  ~400-1600ms to content
SPA First Page:  ~800-3000ms to content (must wait for JS)
```

This is why "Time to Interactive" (TTI) is a problem for SPAs.

### Understanding Browser APIs That Enable SPAs

**The History API (HTML5, 2010):**

Before HTML5, the only way to change the URL was to trigger a page load. The History API changed everything:

```javascript
// Push a new entry to browser history (URL changes, no reload)
history.pushState(stateObject, title, '/new-url');

// Replace current entry (URL changes, no reload, no new history entry)
history.replaceState(stateObject, title, '/new-url');

// Listen for back/forward button clicks
window.addEventListener('popstate', (event) => {
  // event.state contains the stateObject from pushState
  // Your app must now render the appropriate content
  renderRoute(window.location.pathname);
});
```

Without the History API, SPAs would only work with hash URLs (`/#/about`).

**The Fetch API:**

SPAs separate data from presentation. Instead of getting HTML, they get JSON:

```javascript
// MPA: Server returns complete HTML page
// <html><body><h1>Product: Shoes</h1><p>Price: $99</p>...</body></html>

// SPA: Server returns just data
// {"name": "Shoes", "price": 99, "description": "..."}

// SPA renders data into components
const response = await fetch('/api/products/123');
const product = await response.json();
renderProduct(product);  // JavaScript creates the HTML
```

### Memory and State: The Fundamental Difference

**MPA State Management:**

```
Page Load → JavaScript runs → State created in memory
Navigation → Page destroyed → ALL MEMORY FREED → New page loads → Fresh state

Each page is a clean slate. No memory leaks possible (page is destroyed).
State that must persist: cookies, localStorage, URL parameters, server sessions.
```

**SPA State Management:**

```
App Load → JavaScript runs → State created in memory
Navigation → State PERSISTS → Components mount/unmount → State grows
... hours later ...
Navigation → State still in memory → Potential memory leaks

The app NEVER gets a clean slate. Memory management is YOUR responsibility.
```

This is why SPAs can have memory leaks and why tools like React DevTools have memory profilers.

### The Document Object Model (DOM) and Why It Matters

The DOM is a tree structure representing your HTML:

```
document
└── html
    ├── head
    │   ├── title
    │   └── link (CSS)
    └── body
        ├── header
        │   └── nav
        ├── main
        │   ├── h1
        │   └── p
        └── footer
```

**MPA DOM Lifecycle:**

```
1. Browser builds DOM from HTML
2. User interacts with page
3. Navigation → ENTIRE DOM DESTROYED
4. New DOM built from new HTML
```

**SPA DOM Lifecycle:**

```
1. Browser builds initial DOM (shell only)
2. JavaScript modifies DOM to add content
3. Navigation → JavaScript MODIFIES DOM (adds/removes nodes)
4. DOM is never destroyed, only mutated
```

SPAs must be careful about DOM manipulation efficiency. This is why frameworks use:
- Virtual DOM (React)
- Compiler-based reactivity (Svelte)
- Fine-grained reactivity (Solid)

### HTTP/2 and Why It Changed the Equation

HTTP/1.1 had a major limitation: one request at a time per connection (or 6 parallel connections max).

```
HTTP/1.1 MPA:
Request 1: HTML ────────────────►
Request 2: CSS  ─────────────────► (waits or new connection)
Request 3: JS   ──────────────────► (waits or new connection)
Request 4: Image ───────────────────► (max 6 parallel)
```

HTTP/2 introduced multiplexing: unlimited parallel requests on one connection.

```
HTTP/2 MPA:
Request 1: HTML  ───►
Request 2: CSS   ───►
Request 3: JS    ───►  All sent simultaneously!
Request 4: Image ───►
```

This made MPAs much faster and reduced the SPA advantage for initial load.

### Server Load: Understanding the Scale Implications

**MPA Server Load:**

```
Each request:
- Parse request
- Route to handler
- Query database
- Execute template engine
- Generate HTML string
- Send response

CPU: High (template rendering per request)
Memory: Moderate (per-request state)
Bandwidth: High (sending full HTML each time)
```

**SPA Server Load:**

```
Initial request:
- Serve static HTML file (cached by CDN)
- Serve static JS bundle (cached by CDN)

API requests:
- Parse request
- Query database
- Return JSON (smaller than HTML)

CPU: Lower (no template rendering)
Memory: Lower (stateless API)
Bandwidth: Lower (JSON smaller than HTML)
```

SPAs can scale more easily because static assets are cached at the edge (CDN).

### SEO: Why Crawlers Struggle with SPAs

Google's crawler has two phases:

```
Phase 1: Crawling (fast, cheap)
- HTTP request to URL
- Receive HTML response
- Extract links for further crawling
- Index the text content

Phase 2: Rendering (slow, expensive)
- Load page in headless Chrome
- Execute JavaScript
- Wait for content to appear
- Index the rendered content
```

**MPA Crawling:**

```
GET /products/shoes → Receives complete HTML with all content
                    → Indexed immediately in Phase 1
```

**SPA Crawling:**

```
GET /products/shoes → Receives: <div id="root"></div>
                    → No content to index in Phase 1
                    → Must wait for Phase 2 (delayed, not guaranteed)
```

Google DOES execute JavaScript, but:
- It's delayed (crawl budget prioritization)
- It's expensive (limited render budget)
- Some pages may never get rendered
- Dynamic content may timeout

This is why SSR exists: serve complete HTML to crawlers, hydrate to SPA for users.

### Progressive Enhancement: The MPA Philosophy

MPAs embrace progressive enhancement:

```
Layer 1: HTML (content, accessible to all)
       ↓
Layer 2: CSS (styling, enhances presentation)
       ↓
Layer 3: JavaScript (interactivity, enhances experience)
```

Each layer is optional. The page works without JS.

SPAs invert this:

```
Layer 1: JavaScript (required for anything to work)
       ↓
Layer 2: Content rendered by JavaScript
       ↓
Layer 3: Everything depends on JS
```

If JS fails (network error, parsing error, old browser), SPA shows nothing.

### When to Choose What: The Real Decision Framework

**Choose MPA when:**
- Content is the product (blogs, news, documentation)
- SEO is non-negotiable
- Users may have JS disabled
- Team is small/unfamiliar with frontend complexity
- Server-side languages are the team's strength

**Choose SPA when:**
- It's an "application" not a "website" (dashboards, tools)
- Users are authenticated (SEO irrelevant)
- Rich interactivity is core to the experience
- Offline support is needed
- Real-time updates are frequent

**Choose Hybrid when:**
- You need both SEO and interactivity (e-commerce)
- Different parts of the site have different needs
- Performance is critical for both initial and subsequent loads
- You want the best of both worlds (most modern apps)

## Related Skills

- See [rendering-patterns](../rendering-patterns/SKILL.md) for SSR, SSG, CSR, ISR
- See [seo-fundamentals](../seo-fundamentals/SKILL.md) for SEO strategies
- See [hydration-patterns](../hydration-patterns/SKILL.md) for hydration concepts
- See [meta-frameworks-overview](../meta-frameworks-overview/SKILL.md) for framework comparisons
