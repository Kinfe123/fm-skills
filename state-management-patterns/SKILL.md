---
name: state-management-patterns
description: Explains state management in web applications including client state, server state, URL state, and caching strategies. Use when discussing where to store state, choosing between local and global state, or implementing data fetching patterns.
---

# State Management Patterns

## Overview

State is data that changes over time. Where and how you manage state significantly impacts application architecture, performance, and complexity.

## Types of State

| Type | Examples | Characteristics |
|------|----------|-----------------|
| **UI State** | Modal open, tab selected, form input | Local, ephemeral |
| **Server State** | User data, products, posts | Remote, cached, async |
| **URL State** | Page, filters, search query | Shareable, bookmarkable |
| **Browser State** | localStorage, sessionStorage, cookies | Persistent, limited |
| **Application State** | Auth, theme, user preferences | Global, session-scoped |

## Client State vs Server State

### Client State

Data that exists only in the browser:

```javascript
// UI state - component-local
const [isOpen, setIsOpen] = useState(false);

// Application state - global
const [theme, setTheme] = useContext(ThemeContext);
```

### Server State

Data fetched from a server:

```javascript
// Different characteristics
const { data, loading, error } = useQuery('/api/products');

// Server state is:
// - Potentially stale
// - Requires caching strategy
// - Async (loading, error states)
// - Shared across components
```

## State Colocation

**Place state as close as possible to where it's used.**

### Component State (Local)

```jsx
// Good: State used only in this component
function Toggle() {
  const [on, setOn] = useState(false);
  return <button onClick={() => setOn(!on)}>{on ? 'On' : 'Off'}</button>;
}
```

### Lifted State (Shared by Siblings)

```jsx
// Good: State shared between siblings, lifted to parent
function Form() {
  const [value, setValue] = useState('');
  return (
    <>
      <Input value={value} onChange={setValue} />
      <Preview value={value} />
    </>
  );
}
```

### Context (Deep Tree Access)

```jsx
// Good: State needed by many components at different levels
<ThemeProvider value={theme}>
  <App />  {/* Any child can access theme */}
</ThemeProvider>
```

### Global State (Application-Wide)

```javascript
// Use sparingly for truly global concerns:
// - Authentication state
// - Feature flags
// - User preferences
```

## Server State Management

### The Problem

```javascript
// Without caching
function ProductList() {
  const [products, setProducts] = useState([]);
  
  useEffect(() => {
    fetch('/api/products').then(r => r.json()).then(setProducts);
  }, []);
  // Every mount = new request
  // No loading state handling
  // No error handling
  // No caching
}
```

### Server State Libraries

Libraries like TanStack Query, SWR, Apollo handle:

```javascript
// With TanStack Query
function ProductList() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['products'],
    queryFn: () => fetch('/api/products').then(r => r.json()),
    staleTime: 60000,  // Fresh for 1 minute
  });
}
```

Benefits:
- Automatic caching and deduplication
- Background refetching
- Loading and error states
- Cache invalidation
- Optimistic updates

## URL State

**State that should survive refresh and be shareable.**

### When to Use URL State

- Pagination: `/products?page=2`
- Filters: `/products?category=shoes&size=10`
- Search: `/search?q=query`
- Tabs/views: `/dashboard?view=analytics`
- Modal content: `/products/123` with modal

### Managing URL State

```javascript
// Read from URL
const searchParams = useSearchParams();
const page = searchParams.get('page') || '1';

// Update URL
function setPage(newPage) {
  const params = new URLSearchParams(searchParams);
  params.set('page', newPage);
  router.push(`?${params.toString()}`);
}
```

### URL vs Local State

| State | URL | Local |
|-------|-----|-------|
| Current search query | ✓ | |
| Search input (typing) | | ✓ |
| Selected filters | ✓ | |
| Filter dropdown open | | ✓ |
| Current page | ✓ | |
| Scroll position | | ✓ |

## Browser Storage

### Comparison

| Storage | Size | Persistence | Access |
|---------|------|-------------|--------|
| localStorage | ~5MB | Until cleared | Same origin |
| sessionStorage | ~5MB | Tab session | Same tab |
| Cookies | ~4KB | Configurable | Sent to server |
| IndexedDB | Large | Until cleared | Same origin |

### Common Patterns

```javascript
// Persist user preferences
const theme = localStorage.getItem('theme') || 'light';

// Session-specific state
const draftId = sessionStorage.getItem('draft-id');

// Auth tokens (consider security implications)
// HttpOnly cookies preferred for tokens
```

## Server Components and State

### React Server Components

```jsx
// Server Component - no useState, no client-side state
async function ProductPage({ id }) {
  const product = await db.products.findById(id);  // Direct DB access
  return <ProductDetails product={product} />;
}

// Client Component - for interactivity
'use client';
function AddToCart({ productId }) {
  const [quantity, setQuantity] = useState(1);
  return <button onClick={() => addToCart(productId, quantity)}>Add</button>;
}
```

### Server State is Simpler

```
Server Component Approach:
  Database → Server Component → HTML to client
  (No loading states needed, data already there)

Client Fetch Approach:
  Server Component → Client Component → Fetch → Loading → Data → Render
  (More complexity, but necessary for interactivity)
```

## Caching Strategies

### Cache-First (Stale-While-Revalidate)

```javascript
// Show cached data immediately, refetch in background
const { data } = useQuery({
  queryKey: ['products'],
  staleTime: 60000,      // Cached data considered fresh for 1 min
  cacheTime: 300000,     // Keep in cache for 5 min
});
```

### Network-First

```javascript
// Always fetch fresh, fall back to cache
const { data } = useQuery({
  queryKey: ['products'],
  staleTime: 0,          // Always refetch
  networkMode: 'always',
});
```

### Cache Invalidation

```javascript
// After mutation, invalidate related queries
const mutation = useMutation({
  mutationFn: updateProduct,
  onSuccess: () => {
    queryClient.invalidateQueries(['products']);
  },
});
```

## State Machine Patterns

For complex state transitions:

```javascript
// Instead of multiple booleans
const [isLoading, setIsLoading] = useState(false);
const [isError, setIsError] = useState(false);
const [isSuccess, setIsSuccess] = useState(false);
// Problem: Can be in impossible states (loading AND error)

// Use a state machine
const [state, setState] = useState('idle');
// 'idle' | 'loading' | 'success' | 'error'
// Impossible to be in multiple states
```

## Anti-Patterns

### 1. Prop Drilling

```jsx
// Bad: Passing through many levels
<App user={user}>
  <Layout user={user}>
    <Page user={user}>
      <Header user={user}>
        <Avatar user={user} />

// Good: Use context for deep access
<UserContext.Provider value={user}>
  <App />
```

### 2. Global State for Local Needs

```jsx
// Bad: Everything in global store
dispatch({ type: 'SET_MODAL_OPEN', payload: true });

// Good: Local state for UI
const [isOpen, setIsOpen] = useState(false);
```

### 3. Duplicating Server State

```jsx
// Bad: Copying server state to local state
const [products, setProducts] = useState([]);
useEffect(() => {
  fetch('/api/products').then(r => r.json()).then(setProducts);
}, []);

// Good: Server state library handles caching
const { data: products } = useQuery(['products'], fetchProducts);
```

### 4. Not Considering URL State

```jsx
// Bad: Filter state lost on refresh
const [filters, setFilters] = useState({ category: 'all' });

// Good: Filters in URL
const filters = Object.fromEntries(useSearchParams());
```

## Decision Framework

```
Is this UI state (ephemeral, local)?
├── Yes → useState in component
│
└── No → Is it shared between siblings?
          ├── Yes → Lift state to parent
          │
          └── No → Is it needed deep in tree?
                    ├── Yes → Context or global state
                    │
                    └── No → Is it from a server?
                              ├── Yes → Server state library
                              │
                              └── No → Should it survive refresh?
                                        ├── Yes → URL or localStorage
                                        └── No → Component state
```

## Related Skills

- See [web-app-architectures](../web-app-architectures/SKILL.md) for SPA state persistence
- See [rendering-patterns](../rendering-patterns/SKILL.md) for server vs client state
- See [routing-patterns](../routing-patterns/SKILL.md) for URL state
