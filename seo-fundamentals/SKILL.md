---
name: seo-fundamentals
description: Explains SEO for web applications including crawling, indexing, Core Web Vitals, structured data, and SEO challenges with SPAs. Use when optimizing for search engines, discussing SEO implications of architecture decisions, or implementing SEO best practices.
---

# SEO Fundamentals for Web Applications

## Overview

SEO (Search Engine Optimization) ensures search engines can discover, understand, and rank your content. Architecture decisions significantly impact SEO capability.

## How Search Engines Work

### The Three Phases

```
1. CRAWLING: Googlebot discovers URLs and downloads content
2. INDEXING: Google parses content and stores in search index
3. RANKING: Algorithm determines position in search results
```

### What Crawlers See

Crawlers primarily process the **initial HTML response**. JavaScript execution is:
- Delayed (crawl queue, separate rendering queue)
- Resource-intensive (limited render budget)
- Not guaranteed for all pages

```
Your Server Response          What Googlebot Sees First
─────────────────────         ──────────────────────────
<!DOCTYPE html>               Same HTML (good for SSR/SSG)
<html>
<body>                        OR
  <div id="root"></div>       Empty div (bad for CSR)
  <script src="app.js">       Script tag (won't execute immediately)
</body>
</html>
```

## Architecture Impact on SEO

### SEO by Rendering Pattern

| Pattern | Initial HTML | SEO Quality | Notes |
|---------|--------------|-------------|-------|
| SSG | Complete | Excellent | Best for SEO |
| SSR | Complete | Excellent | Dynamic content, good SEO |
| ISR | Complete | Excellent | Fresh + fast |
| CSR | Empty shell | Poor | Requires workarounds |
| Streaming | Progressive | Good | Shell + streamed content |

### The CSR Problem

```javascript
// What your React app renders client-side
<html>
  <head><title>My App</title></head>
  <body>
    <div id="root">
      <!-- JS renders content here AFTER page load -->
      <!-- Crawler may not wait for this -->
    </div>
  </body>
</html>
```

### Solutions for CSR Apps

1. **Server-Side Rendering (SSR):** Render on server, hydrate on client
2. **Pre-rendering:** Generate static HTML at build time for key pages
3. **Dynamic Rendering:** Serve pre-rendered HTML to bots, SPA to users (not recommended by Google)

## Technical SEO Elements

### Essential Meta Tags

```html
<head>
  <!-- Title: 50-60 characters, unique per page -->
  <title>Product Name - Category | Brand</title>
  
  <!-- Description: 150-160 characters -->
  <meta name="description" content="Compelling description with keywords">
  
  <!-- Canonical: Prevent duplicate content -->
  <link rel="canonical" href="https://example.com/page">
  
  <!-- Robots: Control indexing -->
  <meta name="robots" content="index, follow">
  
  <!-- Open Graph: Social sharing -->
  <meta property="og:title" content="Page Title">
  <meta property="og:description" content="Description">
  <meta property="og:image" content="https://example.com/image.jpg">
  
  <!-- Viewport: Mobile-friendliness -->
  <meta name="viewport" content="width=device-width, initial-scale=1">
</head>
```

### Structured Data (JSON-LD)

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Product Name",
  "description": "Product description",
  "price": "29.99",
  "priceCurrency": "USD",
  "availability": "https://schema.org/InStock"
}
</script>
```

Common schema types: Product, Article, FAQ, BreadcrumbList, Organization, LocalBusiness

### Semantic HTML

```html
<!-- Good: Semantic structure -->
<main>
  <article>
    <header>
      <h1>Main Title</h1>
    </header>
    <section>
      <h2>Section Title</h2>
      <p>Content...</p>
    </section>
  </article>
  <aside>Related content</aside>
</main>

<!-- Bad: Div soup -->
<div class="main">
  <div class="title">Main Title</div>
  <div class="content">Content...</div>
</div>
```

## Core Web Vitals

Google's page experience metrics that affect ranking.

### The Three Metrics

| Metric | What It Measures | Good | Needs Improvement | Poor |
|--------|------------------|------|-------------------|------|
| **LCP** (Largest Contentful Paint) | Loading performance | ≤2.5s | 2.5-4s | >4s |
| **INP** (Interaction to Next Paint) | Interactivity | ≤200ms | 200-500ms | >500ms |
| **CLS** (Cumulative Layout Shift) | Visual stability | ≤0.1 | 0.1-0.25 | >0.25 |

### Common Issues by Architecture

**SPA/CSR Issues:**
- LCP: Slow due to JS loading before content
- CLS: Content shifts as JS loads and renders

**SSR Issues:**
- INP: Hydration can block interactivity
- TTFB: Slow server response times

**SSG Issues:**
- Generally best for Core Web Vitals
- CLS: Still possible with lazy-loaded images

### Optimization Strategies

**Improve LCP:**
```html
<!-- Preload critical resources -->
<link rel="preload" href="hero-image.jpg" as="image">
<link rel="preload" href="critical.css" as="style">

<!-- Inline critical CSS -->
<style>/* Above-the-fold styles */</style>

<!-- Defer non-critical JS -->
<script defer src="app.js"></script>
```

**Improve CLS:**
```html
<!-- Reserve space for images -->
<img src="photo.jpg" width="800" height="600" alt="Description">

<!-- Reserve space for ads/embeds -->
<div style="min-height: 250px;">
  <!-- Ad will load here -->
</div>
```

**Improve INP:**
- Break up long tasks
- Minimize hydration cost
- Use `requestIdleCallback` for non-critical work

## URL Structure

### Best Practices

```
Good URLs:
/products/blue-running-shoes
/blog/2024/seo-guide
/category/electronics/phones

Poor URLs:
/products?id=12345
/page.php?cat=1&sub=2
/p/abc123xyz
```

### URL Guidelines

- Use hyphens, not underscores
- Keep URLs short and descriptive
- Include target keywords naturally
- Use lowercase only
- Avoid query parameters for indexable content

## Sitemap and Robots.txt

### XML Sitemap

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://example.com/page</loc>
    <lastmod>2024-01-15</lastmod>
    <changefreq>weekly</changefreq>
    <priority>0.8</priority>
  </url>
</urlset>
```

### Robots.txt

```
User-agent: *
Allow: /
Disallow: /admin/
Disallow: /api/
Sitemap: https://example.com/sitemap.xml
```

## SPA SEO Checklist

For SPAs that need SEO:

- [ ] Implement SSR or pre-rendering for SEO-critical pages
- [ ] Ensure each route has unique meta tags (title, description)
- [ ] Use semantic HTML structure
- [ ] Implement proper heading hierarchy (h1 → h2 → h3)
- [ ] Add structured data (JSON-LD)
- [ ] Generate XML sitemap with all routes
- [ ] Handle redirects server-side (301/302), not client-side
- [ ] Implement canonical URLs
- [ ] Ensure internal links are crawlable `<a href>` tags
- [ ] Test with Google Search Console's URL Inspection tool

## Common SEO Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Client-side only meta tags | Crawler sees defaults | SSR or head management |
| JavaScript-only navigation | Links not crawlable | Use `<a href>` tags |
| Infinite scroll | Content not discoverable | Pagination or "Load More" with URLs |
| Hash-based routing | URLs not indexed | Use History API (`/path` not `/#/path`) |
| Duplicate content | Diluted rankings | Canonical tags |
| Slow loading | Poor rankings | Optimize Core Web Vitals |

## Testing SEO

### Tools

- **Google Search Console:** Indexing status, issues
- **Lighthouse:** Core Web Vitals, SEO audit
- **View Page Source:** What crawler sees (not DevTools)
- **Google Rich Results Test:** Structured data validation

### Quick Checks

```bash
# See what Google sees
curl -A "Googlebot" https://example.com/page

# Check robots.txt
curl https://example.com/robots.txt

# View without JavaScript (approximate crawler view)
# Disable JavaScript in browser DevTools
```

---

## Deep Dive: Understanding Search Engines From First Principles

### How Googlebot Actually Works

Googlebot is not one crawler - it's a massive distributed system:

```
THE GOOGLE CRAWLING INFRASTRUCTURE:

┌──────────────────────────────────────────────────────────────┐
│                     URL FRONTIER                              │
│  (Priority queue of URLs to crawl - billions of entries)     │
│  Priority based on: PageRank, freshness, crawl budget        │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                  CRAWLER FLEET                                │
│  Thousands of servers making HTTP requests in parallel        │
│  Respects robots.txt, crawl-delay, politeness policies       │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│               HTML PROCESSING QUEUE                           │
│  Parse HTML, extract text, links, metadata                    │
│  This is FAST - pure HTML parsing                             │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│              RENDERING QUEUE (WRS)                            │
│  Web Rendering Service - headless Chrome                      │
│  Executes JavaScript for dynamic content                      │
│  This is SLOW and EXPENSIVE - limited capacity                │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│                    INDEXER                                    │
│  Processes content, builds inverted index                     │
│  Maps words → documents for fast retrieval                    │
└──────────────────────────────────────────────────────────────┘
```

**The critical insight:** JavaScript rendering is a SEPARATE phase that happens LATER, if at all.

### What Happens When Googlebot Visits Your Page

```
STEP 1: URL Discovery
- Found via: sitemap, internal links, external links, Search Console
- Added to URL Frontier with priority score

STEP 2: HTTP Request
- Googlebot requests your URL
- Sends headers: User-Agent: Googlebot, Accept-Language, etc.
- Follows redirects (301, 302, 307, 308)

STEP 3: Response Analysis
- HTTP status: 200, 404, 500, etc.
- Content-Type: text/html, application/json, etc.
- Headers: X-Robots-Tag, Canonical, etc.

STEP 4: HTML Parsing (IMMEDIATE)
GET /products/shoes HTTP/1.1
Host: example.com
User-Agent: Googlebot

Response:
<html>
<head>
  <title>Running Shoes | Example</title>
  <meta name="description" content="...">
</head>
<body>
  <div id="root">Loading...</div>
  <script src="/app.js"></script>  ← NOT EXECUTED YET
</body>
</html>

EXTRACTED IMMEDIATELY:
- Title: "Running Shoes | Example"
- Meta description
- Any visible text: "Loading..."
- Links for further crawling
- Nothing from JavaScript!

STEP 5: Rendering Queue (DELAYED)
- If page seems JS-dependent, added to rendering queue
- Could be minutes, hours, or days later
- Headless Chrome executes JavaScript
- Final DOM captured for indexing
```

### The Rendering Budget Problem

Google allocates finite resources to rendering:

```
GOOGLE'S RENDERING CONSTRAINTS:

Total pages to render: ~billions
Rendering capacity: ~millions per day (estimated)
Your site's share: depends on "crawl budget"

CRAWL BUDGET FACTORS:
1. Site authority (PageRank-like signals)
2. Update frequency (how often content changes)
3. Server response time (fast = more crawling)
4. Errors encountered (errors = less crawling)

IMPLICATIONS:
- Large sites: Not all pages get rendered
- Low-authority sites: Lower priority in queue
- Slow sites: Fewer resources allocated
- Error-prone sites: Crawl budget wasted

RECOMMENDATION:
Don't RELY on rendering - serve complete HTML
```

### Understanding Indexing vs Ranking

Many developers confuse these:

```
INDEXING: Is your page in Google's database?
- Google knows the page exists
- Has parsed its content
- Stored in the index

Check: site:example.com/your-page
If it appears: indexed
If it doesn't: not indexed (or blocked)

RANKING: Where does your page appear in results?
- Page is indexed, now competing with millions of others
- Algorithm determines position
- 200+ ranking factors

A page can be:
✓ Indexed but ranking poorly (page 10+)
✓ Indexed but not ranking for your target keywords
✗ Not indexed at all (biggest problem for SPAs)
```

### How Google Processes JavaScript SPAs

```javascript
// YOUR REACT SPA:
// Server returns:
<!DOCTYPE html>
<html>
<head>
  <title>My App</title>  <!-- Google sees this -->
</head>
<body>
  <div id="root"></div>  <!-- Google sees EMPTY DIV -->
  <script src="/bundle.js"></script>
</body>
</html>

// PHASE 1: HTML PARSING (immediate)
// Google extracts:
// - Title: "My App"
// - Body text: "" (empty)
// - Links: none found in content

// PHASE 2: RENDERING (delayed)
// Hours or days later, if ever:
// - Chrome loads page
// - Executes bundle.js
// - React renders into #root
// - Final HTML captured:

<div id="root">
  <header>...</header>
  <main>
    <h1>Welcome to My App</h1>
    <p>Content that was invisible before</p>
  </main>
</div>

// NOW Google can index the real content
// But this delay means:
// - Time-sensitive content may be stale
// - Pages might rank for "Loading..." text
// - Some pages may never get rendered
```

### The Two-Wave Indexing Phenomenon

SPAs often show strange indexing behavior:

```
WAVE 1: HTML-only indexing
- Title and meta description captured
- Body appears empty or "Loading..."
- May rank for title keywords only
- Incomplete representation in SERPs

WAVE 2: Post-render indexing (if it happens)
- Full content now visible
- Rankings may change dramatically
- Could take days or weeks

OBSERVABLE SYMPTOMS:
- Search result shows "Loading..." as snippet
- Page ranks for title but not body content
- Search Console shows "Page is not indexed" then later "Indexed"
- Rankings fluctuate as rendering catches up
```

### Core Web Vitals: The Technical Details

Understanding how metrics are measured:

```javascript
// LARGEST CONTENTFUL PAINT (LCP)
// Measures: When largest visible content renders
// Elements considered: images, videos, block-level text

// Browser tracks LCP candidates:
// t=0ms:    Navigation starts
// t=100ms:  First text paints (small heading) - LCP candidate 1
// t=500ms:  Hero image loads - LCP candidate 2 (larger, replaces)
// t=2500ms: No more updates - final LCP = 500ms ✓ GOOD

// LCP KILLERS:
// - Slow server response (TTFB)
// - Render-blocking JavaScript
// - Slow image loading
// - Client-side rendering (content waits for JS)


// INTERACTION TO NEXT PAINT (INP)
// Measures: Responsiveness to user input
// Captures: click, tap, keypress → visual update

// How it works:
// 1. User clicks button
// 2. Browser creates "click" event
// 3. Your JavaScript handler runs (event processing time)
// 4. React re-renders (presentation delay)
// 5. Browser paints the update
// 6. INP = time from click to paint complete

// INP KILLERS:
// - Long JavaScript tasks (>50ms)
// - Hydration blocking main thread
// - Heavy re-renders
// - Too many event listeners


// CUMULATIVE LAYOUT SHIFT (CLS)
// Measures: Visual stability (unexpected movement)
// Calculated: impact fraction × distance fraction

// Example of bad CLS:
// t=0ms:    Heading renders at y=0
// t=500ms:  Ad loads above heading, pushes it to y=250px
// Impact: 100% of viewport affected
// Distance: 250px / viewport height

// CLS KILLERS:
// - Images without dimensions
// - Ads/embeds without reserved space
// - Dynamically injected content
// - Web fonts causing text resize
```

### How Google Evaluates Page Quality

Beyond technical SEO, Google assesses quality:

```
E-E-A-T SIGNALS (Experience, Expertise, Authoritativeness, Trust):

EXPERIENCE:
- Does content show first-hand experience?
- Product reviews: Did you actually use it?
- Travel guides: Did you actually visit?

EXPERTISE:
- Is the author qualified to write this?
- For medical content: Is author a doctor?
- For legal content: Is author a lawyer?

AUTHORITATIVENESS:
- Is this site a known authority?
- Do other sites link to it?
- Is it cited in the industry?

TRUSTWORTHINESS:
- Secure connection (HTTPS)?
- Clear contact information?
- No deceptive practices?

HOW GOOGLE MEASURES:
- External links (authority)
- Author bios and credentials
- Site reputation
- User behavior signals
- Content accuracy (fact-checking)
```

### Canonical URLs: Preventing Duplicate Content

Duplicate content confuses Google:

```
SCENARIO: Same product at multiple URLs

/products/shoes
/products/shoes?color=red
/products/shoes?color=red&size=10
/products/shoes?utm_source=facebook

PROBLEM:
- Google sees 4 different "pages"
- Splits ranking signals across them
- May pick wrong one as "canonical"

SOLUTION: Canonical tags

<link rel="canonical" href="https://example.com/products/shoes" />

EVERY variant should point to THE ONE canonical URL.

HOW GOOGLE USES CANONICAL:
1. Sees multiple URLs with same/similar content
2. Checks for canonical tag
3. Consolidates signals to canonical URL
4. Returns canonical URL in search results

CANONICAL RULES:
- Self-referencing canonicals are GOOD (each page points to itself)
- Cross-domain canonicals work (if you have duplicate on 2 domains)
- Canonical is a HINT, not directive (Google may ignore)
- Conflicting signals = Google chooses (may be wrong)
```

### Structured Data: How Machines Understand Content

Structured data helps Google understand meaning:

```javascript
// WITHOUT STRUCTURED DATA:
// Google sees text: "Nike Air Max - $129.99 - In Stock"
// Google has to GUESS: Is this a product? What's the price?

// WITH STRUCTURED DATA:
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Product",
  "name": "Nike Air Max",
  "offers": {
    "@type": "Offer",
    "price": "129.99",
    "priceCurrency": "USD",
    "availability": "https://schema.org/InStock"
  }
}
</script>

// NOW Google KNOWS:
// - This is a Product (not an article, event, etc.)
// - The name is "Nike Air Max"
// - It costs $129.99 USD
// - It's in stock

// BENEFITS:
// - Rich snippets in search results (stars, prices, availability)
// - Product panels in shopping results
// - Voice assistant answers
// - Google Merchant Center integration
```

### The JavaScript SEO Testing Protocol

How to verify your SPA is SEO-ready:

```bash
# STEP 1: View raw HTML (what crawler sees first)
curl -s https://yoursite.com/page | head -100

# Look for:
# - Real content in HTML
# - Not just <div id="root"></div>
# - Proper <title> and <meta description>

# STEP 2: Compare rendered vs raw
# In Chrome DevTools:
# - View Page Source (raw HTML)
# - Inspect Element (rendered DOM)
# - If they're very different = SEO risk

# STEP 3: Google's Mobile-Friendly Test
# https://search.google.com/test/mobile-friendly
# Shows JavaScript-rendered version
# Reveals what Google actually sees

# STEP 4: URL Inspection Tool (Search Console)
# Shows exactly how Google indexed your page
# "View Crawled Page" shows HTML Google has
# "View Tested Page" shows rendered version

# STEP 5: Site search
site:yoursite.com/specific-page
# If it appears with wrong snippet = indexing issue
# If it doesn't appear = not indexed
```

### Real-World SEO Architecture Patterns

```
PATTERN 1: SSR/SSG FOR ALL (Safest)
- Every page server-rendered
- No JavaScript rendering dependencies
- Works for all crawlers
- Best for: content sites, e-commerce

PATTERN 2: HYBRID (Practical)
- Public pages: SSR/SSG (SEO critical)
- Authenticated pages: CSR (no SEO needed)
- Example:
  / → SSG
  /products/* → ISR
  /blog/* → SSG
  /dashboard/* → CSR (behind login)

PATTERN 3: EDGE RENDERING (Modern)
- Render at CDN edge for speed
- Still SSR, but geographically distributed
- Best for: global sites, performance critical

PATTERN 4: STREAMING SSR (Advanced)
- Stream HTML progressively
- Critical content first
- Non-critical streams later
- Best for: large pages, slow data sources

ANTI-PATTERN: CSR FOR PUBLIC CONTENT
- Hoping Google will render JavaScript
- Relying on "they support JS now"
- Will have inconsistent indexing
- May rank poorly vs competitors
```

## Related Skills

- See [web-app-architectures](../web-app-architectures/SKILL.md) for SPA vs MPA
- See [rendering-patterns](../rendering-patterns/SKILL.md) for SSR, SSG impact
- See [meta-frameworks-overview](../meta-frameworks-overview/SKILL.md) for built-in SEO features
