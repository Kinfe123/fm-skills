---
name: data-fetching-patterns
description: Explains data fetching strategies including fetch on render, fetch then render, render as you fetch, and server-side data fetching. Use when implementing data loading, optimizing loading performance, or choosing between client and server data fetching.
---

# Data Fetching Patterns

## Overview

Data fetching is how your application retrieves data from APIs or databases. The pattern you choose affects performance, user experience, and code complexity.

## Fetching Locations

| Where | When Executed | Use Case |
|-------|---------------|----------|
| Server (build) | Build time | Static content (SSG) |
| Server (request) | Each request | Dynamic content (SSR) |
| Client (browser) | After hydration | Interactive, real-time |
| Edge | At CDN edge | Personalization, A/B tests |

## Client-Side Patterns

### 1. Fetch on Render (Waterfall)

Components fetch data when they mount.

```jsx
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]);
  
  if (!user) return <Loading />;
  return <Profile user={user} />;
}
```

**Timeline:**

```
Component renders → useEffect runs → Fetch starts → Data arrives → Re-render
                    [-- Waiting --]                 [-- Display --]
```

**Problem: Waterfalls**

```jsx
function Dashboard() {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    fetchUser().then(setUser);  // Fetch 1
  }, []);
  
  if (!user) return <Loading />;
  
  return (
    <UserPosts userId={user.id} />  // Fetch 2 starts AFTER Fetch 1 completes
  );
}

// Timeline:
// [Fetch User]───────►[Fetch Posts]───────►Display
//                     ↑ Can't start until user loads (waterfall)
```

### 2. Fetch Then Render

Fetch all data before rendering any component.

```jsx
function Dashboard() {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    Promise.all([fetchUser(), fetchPosts(), fetchStats()])
      .then(([user, posts, stats]) => setData({ user, posts, stats }));
  }, []);
  
  if (!data) return <Loading />;
  
  return (
    <>
      <UserInfo user={data.user} />
      <Posts posts={data.posts} />
      <Stats stats={data.stats} />
    </>
  );
}
```

**Timeline:**

```
[Fetch User  ]
[Fetch Posts ]──►All Complete───►Render All
[Fetch Stats ]
              ↑ Parallel, but wait for slowest
```

**Problem:** All-or-nothing loading. Fast data waits for slow data.

### 3. Render as You Fetch (Concurrent)

Start fetching immediately, render components as data arrives.

```jsx
// Start fetches immediately (not in useEffect)
const userPromise = fetchUser();
const postsPromise = fetchPosts();

function Dashboard() {
  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserInfo userPromise={userPromise} />
    </Suspense>
    <Suspense fallback={<PostsSkeleton />}>
      <Posts postsPromise={postsPromise} />
    </Suspense>
  );
}

// Each component renders when its data is ready
function UserInfo({ userPromise }) {
  const user = use(userPromise);  // React 19's use() hook
  return <div>{user.name}</div>;
}
```

**Timeline:**

```
[Fetch User  ]───►Render User
[Fetch Posts ]─────────►Render Posts
              ↑ Independent, progressive rendering
```

## Server-Side Patterns

### Server Data Fetching (SSR)

Data fetched on server before sending HTML:

```javascript
// Next.js App Router
async function ProductPage({ params }) {
  const product = await db.products.findById(params.id);  // Runs on server
  return <ProductDetails product={product} />;
}
```

**Benefits:**
- No loading state (data already in HTML)
- SEO-friendly (content in initial HTML)
- Direct database/API access

### Parallel Data Fetching

Fetch multiple resources simultaneously:

```javascript
// Bad: Sequential (waterfall)
const user = await getUser();
const posts = await getPosts(user.id);
const comments = await getComments(posts[0].id);

// Good: Parallel where possible
const [user, stats] = await Promise.all([
  getUser(),
  getStats(),
]);
// Then sequential for dependent data
const posts = await getPosts(user.id);
```

### Streaming and Suspense

Send HTML progressively as data arrives:

```jsx
// Layout streams immediately
// Slow component streams when ready
async function Page() {
  return (
    <div>
      <Header />  {/* Immediate */}
      <Suspense fallback={<Loading />}>
        <SlowComponent />  {/* Streams when ready */}
      </Suspense>
      <Footer />  {/* Immediate */}
    </div>
  );
}
```

## Caching Patterns

### Request Deduplication

Multiple components requesting same data should share request:

```javascript
// Without deduplication
// Component A: fetch('/api/user')
// Component B: fetch('/api/user')
// = 2 requests

// With deduplication (React cache / TanStack Query)
// Component A: useQuery(['user'], fetchUser)
// Component B: useQuery(['user'], fetchUser)
// = 1 request, shared result
```

### Stale-While-Revalidate

Show cached data immediately, update in background:

```javascript
const { data } = useQuery({
  queryKey: ['products'],
  queryFn: fetchProducts,
  staleTime: 60000,      // Fresh for 60s
  refetchOnMount: true,  // Background refetch
});

// Timeline:
// 1. Show cached data immediately
// 2. Check if stale
// 3. If stale, refetch in background
// 4. Update UI when fresh data arrives
```

### Cache Invalidation

Clear or update cache after mutations:

```javascript
const mutation = useMutation({
  mutationFn: createPost,
  onSuccess: () => {
    // Invalidate and refetch
    queryClient.invalidateQueries(['posts']);
    
    // Or update cache directly
    queryClient.setQueryData(['posts'], (old) => [...old, newPost]);
  },
});
```

## Loading State Patterns

### Skeleton Screens

Show content structure while loading:

```jsx
function ProductCard({ product }) {
  if (!product) {
    return (
      <div className="card">
        <div className="skeleton-image" />
        <div className="skeleton-text" />
        <div className="skeleton-text short" />
      </div>
    );
  }
  return (
    <div className="card">
      <img src={product.image} />
      <h3>{product.name}</h3>
      <p>{product.price}</p>
    </div>
  );
}
```

### Optimistic Updates

Update UI immediately, rollback on error:

```javascript
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    // Cancel outgoing refetches
    await queryClient.cancelQueries(['todos']);
    
    // Snapshot previous value
    const previous = queryClient.getQueryData(['todos']);
    
    // Optimistically update
    queryClient.setQueryData(['todos'], (old) =>
      old.map(t => t.id === newTodo.id ? newTodo : t)
    );
    
    return { previous };
  },
  onError: (err, newTodo, context) => {
    // Rollback on error
    queryClient.setQueryData(['todos'], context.previous);
  },
});
```

### Placeholder Data

Show estimated/placeholder data while fetching:

```javascript
const { data } = useQuery({
  queryKey: ['product', id],
  queryFn: () => fetchProduct(id),
  placeholderData: {
    name: 'Loading...',
    price: '--',
    image: '/placeholder.jpg',
  },
});
```

## Error Handling

### Error Boundaries

Catch and display errors gracefully:

```jsx
<ErrorBoundary fallback={<ErrorMessage />}>
  <Suspense fallback={<Loading />}>
    <DataComponent />
  </Suspense>
</ErrorBoundary>
```

### Retry Strategies

```javascript
const { data } = useQuery({
  queryKey: ['products'],
  queryFn: fetchProducts,
  retry: 3,                    // Retry 3 times
  retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30000),  // Exponential backoff
});
```

## Pattern Comparison

| Pattern | First Content | Data Freshness | Complexity |
|---------|---------------|----------------|------------|
| Client fetch (useEffect) | Slow | Real-time capable | Low |
| SSR | Fast | Request-time | Medium |
| SSG | Fastest | Build-time | Low |
| ISR | Fastest | Periodic | Medium |
| Streaming | Progressive | Request-time | Medium |

## Decision Framework

```
Is the data static or rarely changes?
├── Yes → SSG or ISR
│
└── No → Does the page need SEO?
          ├── Yes → SSR or Streaming
          │
          └── No → Does it need real-time updates?
                    ├── Yes → Client-side + WebSocket/polling
                    │
                    └── No → Is initial load critical?
                              ├── Yes → SSR + Client hydration
                              └── No → Client-side fetching
```

## Anti-Patterns

### 1. Fetching in useEffect Without Cleanup

```javascript
// Bad: Race condition possible
useEffect(() => {
  fetchData().then(setData);
}, [id]);

// Good: Cleanup or use library
useEffect(() => {
  let cancelled = false;
  fetchData().then(data => {
    if (!cancelled) setData(data);
  });
  return () => { cancelled = true; };
}, [id]);
```

### 2. Not Handling Loading/Error States

```javascript
// Bad: No loading or error handling
const { data } = useQuery(['products'], fetchProducts);
return <ProductList products={data} />;  // data might be undefined

// Good: Handle all states
if (isLoading) return <Loading />;
if (error) return <Error message={error.message} />;
return <ProductList products={data} />;
```

### 3. Over-fetching

```javascript
// Bad: Fetching more than needed
const { data: user } = useQuery(['user'], () => 
  fetch('/api/users/full-profile')  // 50 fields
);
// Only using: user.name, user.avatar

// Good: Fetch only what's needed (or use GraphQL)
const { data: user } = useQuery(['user-summary'], () =>
  fetch('/api/users/summary')  // 3 fields
);
```

---

## Deep Dive: Understanding Data Fetching From First Principles

### The Network Request Lifecycle

Every data fetch involves multiple steps:

```
CLIENT                          NETWORK                         SERVER

1. fetch() called                                               
         │                                                      
2. DNS Lookup ─────────────────► DNS Server                    
         │                       │                              
         │◄──────────────────── IP: 93.184.216.34              
         │                                                      
3. TCP Handshake ─────────────────────────────────────────────► SYN
         │                                                      │
         │◄─────────────────────────────────────────────────── SYN-ACK
         │                                                      
         │───────────────────────────────────────────────────► ACK
         │                                                      
4. TLS Handshake (HTTPS) ─────────────────────────────────────► 
         │◄─────────────────────────────────────────────────── 
         │                                                      
5. HTTP Request ──────────────────────────────────────────────► 
         │                     GET /api/data HTTP/1.1           
         │                     Host: example.com                │
         │                                                      ▼
         │                                              6. Server processes
         │                                                      │
         │                                              7. Query database
         │                                                      │
         │                                              8. Build response
         │                                                      │
         │◄──────────────────────────────────────────────────── 
         │                     HTTP/1.1 200 OK                  
         │                     {"data": [...]}                  
         │                                                      
9. Parse response                                               
         │                                                      
10. Update state                                                
         │                                                      
11. Re-render                                                   
```

**Time breakdown (typical):**

```
DNS Lookup:         10-100ms (cached: 0ms)
TCP Handshake:      50-100ms (1 round trip)
TLS Handshake:      50-150ms (2 round trips)
Request/Response:   50-500ms (depends on server + data size)
Parsing:            1-10ms
Rendering:          5-50ms
────────────────────────────────
TOTAL:              166-910ms
```

### The Waterfall Problem Explained

Waterfalls occur when requests depend on each other:

```javascript
// WATERFALL CODE:
async function loadDashboard() {
  const user = await fetchUser();           // 200ms ─────┐
  const posts = await fetchPosts(user.id);  // 300ms ─────┼─► WAIT
  const comments = await fetchComments(posts[0].id); // 250ms ─┘
}

// TIMELINE (serial):
// |── fetchUser (200ms) ──|── fetchPosts (300ms) ──|── fetchComments (250ms) ──|
// Total: 750ms


// PARALLEL CODE:
async function loadDashboard() {
  // Start all independent fetches simultaneously
  const [user, globalPosts] = await Promise.all([
    fetchUser(),        // 200ms ──┐
    fetchGlobalPosts(), // 300ms ──┤► PARALLEL
  ]);                            // └► Wait for slowest: 300ms
  
  // Dependent fetch still sequential
  const userPosts = await fetchUserPosts(user.id);  // 250ms
}

// TIMELINE (parallel where possible):
// |── fetchUser (200ms) ───|
// |── fetchGlobalPosts (300ms) ──|── fetchUserPosts (250ms) ──|
// Total: 550ms (vs 750ms waterfall)
```

### Why Caching is Essential

Without caching, you make redundant requests:

```javascript
// SCENARIO: Multiple components need user data

function Header() {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetch('/api/user').then(r => r.json()).then(setUser);
  }, []);
  return <span>{user?.name}</span>;
}

function Sidebar() {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetch('/api/user').then(r => r.json()).then(setUser);  // SAME REQUEST!
  }, []);
  return <img src={user?.avatar} />;
}

function Dashboard() {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetch('/api/user').then(r => r.json()).then(setUser);  // SAME REQUEST!
  }, []);
  return <span>Welcome, {user?.name}</span>;
}

// RESULT: 3 identical network requests!
// Each takes 200ms, wasting bandwidth and time
```

**Request deduplication:**

```javascript
// WITH TANSTACK QUERY:
function Header() {
  const { data: user } = useQuery(['user'], fetchUser);
  // First component to request starts the fetch
}

function Sidebar() {
  const { data: user } = useQuery(['user'], fetchUser);
  // Same query key = shares the same request/cache
}

function Dashboard() {
  const { data: user } = useQuery(['user'], fetchUser);
  // All three components share ONE network request
}

// RESULT: 1 network request, shared across components
```

### How Cache Invalidation Works

Caches must be invalidated when data changes:

```javascript
// PROBLEM: Stale data after mutation

// User edits their profile
await updateProfile({ name: 'New Name' });

// But cached user data still shows old name!
// Other components display stale data

// SOLUTION 1: Invalidate and refetch
const queryClient = useQueryClient();

async function handleUpdate(newData) {
  await updateProfile(newData);
  
  // Invalidate the cache - next access will refetch
  queryClient.invalidateQueries(['user']);
}


// SOLUTION 2: Optimistic update + invalidate
async function handleUpdate(newData) {
  // Immediately update cache (optimistic)
  queryClient.setQueryData(['user'], (old) => ({
    ...old,
    ...newData,
  }));
  
  // Then persist to server
  try {
    await updateProfile(newData);
    // Invalidate to get any server-side changes
    queryClient.invalidateQueries(['user']);
  } catch (error) {
    // Rollback on failure
    queryClient.setQueryData(['user'], originalData);
  }
}


// SOLUTION 3: Server returns updated data
async function handleUpdate(newData) {
  const updatedUser = await updateProfile(newData);
  
  // Server returns the new state - update cache directly
  queryClient.setQueryData(['user'], updatedUser);
}
```

### Race Conditions in Data Fetching

When fetches can overlap, you get race conditions:

```javascript
// THE BUG:
function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  
  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then(r => r.json())
      .then(setResults);
  }, [query]);
  
  return <ResultList items={results} />;
}

// SCENARIO:
// User types "re" - starts fetch A (takes 500ms)
// User types "react" - starts fetch B (takes 200ms)
// Fetch B completes first - shows "react" results
// Fetch A completes second - OVERWRITES with "re" results!
// User sees wrong results!


// THE FIX: Abort previous request
function SearchResults({ query }) {
  const [results, setResults] = useState([]);
  
  useEffect(() => {
    const controller = new AbortController();
    
    fetch(`/api/search?q=${query}`, { signal: controller.signal })
      .then(r => r.json())
      .then(setResults)
      .catch(e => {
        if (e.name === 'AbortError') return; // Expected
        throw e;
      });
    
    // Cleanup: abort if query changes or component unmounts
    return () => controller.abort();
  }, [query]);
  
  return <ResultList items={results} />;
}


// OR use a library that handles this:
function SearchResults({ query }) {
  const { data: results } = useQuery({
    queryKey: ['search', query],
    queryFn: () => fetch(`/api/search?q=${query}`).then(r => r.json()),
    // TanStack Query automatically cancels outdated requests
  });
}
```

### Suspense: A New Data Fetching Paradigm

Traditional approach: component manages its loading state
Suspense approach: component delegates loading to boundary

```javascript
// TRADITIONAL: Each component handles loading
function ProductPage() {
  const [product, setProduct] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetchProduct()
      .then(setProduct)
      .finally(() => setLoading(false));
  }, []);
  
  if (loading) return <Spinner />;  // Loading logic in component
  return <Product data={product} />;
}


// SUSPENSE: Loading delegated to boundary
function ProductPage() {
  const product = use(fetchProduct());  // Suspends if not ready
  return <Product data={product} />;    // Only renders when ready
}

// Boundary handles loading
function App() {
  return (
    <Suspense fallback={<Spinner />}>  {/* Loading UI here */}
      <ProductPage />
    </Suspense>
  );
}
```

**How Suspense works internally:**

```javascript
// CONCEPTUAL MECHANISM:

function use(promise) {
  // Check if promise is already resolved
  if (promise.status === 'fulfilled') {
    return promise.value;
  }
  
  if (promise.status === 'rejected') {
    throw promise.reason;
  }
  
  // Not yet resolved - THROW the promise!
  throw promise;
}

// React catches the thrown promise
// Shows nearest Suspense fallback
// When promise resolves, re-renders the component

// This is why Suspense only works with:
// - React's use() hook
// - Libraries that support Suspense (like TanStack Query)
// - Server Components
```

### Streaming: Progressive Data Loading

Streaming sends data as it becomes available:

```javascript
// TRADITIONAL SSR:
// Server must fetch ALL data before sending ANY HTML

async function ProductPage() {
  // These all must complete before response starts
  const product = await getProduct();      // 100ms
  const reviews = await getReviews();       // 500ms  ← SLOW!
  const related = await getRelatedProducts(); // 200ms
  
  // Total wait: 800ms before ANY HTML sent
  return (
    <div>
      <Product data={product} />
      <Reviews data={reviews} />
      <Related data={related} />
    </div>
  );
}


// STREAMING SSR:
// Send what's ready, stream the rest

async function ProductPage() {
  const product = await getProduct();  // 100ms - quick!
  
  return (
    <div>
      <Product data={product} />  {/* Sent immediately */}
      
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews />  {/* Streams when ready */}
      </Suspense>
      
      <Suspense fallback={<RelatedSkeleton />}>
        <Related />  {/* Streams when ready */}
      </Suspense>
    </div>
  );
}

// Client receives:
// t=100ms: Product HTML + skeleton placeholders
// t=300ms: Related products stream in
// t=600ms: Reviews stream in

// User sees content progressively!
```

### Optimistic Updates: The UX Advantage

Optimistic updates show expected result before server confirms:

```javascript
// WITHOUT OPTIMISTIC UPDATE:
// User clicks "Like"
// Spinner shows for 200ms
// Heart fills in
// Feels laggy

async function handleLike() {
  setIsLoading(true);
  await api.likePost(postId);  // 200ms
  await refetchPost();          // Another 200ms
  setIsLoading(false);
  // 400ms of waiting!
}


// WITH OPTIMISTIC UPDATE:
// User clicks "Like"
// Heart fills in IMMEDIATELY
// Server confirms in background

async function handleLike() {
  // Immediately update local state
  setIsLiked(true);
  setLikeCount(c => c + 1);
  
  try {
    await api.likePost(postId);
    // Success! State is already correct
  } catch (error) {
    // Failure! Revert the optimistic update
    setIsLiked(false);
    setLikeCount(c => c - 1);
    showError('Failed to like post');
  }
}

// User perceives INSTANT response
// Only 1 in 100 interactions might need rollback
```

### GraphQL vs REST: Data Fetching Implications

Different protocols have different fetching patterns:

```javascript
// REST: Multiple endpoints, potential over/under-fetching

// Need user profile + their posts + comment counts
// Requires 3 requests:
const user = await fetch('/api/users/123');
const posts = await fetch('/api/users/123/posts');
const comments = await fetch('/api/users/123/posts/comments/count');

// Over-fetching: Each endpoint returns all fields
// User endpoint returns 50 fields, you need 3


// GRAPHQL: Single endpoint, precise data

const query = `
  query UserProfile($id: ID!) {
    user(id: $id) {
      name       # Only fields you need
      avatar
      posts {
        title
        commentCount
      }
    }
  }
`;

const { user } = await graphqlFetch(query, { id: '123' });
// One request, exact data shape you need


// TRADEOFFS:

// REST:
// + Simple, well-understood
// + HTTP caching works naturally
// + Each endpoint cacheable independently
// - Over/under-fetching
// - Multiple round trips

// GRAPHQL:
// + Fetch exactly what you need
// + Single request
// + Strongly typed
// - More complex server setup
// - Caching more complex
// - Potential for expensive queries
```

### Error Handling Strategies

Different errors need different handling:

```javascript
// ERROR TYPES:

// 1. Network errors (offline, timeout)
try {
  await fetch('/api/data');
} catch (e) {
  if (!navigator.onLine) {
    showToast('You appear to be offline');
    // Maybe return cached data
    return cache.get('data');
  }
  if (e.name === 'AbortError') {
    // Request was cancelled, ignore
    return;
  }
  throw e;
}

// 2. HTTP errors (4xx, 5xx)
const response = await fetch('/api/data');
if (!response.ok) {
  if (response.status === 401) {
    // Unauthorized - redirect to login
    router.push('/login');
    return;
  }
  if (response.status === 404) {
    // Not found - show not found UI
    setNotFound(true);
    return;
  }
  if (response.status >= 500) {
    // Server error - retry later
    throw new Error('Server error, please try again');
  }
}

// 3. Application errors (in response body)
const data = await response.json();
if (data.error) {
  // Business logic error
  showError(data.error.message);
  return;
}


// ERROR BOUNDARY PATTERN:
<ErrorBoundary
  fallback={<ErrorPage />}
  onError={(error) => logToSentry(error)}
>
  <Suspense fallback={<Loading />}>
    <DataComponent />
  </Suspense>
</ErrorBoundary>
```

### The Request Deduplication Deep Dive

How libraries prevent duplicate requests:

```javascript
// NAIVE IMPLEMENTATION:
const cache = new Map();
const inFlight = new Map();

async function dedupedFetch(key, fetchFn) {
  // Return cached data if fresh
  if (cache.has(key)) {
    const { data, timestamp } = cache.get(key);
    if (Date.now() - timestamp < STALE_TIME) {
      return data;
    }
  }
  
  // If request already in flight, wait for it
  if (inFlight.has(key)) {
    return inFlight.get(key);
  }
  
  // Start new request
  const promise = fetchFn().then(data => {
    cache.set(key, { data, timestamp: Date.now() });
    inFlight.delete(key);
    return data;
  });
  
  inFlight.set(key, promise);
  return promise;
}

// USAGE:
// Both calls share the SAME network request
const data1 = await dedupedFetch('user', () => fetch('/api/user'));
const data2 = await dedupedFetch('user', () => fetch('/api/user'));
```

## Related Skills

- See [rendering-patterns](../rendering-patterns/SKILL.md) for SSR/SSG context
- See [state-management-patterns](../state-management-patterns/SKILL.md) for caching
- See [hydration-patterns](../hydration-patterns/SKILL.md) for client-side hydration
