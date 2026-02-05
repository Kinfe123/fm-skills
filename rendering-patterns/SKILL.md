---
name: rendering-patterns
description: Explains web rendering strategies including Client-Side Rendering (CSR), Server-Side Rendering (SSR), Static Site Generation (SSG), Incremental Static Regeneration (ISR), and streaming. Use when deciding rendering strategy, optimizing performance, or understanding when content is generated and sent to users.
---

# Web Rendering Patterns

## Overview

Rendering determines **when** and **where** HTML is generated. Each pattern has distinct performance, SEO, and infrastructure implications.

## Rendering Pattern Summary

| Pattern | When Generated | Where Generated | Use Case |
|---------|----------------|-----------------|----------|
| CSR | Runtime (browser) | Client | Dashboards, authenticated apps |
| SSR | Runtime (each request) | Server | Dynamic, personalized content |
| SSG | Build time | Server/Build | Static content, blogs, docs |
| ISR | Build + revalidation | Server | Content that changes periodically |
| Streaming | Runtime (progressive) | Server | Long pages, slow data sources |

## Client-Side Rendering (CSR)

**Browser generates HTML using JavaScript after page load.**

### Flow

```
1. Browser requests page
2. Server returns minimal HTML shell + JS bundle
3. JS executes in browser
4. JS fetches data from API
5. JS renders HTML into DOM
```

### Timeline

```
Request → Empty Shell → JS Downloads → JS Executes → Data Fetches → Content Visible
         [--------------- Time to Interactive (slow) ---------------]
```

### Characteristics

- **First Contentful Paint (FCP):** Slow (waiting for JS)
- **Time to Interactive (TTI):** Slow (JS must execute + fetch data)
- **SEO:** Poor (crawlers see empty shell unless they execute JS)
- **Server load:** Low (static files only)
- **Caching:** Easy (static assets)

### When to Use

- Authenticated dashboards (no SEO needed)
- Highly interactive apps
- Real-time data that can't be pre-rendered
- When server infrastructure is limited

### Code Pattern

```jsx
// Pure CSR - data fetched in browser
function ProductPage() {
  const [product, setProduct] = useState(null);
  
  useEffect(() => {
    fetch('/api/products/123')
      .then(res => res.json())
      .then(setProduct);
  }, []);
  
  if (!product) return <Loading />;
  return <Product data={product} />;
}
```

## Server-Side Rendering (SSR)

**Server generates complete HTML for each request.**

### Flow

```
1. Browser requests page
2. Server fetches data
3. Server renders HTML with data
4. Server sends complete HTML
5. Browser displays content immediately
6. JS hydrates for interactivity
```

### Timeline

```
Request → Server Fetches Data → Server Renders → HTML Sent → Content Visible → Hydration → Interactive
          [--- Server Time ---]                              [--- Hydration ---]
```

### Characteristics

- **FCP:** Fast (complete HTML from server)
- **TTI:** Depends on hydration time
- **SEO:** Excellent (full HTML for crawlers)
- **Server load:** High (render on every request)
- **Caching:** Complex (varies by user/request)

### When to Use

- SEO-critical pages with dynamic content
- Personalized content (user-specific)
- Frequently changing data
- Pages that need fresh data on every request

### Code Pattern (Framework-agnostic concept)

```javascript
// Server-side: runs on each request
async function renderPage(request) {
  const data = await fetchData(request.params.id);
  const html = renderToString(<Page data={data} />);
  return html;
}
```

## Static Site Generation (SSG)

**HTML generated once at build time.**

### Flow

```
Build Time:
1. Build process fetches all data
2. Generates HTML for all pages
3. Outputs static files

Runtime:
1. Browser requests page
2. CDN serves pre-built HTML instantly
```

### Timeline

```
Request → CDN Cache Hit → HTML Delivered → Content Visible (instant)
```

### Characteristics

- **FCP:** Fastest (pre-built, CDN-cached)
- **TTI:** Fast (minimal JS, or none)
- **SEO:** Excellent (complete HTML)
- **Server load:** None at runtime (static files)
- **Caching:** Trivial (immutable until next build)

### When to Use

- Blogs, documentation, marketing pages
- Content that changes infrequently
- Pages where all possible URLs are known at build time
- Maximum performance is required

### Limitations

- Content stale until rebuild
- Build time grows with page count
- Can't handle dynamic routes unknown at build time
- Personalization requires client-side hydration

## Incremental Static Regeneration (ISR)

**SSG with automatic background regeneration.**

### Flow

```
1. Initial build generates static pages
2. Pages served from cache
3. After revalidation time expires:
   - Serve stale page (fast)
   - Regenerate in background
   - Next request gets fresh page
```

### Revalidation Strategies

**Time-based:**
```
Page generated → Serve for 60 seconds → Regenerate on next request after 60s
```

**On-demand:**
```
CMS publishes → Webhook triggers regeneration → Page updated immediately
```

### Characteristics

- **FCP:** Fast (cached HTML)
- **Freshness:** Configurable (seconds to hours)
- **SEO:** Excellent
- **Server load:** Low (regenerate occasionally)
- **Scalability:** High (mostly static)

### When to Use

- E-commerce product pages
- News sites
- Content that updates but not real-time
- High-traffic pages that need freshness

## Streaming / Progressive Rendering

**Server sends HTML in chunks as data becomes available.**

### Flow

```
1. Browser requests page
2. Server immediately sends HTML shell
3. Server fetches data (possibly in parallel)
4. Server streams HTML chunks as ready
5. Browser renders progressively
```

### Timeline

```
Request → Shell Sent → [Chunk 1 Streams] → [Chunk 2 Streams] → Complete
          [Visible]    [More Visible]      [Fully Visible]
```

### Characteristics

- **FCP:** Very fast (shell immediate)
- **TTFB:** Fast (no waiting for all data)
- **User Experience:** Progressive disclosure
- **Complexity:** Higher implementation

### When to Use

- Pages with multiple data sources
- Slow API dependencies
- Long pages where top should render first
- Improving perceived performance

## Rendering Decision Flowchart

```
Does the page need SEO?
├── No → Does it need real-time data?
│        ├── Yes → CSR
│        └── No → SSG or CSR
│
└── Yes → Is content the same for all users?
          ├── Yes → Does content change often?
          │         ├── Rarely → SSG
          │         ├── Sometimes → ISR
          │         └── Every request → SSR
          │
          └── No (personalized) → SSR with caching strategies
```

## Hybrid Approaches

Modern apps mix patterns per route:

| Route | Pattern | Reason |
|-------|---------|--------|
| `/` (home) | SSG | Static marketing content |
| `/blog/*` | SSG/ISR | Content changes occasionally |
| `/products/*` | ISR | Prices update, need SEO |
| `/dashboard` | CSR | Authenticated, no SEO |
| `/search` | SSR | Dynamic query results |

## Performance Metrics Impact

| Pattern | TTFB | FCP | LCP | TTI |
|---------|------|-----|-----|-----|
| CSR | Fast | Slow | Slow | Slow |
| SSR | Slower | Fast | Fast | Medium |
| SSG | Fastest | Fastest | Fastest | Fast |
| ISR | Fastest | Fastest | Fastest | Fast |
| Streaming | Fast | Fast | Progressive | Medium |

---

## Deep Dive: Understanding How Rendering Actually Works

### What Does "Rendering" Actually Mean?

Rendering is the process of converting your application code into HTML that a browser can display. Understanding this deeply requires knowing what happens at each layer.

**The Rendering Pipeline:**

```
Your Code (JSX, Vue template, Svelte)
        ↓
Framework Virtual Representation (Virtual DOM, reactive graph)
        ↓
HTML String or DOM Operations
        ↓
Browser's DOM Tree
        ↓
Browser's Render Tree (DOM + CSS)
        ↓
Layout (position, size calculations)
        ↓
Paint (pixels on screen)
        ↓
Composite (layers combined)
```

When we say "server rendering" vs "client rendering", we're talking about WHERE the first few steps happen.

### The Fundamental Trade-off: Work Location

Every web page requires work to be done. The question is: who does the work?

```
SERVER RENDERING:
┌─────────────────────────────────────────────────────────────────┐
│ SERVER (powerful, shared)                                       │
│                                                                 │
│  [Fetch Data] → [Build HTML String] → [Send HTML]              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ CLIENT (varies, individual)                                     │
│                                                                 │
│  [Parse HTML] → [Build DOM] → [Display]                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘


CLIENT RENDERING:
┌─────────────────────────────────────────────────────────────────┐
│ SERVER (powerful, shared)                                       │
│                                                                 │
│  [Send static JS bundle]                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────┐
│ CLIENT (varies, individual)                                     │
│                                                                 │
│  [Download JS] → [Execute JS] → [Fetch Data] → [Build DOM]     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

Server rendering: Server does more work, client does less
Client rendering: Server does less work, client does more

### How Server-Side Rendering Works Internally

When a server renders HTML, it runs your component code to produce a string:

```javascript
// What your React component looks like
function ProductPage({ product }) {
  return (
    <div className="product">
      <h1>{product.name}</h1>
      <p>${product.price}</p>
    </div>
  );
}

// What the server actually does (simplified)
function renderToString(component) {
  // 1. Execute component function
  const virtualDOM = component({ product: { name: 'Shoes', price: 99 } });
  
  // 2. virtualDOM is a tree structure:
  // {
  //   type: 'div',
  //   props: { className: 'product' },
  //   children: [
  //     { type: 'h1', children: ['Shoes'] },
  //     { type: 'p', children: ['$99'] }
  //   ]
  // }
  
  // 3. Convert tree to HTML string
  return '<div class="product"><h1>Shoes</h1><p>$99</p></div>';
}
```

The server runs JavaScript to produce HTML. It's executing your React/Vue/Svelte code, but instead of updating a browser DOM, it builds a string.

### How Client-Side Rendering Works Internally

CSR works differently - it manipulates the live DOM:

```javascript
// Browser receives empty HTML:
// <div id="root"></div>

// JavaScript bundle executes:
const root = document.getElementById('root');

// Framework creates elements and appends them
const div = document.createElement('div');
div.className = 'product';

const h1 = document.createElement('h1');
h1.textContent = 'Shoes';

const p = document.createElement('p');
p.textContent = '$99';

div.appendChild(h1);
div.appendChild(p);
root.appendChild(div);

// Each DOM operation triggers browser work
```

Every DOM manipulation can trigger layout and paint. This is why virtual DOM exists - to batch changes.

### Why SSG is Fastest: CDN Edge Caching

Static Site Generation's speed comes from CDN distribution:

```
Traditional SSR:
User (Tokyo) → Request → Origin Server (New York) → Process → Response
              [───────────── 200-500ms ──────────────]

Static + CDN:
User (Tokyo) → Request → CDN Edge (Tokyo) → Cache Hit → Response
              [──────────── 20-50ms ────────────]
```

**How CDNs Work:**

```
First Request (cache miss):
1. User requests /products
2. CDN edge has no cache
3. Request goes to origin
4. Origin returns HTML
5. CDN caches it
6. User receives response

Subsequent Requests (cache hit):
1. User requests /products
2. CDN edge has cached HTML
3. Immediate response (no origin contact)
```

With SSG, the HTML is pre-built and distributed to CDN edges worldwide BEFORE any user requests it.

### ISR: How Stale-While-Revalidate Works

ISR combines caching with freshness. Understanding the timeline is key:

```
Build Time: Page generated, cached with revalidate: 60

Timeline:
[0s]     Page built, served to all users
[30s]    User A requests → Gets cached page (age: 30s, still fresh)
[60s]    Page is now "stale" but still cached
[61s]    User B requests → Gets stale page immediately
         Background: Server regenerates page
[62s]    New page ready, replaces old in cache
[70s]    User C requests → Gets fresh page (age: 8s)
```

The key insight: **the user who triggers revalidation gets the stale page**. The NEXT user gets fresh content. This is "stale-while-revalidate" strategy.

### Streaming: How HTTP Chunked Transfer Works

Streaming isn't magic - it uses HTTP's chunked transfer encoding:

```
Normal Response:
HTTP/1.1 200 OK
Content-Length: 5000

[...waits until all 5000 bytes ready...]
[...sends all at once...]


Chunked/Streaming Response:
HTTP/1.1 200 OK
Transfer-Encoding: chunked

100                          ← Chunk size in hex (256 bytes)
<html><head>...</head><body> ← Actual content
0                            ← More chunks coming

200                          ← Next chunk (512 bytes)
<main>Content here...</main>
0

0                            ← Final chunk (0 means done)
```

The browser can start rendering BEFORE the full response arrives.

**React Streaming Example:**

```jsx
// Server Component
async function Page() {
  return (
    <html>
      <body>
        <Header />  {/* Immediate - no data */}
        
        <Suspense fallback={<LoadingSkeleton />}>
          <SlowDataComponent />  {/* Streams when ready */}
        </Suspense>
        
        <Footer />  {/* Immediate - no data */}
      </body>
    </html>
  );
}

// What gets streamed:
// Chunk 1: <html><body><Header>...</Header><LoadingSkeleton />
// [... server fetching SlowDataComponent data ...]
// Chunk 2: <script>swapContent('slow-component', '<SlowData>...</SlowData>')</script>
// Chunk 3: <Footer>...</Footer></body></html>
```

### Time to First Byte (TTFB) vs Time to Last Byte

Understanding these metrics clarifies rendering tradeoffs:

```
CSR Request:
[Request]─────[TTFB: 50ms]─────[TTLB: 100ms]
              Small static file, fast to send

SSR Request (blocking):
[Request]─────────────────────[TTFB: 500ms]─────[TTLB: 520ms]
              Server processing delays first byte

SSR Request (streaming):
[Request]───[TTFB: 50ms]─────────────────────────[TTLB: 500ms]
            Shell immediate      Content streams progressively
```

Streaming improves TTFB dramatically while allowing complex server work.

### The N+1 Problem in Rendering

Both SSR and CSR can suffer from N+1 data fetching:

```javascript
// Waterfall problem
async function ProductsPage() {
  const products = await fetch('/api/products');  // 1 request
  
  return products.map(product => (
    <Product 
      product={product}
      reviews={await fetch(`/api/reviews/${product.id}`)}  // N requests
    />
  ));
}

// If you have 50 products = 51 requests (serial!)
// Each awaits the previous = massive latency
```

**Solution - Parallel fetching:**

```javascript
async function ProductsPage() {
  const products = await fetch('/api/products');
  
  // Fetch all reviews in parallel
  const reviewsPromises = products.map(p => 
    fetch(`/api/reviews/${p.id}`)
  );
  const reviews = await Promise.all(reviewsPromises);
  
  // Now render with all data
}
```

### Cache Invalidation: The Hard Problem

"There are only two hard things in Computer Science: cache invalidation and naming things."

```
Problem Scenario:
1. Page generated with product price $99
2. Price changes to $89 in database
3. Cached page still shows $99
4. User sees wrong price

Cache Strategies:

TIME-BASED (ISR):
- Revalidate every 60 seconds
- Pros: Simple, predictable
- Cons: Up to 60s stale data

EVENT-BASED (On-demand revalidation):
- CMS webhook triggers rebuild
- Pros: Immediate freshness
- Cons: Complex, webhook reliability

CACHE TAGS:
- Tag pages by dependency: ['product-123', 'category-shoes']
- Invalidate by tag when data changes
- Pros: Precise invalidation
- Cons: Complex dependency tracking
```

### Edge Rendering vs Origin Rendering

Modern architectures introduce edge computing:

```
ORIGIN RENDERING (traditional SSR):
User → CDN Edge → Origin Server (single location) → Response
       [20ms]     [─────── 200ms ───────]

EDGE RENDERING:
User → CDN Edge (runs your code) → Response
       [──────── 50ms ───────]
       No origin round-trip needed

EDGE + ORIGIN:
User → Edge (static parts) → Origin (dynamic parts)
       [──── 50ms ────]     [─── 150ms for dynamic ───]
       Shell immediate       Data streams in
```

Edge functions run your server code geographically close to users. Platforms like Cloudflare Workers, Vercel Edge, Deno Deploy enable this.

### Memory and CPU: The Rendering Costs

**CSR Memory Pattern:**

```
Browser Memory Over Time:
[Page Load] → [JS Parsed: 50MB heap] → [App Runs: 80MB] → [Navigation: 100MB] → ...
                                       Memory grows, garbage collected periodically
                                       Memory leaks possible if state not cleaned
```

**SSR Memory Pattern:**

```
Server Memory Per Request:
[Request] → [Render: 20MB allocated] → [Response Sent] → [Memory freed]
            Fresh start each request
            No memory leaks across requests
            But: concurrent requests = concurrent memory
```

**CPU Considerations:**

```
CSR: Each user's device does rendering work
     1000 users = 1000 CPUs doing work (distributed)

SSR: Server does all rendering work
     1000 users = 1 server CPU doing 1000x work (concentrated)
     
Solution: Caching (SSG/ISR) or scaling servers
```

### When to Use What: Deep Analysis

**Choose CSR when:**
- Users are authenticated (can't cache anyway)
- Data is real-time (WebSockets, polling)
- Heavy interactivity (dashboards, editors)
- Server costs must be minimal
- SEO doesn't matter

**Choose SSR when:**
- SEO is critical AND data is dynamic
- Personalization needed (user-specific content)
- Data changes every request
- You can handle server costs
- First paint must include real content

**Choose SSG when:**
- Content changes rarely (docs, blogs)
- All URLs known at build time
- Maximum performance needed
- Minimal server infrastructure
- Content is same for all users

**Choose ISR when:**
- Content changes but not constantly
- SEO needed but data updates
- Can tolerate brief staleness
- Want SSG benefits with some dynamism

**Choose Streaming when:**
- Parts of page have slow data dependencies
- Want fast first paint with SSR
- Page has mixed static/dynamic content
- User experience during load matters

## Related Skills

- See [web-app-architectures](../web-app-architectures/SKILL.md) for SPA vs MPA
- See [seo-fundamentals](../seo-fundamentals/SKILL.md) for SEO implications
- See [hydration-patterns](../hydration-patterns/SKILL.md) for hydration strategies
