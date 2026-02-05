---
name: routing-patterns
description: Explains client-side routing, server-side routing, file-based routing, and navigation patterns in web applications. Use when implementing routing, understanding how navigation works in SPAs vs MPAs, or configuring routes in meta-frameworks.
---

# Routing Patterns

## Overview

Routing determines how URLs map to content and how navigation between pages works. The approach differs significantly between traditional server-rendered apps and modern SPAs.

## Server-Side Routing

**Server determines what to render based on the URL.**

### How It Works

```
1. User navigates to /about
2. Browser sends request to server
3. Server matches /about to handler
4. Server renders HTML
5. Server sends complete HTML response
6. Browser loads new page (full reload)
```

### Characteristics

```
Request: GET /products/123

Server:
  Route table:
    /            → HomeController
    /about       → AboutController
    /products/:id → ProductController  ← matches

  ProductController:
    - Fetches product 123
    - Renders HTML
    - Returns response
```

### Pros and Cons

| Pros | Cons |
|------|------|
| SEO-friendly (full HTML) | Full page reload |
| Simple mental model | Slower navigation |
| Works without JavaScript | State lost between pages |
| Direct URL to content mapping | Server must handle every route |

## Client-Side Routing

**JavaScript handles routing in the browser, no server round-trip for navigation.**

### How It Works

```
1. Initial: Browser loads app shell + JS
2. User clicks link
3. JavaScript intercepts click (preventDefault)
4. JS updates URL using History API
5. JS renders new view/component
6. No server request for HTML
```

### The History API

```javascript
// Push new URL to history (no page reload)
history.pushState({ page: 'about' }, '', '/about');

// Replace current URL
history.replaceState({ page: 'home' }, '', '/');

// Listen for back/forward navigation
window.addEventListener('popstate', (event) => {
  // Render based on current URL
  renderRoute(window.location.pathname);
});
```

### Basic Router Implementation

```javascript
// Simplified client-side router concept
class Router {
  constructor() {
    this.routes = {};
    window.addEventListener('popstate', () => this.handleRoute());
  }

  register(path, component) {
    this.routes[path] = component;
  }

  navigate(path) {
    history.pushState({}, '', path);
    this.handleRoute();
  }

  handleRoute() {
    const path = window.location.pathname;
    const component = this.routes[path] || this.routes['/404'];
    component.render();
  }
}

// Usage
router.register('/about', AboutComponent);
router.register('/products/:id', ProductComponent);
```

### Link Interception

```javascript
// Intercept link clicks for client-side navigation
document.addEventListener('click', (e) => {
  const link = e.target.closest('a');
  if (link && link.href.startsWith(window.location.origin)) {
    e.preventDefault();
    router.navigate(link.pathname);
  }
});
```

## File-Based Routing

**Route structure derived from file system layout.**

### Common Patterns

```
File System                          URL
──────────────────────────────       ─────────────
pages/index.tsx                  →   /
pages/about.tsx                  →   /about
pages/blog/index.tsx             →   /blog
pages/blog/[slug].tsx            →   /blog/:slug
pages/docs/[...path].tsx         →   /docs/*
pages/[category]/[id].tsx        →   /:category/:id
```

### Framework Implementations

**Next.js (App Router):**
```
app/
├── page.tsx                    → /
├── about/page.tsx              → /about
├── blog/[slug]/page.tsx        → /blog/:slug
└── (marketing)/                → Route group (no URL segment)
    └── pricing/page.tsx        → /pricing
```

**SvelteKit:**
```
src/routes/
├── +page.svelte                → /
├── about/+page.svelte          → /about
├── blog/[slug]/+page.svelte    → /blog/:slug
└── [[optional]]/+page.svelte   → / or /:optional
```

**Remix:**
```
app/routes/
├── _index.tsx                  → /
├── about.tsx                   → /about
├── blog._index.tsx             → /blog
├── blog.$slug.tsx              → /blog/:slug
└── $.tsx                       → Catch-all (splat)
```

## Dynamic Routes

### Parameter Types

```
Static:     /about          → Exact match
Dynamic:    /blog/[slug]    → Single parameter
Catch-all:  /docs/[...path] → Multiple segments
Optional:   /[[locale]]/    → With or without parameter
```

### Parameter Access

**Concept (framework-agnostic):**
```javascript
// URL: /products/shoes/nike-air-max

// Route: /products/[category]/[id]
// Parameters: { category: 'shoes', id: 'nike-air-max' }

// Route: /products/[...path]
// Parameters: { path: ['shoes', 'nike-air-max'] }
```

## Route Groups and Layouts

### Nested Layouts

```
app/
├── layout.tsx              → Root layout (applies to all)
├── (shop)/
│   ├── layout.tsx          → Shop layout (header, cart)
│   ├── products/page.tsx   → /products (uses shop layout)
│   └── cart/page.tsx       → /cart (uses shop layout)
└── (marketing)/
    ├── layout.tsx          → Marketing layout (different nav)
    ├── about/page.tsx      → /about (uses marketing layout)
    └── pricing/page.tsx    → /pricing (uses marketing layout)
```

### Parallel Routes

Load multiple views simultaneously:

```
app/
├── @sidebar/
│   └── page.tsx            → Sidebar content
├── @main/
│   └── page.tsx            → Main content
└── layout.tsx              → Renders both in parallel
```

### Intercepting Routes

Handle routes differently in different contexts:

```
app/
├── photos/[id]/page.tsx        → Full page view
├── @modal/(.)photos/[id]/       → Modal overlay (when navigating internally)
│   └── page.tsx
└── feed/page.tsx                → Grid with links to photos
```

## Navigation Patterns

### Declarative Navigation

```jsx
// React Router / Next.js
<Link href="/about">About</Link>

// Vue Router
<router-link to="/about">About</router-link>

// SvelteKit
<a href="/about">About</a>  <!-- Enhanced automatically -->
```

### Programmatic Navigation

```javascript
// React Router
const navigate = useNavigate();
navigate('/dashboard');

// Next.js
const router = useRouter();
router.push('/dashboard');

// Vue Router
router.push('/dashboard');

// SvelteKit
import { goto } from '$app/navigation';
goto('/dashboard');
```

### Navigation Options

```javascript
// Replace history (no back button)
router.replace('/login');

// With query parameters
router.push('/search?q=term');

// With hash
router.push('/docs#section');

// Scroll to top
router.push('/page', { scroll: true });

// Shallow routing (no data refetch)
router.push('/page', { shallow: true });
```

## Route Protection

### Authentication Guards

```javascript
// Middleware pattern (Next.js)
export function middleware(request) {
  const token = request.cookies.get('auth-token');
  
  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
}

// Route-level check
export async function loader({ request }) {
  const user = await getUser(request);
  if (!user) {
    throw redirect('/login');
  }
  return { user };
}
```

### Authorization Patterns

```javascript
// Role-based routing
if (!user.roles.includes('admin')) {
  redirect('/unauthorized');
}

// Feature flags
if (!features.newDashboard) {
  redirect('/legacy-dashboard');
}
```

## URL State Management

### Query Parameters

```javascript
// Read query params
const searchParams = useSearchParams();
const query = searchParams.get('q');

// Update query params
const params = new URLSearchParams(searchParams);
params.set('page', '2');
router.push(`/products?${params.toString()}`);
```

### Hash-Based State

```javascript
// URL: /docs#installation
// Scroll to section
useEffect(() => {
  const hash = window.location.hash;
  if (hash) {
    document.querySelector(hash)?.scrollIntoView();
  }
}, []);
```

### State in URL vs Component

```
URL State (shareable, bookmarkable):
  - Current page, filters, search query
  - Tab selection
  - Modal open state (sometimes)

Component State (ephemeral):
  - Form input before submission
  - Hover/focus states
  - Temporary UI state
```

## Performance Patterns

### Prefetching

```jsx
// Next.js - automatic on hover
<Link href="/about" prefetch={true}>About</Link>

// Manual prefetch
router.prefetch('/dashboard');
```

### Code Splitting by Route

```javascript
// Lazy load route components
const Dashboard = lazy(() => import('./Dashboard'));

// Routes load only when accessed
<Route path="/dashboard" element={
  <Suspense fallback={<Loading />}>
    <Dashboard />
  </Suspense>
} />
```

### Route Transitions

```javascript
// Loading states during navigation
const navigation = useNavigation();

{navigation.state === 'loading' && <LoadingBar />}
```

## Common Routing Pitfalls

### 1. Breaking the Back Button

```javascript
// Bad: Replaces every navigation
router.replace('/page');  // Can't go back

// Good: Push for normal navigation
router.push('/page');  // Back button works
```

### 2. Not Handling 404s

```javascript
// Ensure catch-all route exists
// app/[...not-found]/page.tsx or pages/404.tsx
```

### 3. Client-Only Routes in SSR

```javascript
// Bad: useSearchParams in server component
// Causes hydration mismatch

// Good: Use in client component or read in middleware
```

### 4. Hash URLs for SEO Content

```javascript
// Bad: /#/products/123 (not indexable)
// Good: /products/123 (indexable)
```

## Related Skills

- See [web-app-architectures](../web-app-architectures/SKILL.md) for SPA vs MPA routing
- See [meta-frameworks-overview](../meta-frameworks-overview/SKILL.md) for framework-specific routing
- See [seo-fundamentals](../seo-fundamentals/SKILL.md) for SEO-friendly URLs
