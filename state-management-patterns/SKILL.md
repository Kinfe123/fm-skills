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

---

## Deep Dive: Understanding State From First Principles

### What Is State, Really?

State is any data that changes over time and affects what the user sees or can do. Understanding this precisely helps you manage it correctly:

```javascript
// STATE: Data that changes and triggers UI updates

// Example 1: Counter
let count = 0;  // ← This is state (changes on click)
button.onclick = () => {
  count++;
  display.textContent = count;  // Must update UI manually!
};

// Example 2: Form input
let inputValue = '';  // ← This is state (changes on type)

// Example 3: Fetched data
let products = null;  // ← This is state (changes on fetch complete)


// NOT STATE: Static data that never changes
const API_URL = 'https://api.example.com';  // Configuration, not state
const TAX_RATE = 0.08;  // Constant, not state


// THE PROBLEM: Keeping UI in sync with state
// Without a framework, YOU must update the DOM manually every time state changes
// Frameworks automate this: state change → automatic re-render
```

### The Reactivity Problem

Why do frameworks exist? Because keeping UI in sync with state is HARD:

```javascript
// VANILLA JAVASCRIPT - Manual synchronization

const state = {
  user: null,
  posts: [],
  selectedPost: null,
  isLoading: false,
};

// Every state change requires manual DOM updates
function setUser(user) {
  state.user = user;
  
  // Now manually update every place user is displayed:
  document.getElementById('user-name').textContent = user.name;
  document.getElementById('user-avatar').src = user.avatar;
  document.getElementById('welcome-message').textContent = `Welcome, ${user.name}`;
  // ... and any other places
  // Miss one? Bug. UI and state are out of sync.
}

function setPosts(posts) {
  state.posts = posts;
  
  // Clear and rebuild entire list
  const container = document.getElementById('posts');
  container.innerHTML = '';
  posts.forEach(post => {
    const el = document.createElement('div');
    el.textContent = post.title;
    el.onclick = () => setSelectedPost(post);
    container.appendChild(el);
  });
}

// 100 state variables = 100 manual update functions
// Complex UIs become unmanageable
```

**The framework solution:**

```jsx
// REACT - Declarative UI based on state

function App() {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);
  
  // Describe WHAT UI should look like given state
  // React figures out HOW to update DOM
  return (
    <div>
      {user && <span>Welcome, {user.name}</span>}
      {posts.map(post => (
        <div key={post.id}>{post.title}</div>
      ))}
    </div>
  );
}

// Change state → React re-renders → DOM automatically updated
// You never touch the DOM directly
```

### Memory Model: Where State Lives

Understanding memory helps you understand state:

```
BROWSER MEMORY MODEL:

┌─────────────────────────────────────────────────────────────────┐
│                        JavaScript Heap                           │
│                                                                  │
│  ┌──────────────────────┐  ┌──────────────────────┐             │
│  │   Component State    │  │   Closure Scopes     │             │
│  │   (useState, etc.)   │  │   (functions, refs)  │             │
│  └──────────────────────┘  └──────────────────────┘             │
│                                                                  │
│  ┌──────────────────────┐  ┌──────────────────────┐             │
│  │   Module Scope       │  │   Global Variables   │             │
│  │   (imports, consts)  │  │   (window.*)         │             │
│  └──────────────────────┘  └──────────────────────┘             │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        Browser Storage                           │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌───────────┐ │
│  │localStorage│  │sessionStor.│  │  Cookies   │  │ IndexedDB │ │
│  │  (~5MB)    │  │   (~5MB)   │  │   (~4KB)   │  │  (large)  │ │
│  └────────────┘  └────────────┘  └────────────┘  └───────────┘ │
└─────────────────────────────────────────────────────────────────┘

JavaScript Heap: Cleared on page reload
Browser Storage: Persists across reloads (until cleared)
```

**State lifecycle in SPAs:**

```javascript
// 1. Page loads, JavaScript executes
// Heap is empty

// 2. Components mount, state initialized
const [count, setCount] = useState(0);  // Allocated in heap

// 3. User interacts, state changes
setCount(1);  // New value in heap, old garbage collected

// 4. Navigation within SPA
// Heap PERSISTS - state survives navigation!

// 5. Page reload or close
// Heap DESTROYED - all in-memory state lost
// Only browser storage survives
```

### State Updates: Synchronous vs Asynchronous

React batches state updates for performance:

```javascript
// You might expect this to increment by 3:
function handleClick() {
  setCount(count + 1);  // count is 0, sets to 1
  setCount(count + 1);  // count is STILL 0, sets to 1
  setCount(count + 1);  // count is STILL 0, sets to 1
}
// Result: count is 1, not 3!

// WHY?
// React batches updates and uses the SAME count value for all three
// count doesn't change until the next render

// SOLUTION: Functional updates
function handleClick() {
  setCount(c => c + 1);  // c is 0, returns 1
  setCount(c => c + 1);  // c is 1, returns 2
  setCount(c => c + 1);  // c is 2, returns 3
}
// Result: count is 3!

// Each function receives the LATEST pending state value
```

### Immutability: Why It Matters

React uses reference equality to detect changes:

```javascript
// MUTABLE UPDATE (WRONG)
const [user, setUser] = useState({ name: 'John', age: 30 });

function birthday() {
  user.age = 31;  // Mutating existing object
  setUser(user);  // Same reference!
}
// React sees: old reference === new reference
// React thinks: "nothing changed, skip re-render"
// Bug: UI doesn't update!


// IMMUTABLE UPDATE (CORRECT)
function birthday() {
  setUser({ ...user, age: 31 });  // New object!
}
// React sees: old reference !== new reference
// React knows: "state changed, re-render"


// NESTED IMMUTABILITY
const [state, setState] = useState({
  user: {
    profile: {
      name: 'John',
      settings: { theme: 'dark' }
    }
  }
});

// Updating deeply nested property:
setState({
  ...state,
  user: {
    ...state.user,
    profile: {
      ...state.user.profile,
      settings: {
        ...state.user.profile.settings,
        theme: 'light'  // The actual change
      }
    }
  }
});

// This is verbose! Libraries like Immer help:
setState(produce(state, draft => {
  draft.user.profile.settings.theme = 'light';
}));
```

### Context: How It Actually Works

React Context creates a "wormhole" through the component tree:

```javascript
// WITHOUT CONTEXT: Prop drilling
<App user={user}>
  <Layout user={user}>
    <Sidebar user={user}>
      <UserMenu user={user} />   // Finally used here!
    </Sidebar>
  </Layout>
</App>
// Every intermediate component must forward the prop


// WITH CONTEXT: Direct access
const UserContext = createContext(null);

<UserContext.Provider value={user}>
  <App>
    <Layout>
      <Sidebar>
        <UserMenu />  // Can access user directly!
      </Sidebar>
    </Layout>
  </App>
</UserContext.Provider>

function UserMenu() {
  const user = useContext(UserContext);  // "Wormhole" access
  return <span>{user.name}</span>;
}
```

**Context performance gotcha:**

```javascript
// PROBLEM: Context changes re-render ALL consumers

const AppContext = createContext();

function App() {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');
  const [notifications, setNotifications] = useState([]);
  
  // Every state in one context
  return (
    <AppContext.Provider value={{ user, theme, notifications }}>
      <Header />      {/* Re-renders when ANYTHING changes */}
      <Main />        {/* Re-renders when ANYTHING changes */}
      <Footer />      {/* Re-renders when ANYTHING changes */}
    </AppContext.Provider>
  );
}

// SOLUTION: Split contexts by update frequency

<UserContext.Provider value={user}>
  <ThemeContext.Provider value={theme}>
    <NotificationContext.Provider value={notifications}>
      <App />
    </NotificationContext.Provider>
  </ThemeContext.Provider>
</UserContext.Provider>

// Now: theme change only re-renders theme consumers
```

### Server State: A Different Beast

Server state has fundamentally different characteristics:

```javascript
// CLIENT STATE:
// - Created locally
// - You control it completely
// - Synchronous access
// - Always fresh (it's the source of truth)

const [isOpen, setIsOpen] = useState(false);
// isOpen is ALWAYS correct - you just set it


// SERVER STATE:
// - Created remotely
// - You have a COPY, not the source
// - Asynchronous access
// - Potentially stale (someone else may have changed it)

const [products, setProducts] = useState(null);

useEffect(() => {
  fetch('/api/products').then(r => r.json()).then(setProducts);
}, []);

// products might be:
// - null (loading)
// - stale (database changed since fetch)
// - from a failed request (error)
// - cached (do we refetch?)
```

**Why server state libraries exist:**

```javascript
// Manual server state management:
function ProductList() {
  const [products, setProducts] = useState(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(null);
  const [lastFetched, setLastFetched] = useState(null);
  
  const fetchProducts = async () => {
    setIsLoading(true);
    setError(null);
    try {
      const res = await fetch('/api/products');
      if (!res.ok) throw new Error('Failed');
      const data = await res.json();
      setProducts(data);
      setLastFetched(Date.now());
    } catch (e) {
      setError(e);
    } finally {
      setIsLoading(false);
    }
  };
  
  useEffect(() => {
    fetchProducts();
    // Refetch on window focus?
    // Refetch on interval?
    // Cache between components?
    // Cancel on unmount?
  }, []);
  
  // This is getting complex...
}

// With TanStack Query:
function ProductList() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['products'],
    queryFn: () => fetch('/api/products').then(r => r.json()),
    staleTime: 60000,           // Fresh for 1 minute
    refetchOnWindowFocus: true,  // Refetch when user returns
    retry: 3,                    // Retry failed requests
  });
  
  // All the complexity is handled
}
```

### URL State: The Serialization Challenge

URL state must be serializable to a string:

```javascript
// URL: /products?category=shoes&sizes=7,8,9&page=2&sort=price-asc

// STATE OBJECT:
{
  category: 'shoes',
  sizes: [7, 8, 9],
  page: 2,
  sort: { field: 'price', order: 'asc' }
}

// SERIALIZATION CHALLENGES:

// Arrays: Multiple values for same key or comma-separated?
// ?sizes=7&sizes=8&sizes=9  vs  ?sizes=7,8,9

// Objects: How to represent nested data?
// ?sort=price-asc  (custom encoding)
// ?sortField=price&sortOrder=asc  (flattened)

// Types: Everything is a string in URLs
// ?page=2  → page is "2" (string), need to parse to number

// MANAGING URL STATE:
import { useSearchParams } from 'next/navigation';

function ProductsPage() {
  const searchParams = useSearchParams();
  
  // Reading
  const category = searchParams.get('category');
  const page = parseInt(searchParams.get('page') || '1');
  const sizes = searchParams.get('sizes')?.split(',').map(Number) || [];
  
  // Writing
  function setFilters(newFilters) {
    const params = new URLSearchParams();
    if (newFilters.category) params.set('category', newFilters.category);
    if (newFilters.page > 1) params.set('page', String(newFilters.page));
    if (newFilters.sizes.length) params.set('sizes', newFilters.sizes.join(','));
    
    router.push(`/products?${params.toString()}`);
  }
}
```

### State Machines: Eliminating Impossible States

Complex state often has implicit rules:

```javascript
// BOOLEAN SOUP:
const [isLoading, setIsLoading] = useState(false);
const [isError, setIsError] = useState(false);
const [isSuccess, setIsSuccess] = useState(false);
const [data, setData] = useState(null);

// BUGS WAITING TO HAPPEN:
// - What if isLoading AND isError are both true?
// - What if isSuccess is true but data is null?
// - Caller must set multiple booleans correctly

async function fetchData() {
  setIsLoading(true);
  setIsError(false);    // Don't forget!
  setIsSuccess(false);  // Don't forget!
  
  try {
    const result = await fetch('/api/data');
    setData(result);
    setIsSuccess(true);
  } catch (e) {
    setIsError(true);
  } finally {
    setIsLoading(false);
  }
}


// STATE MACHINE APPROACH:
type State = 
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success', data: Data }
  | { status: 'error', error: Error };

// ONLY ONE STATE POSSIBLE AT A TIME
const [state, setState] = useState<State>({ status: 'idle' });

async function fetchData() {
  setState({ status: 'loading' });  // One call, complete transition
  
  try {
    const data = await fetch('/api/data');
    setState({ status: 'success', data });  // One call, complete transition
  } catch (error) {
    setState({ status: 'error', error });   // One call, complete transition
  }
}

// USAGE:
switch (state.status) {
  case 'idle': return <button onClick={fetchData}>Load</button>;
  case 'loading': return <Spinner />;
  case 'success': return <DataView data={state.data} />;
  case 'error': return <Error message={state.error.message} />;
}
// TypeScript knows exactly what's available in each case
```

### The Single Source of Truth Principle

Every piece of state should have ONE authoritative location:

```javascript
// VIOLATION: Same data in multiple places

// User data in context
const userContext = { user: { name: 'John' } };

// User data also in component state
const [userData, setUserData] = useState({ name: 'John' });

// User data also in form state
const [formName, setFormName] = useState('John');

// NOW: Which is correct if they differ?
// - User edits form: formName = 'Jane'
// - userData still says 'John'
// - userContext still says 'John'
// CONFLICT!


// CORRECT: Single source of truth

// User data lives in ONE place (context)
const { user, updateUser } = useUserContext();

// Form uses a DRAFT (clearly temporary)
const [draft, setDraft] = useState(user.name);

// On save, update the source of truth
function handleSave() {
  updateUser({ ...user, name: draft });
  // Now everything derives from the same source
}

// Components READ from the source
function Header() {
  const { user } = useUserContext();
  return <span>{user.name}</span>;  // Always reflects truth
}
```

## Related Skills

- See [web-app-architectures](../web-app-architectures/SKILL.md) for SPA state persistence
- See [rendering-patterns](../rendering-patterns/SKILL.md) for server vs client state
- See [routing-patterns](../routing-patterns/SKILL.md) for URL state
