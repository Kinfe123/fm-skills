# Developer Experience (DX) Patterns

## Overview

Developer Experience encompasses the tools, workflows, and feedback loops that make building applications fast and enjoyable. Modern frameworks compete heavily on DX—instant hot reloading, meaningful error messages, and zero-config setups have become table stakes.

This skill covers the architecture behind great DX: how dev servers work, what makes HMR possible, why Vite changed everything, and how to build developer tools that delight.

## Prerequisites

- Understanding of [build-pipelines-bundling](../build-pipelines-bundling/SKILL.md)
- Familiarity with [meta-frameworks-overview](../meta-frameworks-overview/SKILL.md)
- Basic knowledge of module systems (ESM, CommonJS)

---

## The Evolution of Dev Servers

```
TRADITIONAL BUNDLER-BASED DEV (Webpack, 2015-2020):

Source Files ──► Full Bundle ──► Dev Server ──► Browser
     │              │                              │
     │         (slow!)                             │
     │              │                              │
     └── Change ───►│◄──── Rebuild entire bundle ──┘
                    │
              (seconds to minutes)


MODERN UNBUNDLED DEV (Vite, 2020+):

Source Files ──► Dev Server ──► Browser (native ESM)
     │              │                │
     │         (instant!)            │
     │              │                │
     └── Change ───►│◄─── Only transform changed file
                    │
              (milliseconds)
```

### Why Traditional Bundling Was Slow

```javascript
// THE COLD START PROBLEM

// Traditional dev server startup:
// 1. Scan all files in project
// 2. Parse every file to find imports
// 3. Build complete dependency graph
// 4. Transform all files (JSX, TypeScript, etc.)
// 5. Bundle everything into one or few files
// 6. Start server and serve bundle

// For a large project (10,000+ modules):
// - Startup time: 30 seconds to 2+ minutes
// - Each save: 5-30 seconds to rebuild

// THE BUNDLING TAX
// Every file change requires:
// 1. Invalidate affected modules
// 2. Re-transform changed files
// 3. Re-bundle affected chunks
// 4. Send entire chunk to browser
// 5. Browser re-executes entire chunk
```

### The Unbundled Revolution

```javascript
// VITE'S KEY INSIGHT:
// Browsers now support ES Modules natively!
// Why bundle during development at all?

// Instead of:
import { Button } from './components/Button.bundle.js';

// Browser can directly load:
import { Button } from './components/Button.tsx';
// Dev server transforms on-the-fly and serves

// BENEFITS:
// ✓ No bundling step - instant server start
// ✓ On-demand transformation - only transform what's requested
// ✓ Native browser caching - unchanged files stay cached
// ✓ True HMR - update single module, not entire chunk
```

---

## Hot Module Replacement (HMR) Deep Dive

### What is HMR?

```
WITHOUT HMR (Full Page Reload):

Code Change → Rebuild → Reload Page → Lose All State
                                           │
                                    ┌──────┴──────┐
                                    │ Form data   │
                                    │ Scroll pos  │
                                    │ Modal state │
                                    │ User input  │
                                    └─────────────┘
                                        (LOST!)


WITH HMR (Hot Update):

Code Change → Transform File → Patch Module → State Preserved!
                                     │
                              ┌──────┴──────┐
                              │ Component   │
                              │ re-renders  │
                              │ with new    │
                              │ code, state │
                              │ intact      │
                              └─────────────┘
```

### HMR Protocol

```javascript
// HMR operates through a WebSocket connection:

// 1. SERVER → CLIENT: Connection established
{ type: 'connected' }

// 2. SERVER → CLIENT: File changed, here's what to update
{
  type: 'update',
  updates: [{
    type: 'js-update',
    path: '/src/components/Button.tsx',
    acceptedPath: '/src/components/Button.tsx',
    timestamp: 1699123456789
  }]
}

// 3. CLIENT: Fetches new module
GET /src/components/Button.tsx?t=1699123456789

// 4. CLIENT: Replaces old module, calls accept callback
import.meta.hot.accept(() => {
  // Re-render with new component
});

// 5. SERVER → CLIENT: Full reload needed (can't HMR)
{ type: 'full-reload', path: '/src/main.tsx' }
```

### HMR Boundaries

```javascript
// HMR BOUNDARY: A module that "accepts" hot updates

// ✓ HMR BOUNDARY - accepts updates
// Button.tsx
export function Button({ children }) {
  return <button className="btn">{children}</button>;
}

// Vite/React plugin auto-injects:
if (import.meta.hot) {
  import.meta.hot.accept();
}


// ✗ NOT A BOUNDARY - updates bubble up
// utils.ts (no accept call)
export function formatDate(date) {
  return date.toISOString();
}


// HOW UPDATES PROPAGATE:
//
//     main.tsx (boundary)
//         │
//    ┌────┴────┐
//    │         │
// App.tsx   utils.ts ◄── Change here
//    │         │
// Button.tsx   │
//    │         │
//    └────┬────┘
//         │
//         ▼
// Update bubbles up to nearest boundary (main.tsx)
// If no boundary found → full page reload
// This means it will be bubble to those needs it. 
```

### React Fast Refresh

```javascript
// REACT FAST REFRESH: HMR that preserves React state

// How it works:
// 1. Detect which exports are React components
// 2. Wrap each component to capture its state
// 3. On update, re-render with new code, inject old state

// WHAT GETS PRESERVED:
// ✓ useState values
// ✓ useRef values  
// ✓ Component local state

// WHAT TRIGGERS FULL REMOUNT:
// ✗ Changed hooks order
// ✗ Changed hook dependencies
// ✗ Class components (legacy)
// ✗ Non-component exports mixed in


// FAST REFRESH BOUNDARIES:

// ✓ Good - only React components
// Button.tsx
export function Button() { ... }
export function IconButton() { ... }

// ✗ Bad - mixed exports force full reload
// Button.tsx
export function Button() { ... }
export const BUTTON_SIZES = ['sm', 'md', 'lg']; // Not a component!

// Solution: separate non-component exports
// constants.ts
export const BUTTON_SIZES = ['sm', 'md', 'lg'];
```

---

## Vite Architecture Deep Dive

### Core Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         VITE                                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   Connect    │    │   Module     │    │    Plugin    │  │
│  │   Server     │    │   Graph      │    │   Container  │  │
│  │              │    │              │    │              │  │
│  │ • HTTP       │    │ • Dependency │    │ • Transform  │  │
│  │ • WebSocket  │    │   tracking   │    │ • Resolve    │  │
│  │ • Middleware │    │ • HMR edges  │    │ • Load       │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│         │                   │                   │           │
│         └───────────────────┼───────────────────┘           │
│                             │                               │
│  ┌──────────────────────────┴──────────────────────────┐   │
│  │                    esbuild                            │   │
│  │  • Pre-bundle dependencies (node_modules)            │   │
│  │  • TypeScript/JSX transform (fast path)              │   │
│  └───────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                       BROWSER                                │
│  • Native ES Modules                                        │
│  • Import maps (optional)                                   │
│  • Module-level caching                                     │
└─────────────────────────────────────────────────────────────┘
```

### Dependency Pre-Bundling

```javascript
// THE PROBLEM: node_modules aren't ESM-ready

// 1. CommonJS modules (can't be loaded natively)
const lodash = require('lodash');

// 2. Many small files (hundreds of requests)
// lodash-es has 600+ individual modules!
import debounce from 'lodash-es/debounce';
// Browser would make 600+ HTTP requests

// 3. Different export formats
// Some packages: module.exports = ...
// Some packages: export default ...
// Some packages: export { named }


// VITE'S SOLUTION: Pre-bundle with esbuild

// On first run, Vite:
// 1. Scans your code for bare imports
import React from 'react';
import { debounce } from 'lodash-es';

// 2. Pre-bundles each dependency into single ESM file
// node_modules/.vite/deps/react.js
// node_modules/.vite/deps/lodash-es.js

// 3. Rewrites imports to point to pre-bundled versions
import React from '/node_modules/.vite/deps/react.js?v=abc123';

// BENEFITS:
// ✓ CommonJS → ESM conversion
// ✓ 600 files → 1 file (fewer requests)
// ✓ Cached until dependency changes
// ✓ Done with esbuild (incredibly fast)
```

### The Transform Pipeline

```javascript
// REQUEST LIFECYCLE IN VITE:

// 1. Browser requests module
GET /src/components/Button.tsx

// 2. Vite intercepts, runs through plugins
async function transformRequest(url) {
  // Resolve: determine actual file path
  const resolved = await pluginContainer.resolveId(url);
  // → /Users/you/project/src/components/Button.tsx
  
  // Load: read file contents (or virtual module)
  const loaded = await pluginContainer.load(resolved.id);
  // → "import React from 'react';\nexport function Button..."
  
  // Transform: convert to browser-ready code
  const transformed = await pluginContainer.transform(loaded.code, resolved.id);
  // → TypeScript compiled, JSX transformed, imports rewritten
  
  return transformed;
}

// 3. Serve transformed code with proper headers
// Content-Type: application/javascript
// Cache-Control: max-age=31536000 (if dependency)
// Cache-Control: no-cache (if source file)
```

### Import Rewriting

```javascript
// VITE REWRITES ALL IMPORTS:

// Your code:
import React from 'react';
import { useState } from 'react';
import Button from './Button';
import styles from './App.module.css';
import logo from './logo.svg';

// After Vite transforms:
import React from '/node_modules/.vite/deps/react.js?v=abc123';
import { useState } from '/node_modules/.vite/deps/react.js?v=abc123';
import Button from '/src/Button.tsx?t=1699123456789';
import styles from '/src/App.module.css?t=1699123456789';
import logo from '/src/logo.svg';

// WHY THE QUERY PARAMS?
// ?v=abc123  → Content hash for dependencies (long cache)
// ?t=16991.. → Timestamp for source files (bust cache on change)

// HOW IT WORKS:
function rewriteImports(code, importer) {
  // Parse with es-module-lexer (fast!)
  const [imports] = parse(code);
  
  let result = code;
  for (const imp of imports.reverse()) { // reverse to preserve positions
    const resolved = resolveImport(imp.n, importer);
    result = result.slice(0, imp.s) + resolved + result.slice(imp.e);
  }
  
  return result;
}
```

---

## Error Overlays

### The Error Overlay Experience

```
┌─────────────────────────────────────────────────────────────┐
│  ✕                                                      [x] │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  TypeError: Cannot read property 'map' of undefined         │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  12 │   return (                                        ││
│  │  13 │     <ul>                                          ││
│  │▶ 14 │       {items.map(item => (                        ││
│  │  15 │         <li key={item.id}>{item.name}</li>        ││
│  │  16 │       ))}                                         ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  src/components/List.tsx:14:15                              │
│                                                              │
│  Click outside or fix the error to dismiss                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Types of Errors

```javascript
// 1. COMPILE-TIME ERRORS (caught during transform)
// - Syntax errors
// - TypeScript errors
// - Invalid imports

// Example: SyntaxError
const x = {
  foo: 'bar'
  baz: 'qux'  // Missing comma
};

// Overlay shows:
// SyntaxError: Unexpected token (line 3)
// With syntax-highlighted code frame


// 2. RUNTIME ERRORS (caught in browser)
// - Undefined variables
// - Type errors
// - Failed assertions

// Example: TypeError
function Component({ items }) {
  return items.map(i => <div>{i}</div>); // items is undefined
}

// Overlay shows:
// TypeError: Cannot read property 'map' of undefined
// With stack trace mapped to source


// 3. HMR ERRORS (caught during hot update)
// - Failed module execution
// - React hooks violations

// Overlay shows:
// [hmr] Failed to reload /src/App.tsx
// With specific error and suggestion
```

### Source Maps for Better Errors

```javascript
// WITHOUT SOURCE MAPS:
// Error at bundle.js:15847:23
// 😱 Good luck finding that!

// WITH SOURCE MAPS:
// Error at src/components/Button.tsx:42:15
// 🎯 Exactly where you need to look!


// HOW SOURCE MAPS WORK:
//
// Original (Button.tsx):          Compiled (Button.js):
// Line 1: import React...    →    Line 1: import React...
// Line 2:                    →    Line 2: 
// Line 3: export function... →    Line 3: export function...
// Line 4:   const [x, set]   →    Line 4:   const [x, set]
//          ↑ column 8        →           ↑ column 8
//
// Source Map (Button.js.map):
// {
//   "mappings": "AAAA,OAAO,QAAQ,...",
//   "sources": ["Button.tsx"],
//   "sourcesContent": ["import React..."]
// }

// The "mappings" field encodes:
// - Output line/column → Source line/column
// - Uses VLQ encoding for compactness
```

---

## File Watching

### Chokidar: The Standard File Watcher

```javascript
// VITE USES CHOKIDAR FOR FILE WATCHING

import chokidar from 'chokidar';

const watcher = chokidar.watch('src', {
  ignored: [
    '**/node_modules/**',
    '**/.git/**',
  ],
  ignoreInitial: true,  // Don't fire for existing files
  ignorePermissionErrors: true,
});

watcher
  .on('change', (path) => {
    // File modified
    handleFileChange(path);
  })
  .on('add', (path) => {
    // New file created
    handleNewFile(path);
  })
  .on('unlink', (path) => {
    // File deleted
    handleFileDelete(path);
  });


// DEBOUNCING RAPID CHANGES:
// IDEs often save multiple times quickly

let pendingChanges = new Set();
let timeout = null;

function handleFileChange(path) {
  pendingChanges.add(path);
  
  clearTimeout(timeout);
  timeout = setTimeout(() => {
    processChanges([...pendingChanges]);
    pendingChanges.clear();
  }, 50); // Wait 50ms for more changes
}
```

### Platform-Specific Watching

```javascript
// FILE WATCHING IS OS-SPECIFIC:

// macOS: FSEvents (fast, efficient)
// - Uses kernel-level events
// - Low CPU usage
// - Handles large directories well

// Linux: inotify (good, some limits)
// - Kernel-level events
// - Watch limit: /proc/sys/fs/inotify/max_user_watches
// - May need: echo 524288 | sudo tee /proc/sys/fs/inotify/max_user_watches

// Windows: ReadDirectoryChangesW (okay, some issues)
// - Can miss rapid changes
// - Network drives problematic
// - WSL2 has cross-filesystem issues


// POLLING FALLBACK:
// When native watching fails, poll every N ms

chokidar.watch('src', {
  usePolling: true,        // Force polling
  interval: 100,           // Check every 100ms
  binaryInterval: 300,     // Less frequent for binaries
});
```

---

## Module Graph

### Tracking Dependencies

```javascript
// THE MODULE GRAPH: Heart of HMR

class ModuleGraph {
  constructor() {
    // URL → ModuleNode
    this.urlToModuleMap = new Map();
    // File path → ModuleNode
    this.fileToModulesMap = new Map();
  }
}

class ModuleNode {
  constructor(url) {
    this.url = url;
    this.file = null;
    this.importers = new Set();   // Who imports this module
    this.importedModules = new Set(); // What this module imports
    this.acceptedHmrDeps = new Set(); // HMR boundaries
    this.transformResult = null;  // Cached transform
    this.lastHMRTimestamp = 0;
  }
}

// EXAMPLE GRAPH:
//
//                    main.tsx
//                       │
//              ┌────────┴────────┐
//              ▼                 ▼
//           App.tsx          utils.ts
//              │                 │
//      ┌───────┼───────┐         │
//      ▼       ▼       ▼         │
//  Header  Content  Footer       │
//      │       │       │         │
//      └───────┴───────┴─────────┘
//              │
//              ▼
//          Button.tsx
```

### Invalidation and Propagation

```javascript
// WHEN A FILE CHANGES:

function handleFileChange(file) {
  const modules = moduleGraph.getModulesByFile(file);
  
  for (const mod of modules) {
    // 1. Invalidate this module's cache
    mod.transformResult = null;
    
    // 2. Find update boundaries
    const boundaries = propagateUpdate(mod);
    
    // 3. Send update to client
    if (boundaries.size > 0) {
      ws.send({
        type: 'update',
        updates: [...boundaries].map(b => ({
          type: 'js-update',
          path: b.url,
          timestamp: Date.now(),
        })),
      });
    } else {
      // No boundary found - full reload
      ws.send({ type: 'full-reload' });
    }
  }
}

function propagateUpdate(node, boundaries = new Set(), visited = new Set()) {
  if (visited.has(node)) return boundaries;
  visited.add(node);
  
  // Is this module an HMR boundary?
  if (node.acceptedHmrDeps.has(node)) {
    boundaries.add(node);
    return boundaries;
  }
  
  // Propagate to importers
  for (const importer of node.importers) {
    propagateUpdate(importer, boundaries, visited);
  }
  
  return boundaries;
}
```

---

## CSS Hot Update

### CSS HMR Without JavaScript

```javascript
// CSS CAN BE HOT-UPDATED WITHOUT JS EXECUTION!

// 1. Browser loads stylesheet
<link rel="stylesheet" href="/src/App.css">

// 2. File changes, Vite detects
// 3. Vite sends update message
{ type: 'css-update', path: '/src/App.css', timestamp: 123456 }

// 4. Client adds cache-busting param
const link = document.querySelector('link[href*="App.css"]');
link.href = '/src/App.css?t=123456';

// 5. Browser fetches new stylesheet
// 6. Styles update instantly - no flash!


// WHY NO FLASH?
// - Old stylesheet stays active until new one loads
// - Browser swaps atomically
// - No JavaScript re-execution needed
```

### CSS Modules HMR

```javascript
// CSS MODULES ARE TRICKIER:

// styles.module.css
.button {
  color: red;
}

// Compiles to:
export default {
  button: 'button_abc123'
};

// On HMR:
// 1. Update the stylesheet (visual change)
// 2. Update the JS module (class name mapping)
// 3. Components using new class names re-render


// VITE'S CSS MODULE TRANSFORM:
function transformCSSModule(css, id) {
  const { code, map, modules } = compileCSSModules(css);
  // since we can not run logic in css. we need to hook it with client js. 
  return `
import { updateStyle } from '/@vite/client';
const css = ${JSON.stringify(code)};
updateStyle(${JSON.stringify(id)}, css);
export default ${JSON.stringify(modules)};

if (import.meta.hot) {
  import.meta.hot.accept();
}
  `;
}
```

---

## TypeScript in Development

### Why Not tsc?

```javascript
// TYPESCRIPT COMPILER (tsc) IS SLOW:
// - Full type checking
// - Reads entire project
// - Builds complete type graph
// - Emits declarations

// FOR DEV SERVER, WE ONLY NEED:
// - Strip type annotations
// - Transform enums/decorators
// - Convert TSX

// SOLUTION: Use esbuild for transform, skip type checking

// esbuild transform (< 1ms per file):
import { transform } from 'esbuild';

const result = await transform(code, {
  loader: 'tsx',
  target: 'esnext',
});

// vs tsc (100ms+ for project):
// Full project analysis, type checking, etc.


// TYPE CHECKING SEPARATELY:
// Run tsc in a separate process, show errors in overlay

// vite.config.ts
import checker from 'vite-plugin-checker';

export default {
  plugins: [
    checker({
      typescript: true, // Runs tsc --noEmit in background
    }),
  ],
};
```

### SWC: Even Faster Transforms

```javascript
// SWC (Speedy Web Compiler) - Rust-based

// Transform benchmarks (10,000 files):
// Babel:   45 seconds
// esbuild: 0.5 seconds
// SWC:     0.3 seconds

// Vite can use SWC for React:
import react from '@vitejs/plugin-react-swc';

export default {
  plugins: [react()],
};

// SWC FEATURES:
// ✓ TypeScript transform
// ✓ JSX transform
// ✓ React Fast Refresh
// ✓ Decorators
// ✓ Minification
```

---

## Proxy and API Mocking

### Development Proxy

```javascript
// PROXY API REQUESTS TO BACKEND

// vite.config.ts
export default {
  server: {
    proxy: {
      // /api/users → http://localhost:8080/api/users
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
      
      // /graphql → http://localhost:4000/graphql
      '/graphql': {
        target: 'http://localhost:4000',
        changeOrigin: true,
      },
      
      // Rewrite paths
      '/old-api': {
        target: 'http://localhost:8080',
        rewrite: (path) => path.replace(/^\/old-api/, '/v2'),
      },
      
      // WebSocket proxy
      '/ws': {
        target: 'ws://localhost:8080',
        ws: true,
      },
    },
  },
};


// HOW PROXY WORKS:
// 1. Browser requests: http://localhost:5173/api/users
// 2. Vite intercepts, sees /api prefix
// 3. Vite forwards to: http://localhost:8080/api/users
// 4. Backend responds
// 5. Vite forwards response to browser
// 6. No CORS issues! (same origin from browser's view)
```

### Mock Service Worker (MSW)

```javascript
// MOCK APIs WITHOUT BACKEND

// mocks/handlers.ts
import { rest } from 'msw';

export const handlers = [
  rest.get('/api/users', (req, res, ctx) => {
    return res(
      ctx.json([
        { id: 1, name: 'Alice' },
        { id: 2, name: 'Bob' },
      ])
    );
  }),
  
  rest.post('/api/users', async (req, res, ctx) => {
    const body = await req.json();
    return res(
      ctx.status(201),
      ctx.json({ id: 3, ...body })
    );
  }),
  
  // Simulate errors
  rest.get('/api/error', (req, res, ctx) => {
    return res(ctx.status(500), ctx.json({ error: 'Server error' }));
  }),
  
  // Simulate latency
  rest.get('/api/slow', (req, res, ctx) => {
    return res(ctx.delay(2000), ctx.json({ data: 'slow response' }));
  }),
];

// mocks/browser.ts
import { setupWorker } from 'msw';
import { handlers } from './handlers';

export const worker = setupWorker(...handlers);

// main.tsx
if (process.env.NODE_ENV === 'development') {
  const { worker } = await import('./mocks/browser');
  await worker.start();
}
```

---

## Dev Server Middleware

### Custom Middleware in Vite

```javascript
// vite.config.ts
export default {
  plugins: [
    {
      name: 'custom-middleware',
      configureServer(server) {
        // Runs BEFORE Vite's middleware
        server.middlewares.use((req, res, next) => {
          if (req.url === '/__custom') {
            res.setHeader('Content-Type', 'application/json');
            res.end(JSON.stringify({ status: 'ok' }));
            return;
          }
          next();
        });
        
        // Runs AFTER Vite's middleware (return function)
        return () => {
          server.middlewares.use((req, res, next) => {
            // Post-middleware logic
            next();
          });
        };
      },
    },
  ],
};


// COMMON MIDDLEWARE USE CASES:

// 1. Custom API routes
configureServer(server) {
  server.middlewares.use('/api', (req, res, next) => {
    // Handle API
  });
}

// 2. Server-sent events for live data
configureServer(server) {
  server.middlewares.use('/__events', (req, res) => {
    res.writeHead(200, {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    });
    
    const interval = setInterval(() => {
      res.write(`data: ${JSON.stringify({ time: Date.now() })}\n\n`);
    }, 1000);
    
    req.on('close', () => clearInterval(interval));
  });
}

// 3. Authentication mock
configureServer(server) {
  server.middlewares.use((req, res, next) => {
    if (req.url.startsWith('/api/')) {
      req.headers['x-user-id'] = 'dev-user-123';
    }
    next();
  });
}
```

---

## Zero-Config Philosophy

### Convention Over Configuration

```javascript
// VITE'S ZERO-CONFIG DEFAULTS:

// 1. Entry point: index.html (not JS!)
// 2. Source directory: / (root)
// 3. Output directory: dist/
// 4. Dev port: 5173
// 5. Asset handling: automatic
// 6. CSS: just import it
// 7. TypeScript: works out of the box
// 8. JSX: auto-detected


// MINIMAL SETUP:
// package.json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build"
  },
  "devDependencies": {
    "vite": "^5.0.0"
  }
}

// index.html
<!DOCTYPE html>
<html>
<body>
  <div id="root"></div>
  <script type="module" src="/src/main.tsx"></script>
</body>
</html>

// That's it! No config file needed.


// FRAMEWORK PRESETS:
npm create vite@latest my-app -- --template react-ts
npm create vite@latest my-app -- --template vue
npm create vite@latest my-app -- --template svelte
```

### Sensible Defaults with Escape Hatches

```javascript
// DEFAULTS CAN BE OVERRIDDEN:

// vite.config.ts
export default {
  // Change root
  root: 'src',
  
  // Change output
  build: {
    outDir: '../build',
  },
  
  // Change public dir
  publicDir: 'static',
  
  // Custom port
  server: {
    port: 3000,
    open: true, // Auto-open browser
  },
  
  // Alias paths
  resolve: {
    alias: {
      '@': '/src',
      '@components': '/src/components',
    },
  },
};
```

---

## Beyond Vite: Other DX Tools

### Turbopack (Next.js)

```javascript
// TURBOPACK: Incremental bundler by Vercel

// Key differences from Vite:
// 1. Bundler-based (like Webpack) but incremental
// 2. Rust-based (like SWC)
// 3. Function-level caching
// 4. Designed for Next.js specifically

// Turbo's approach:
// - Don't re-transform unchanged functions
// - Cache at granular level
// - Parallelize everything

// Enable in Next.js:
next dev --turbo
```

### Bun as Dev Server

```javascript
// BUN: All-in-one JavaScript runtime

// Bun is FAST:
// - Written in Zig
// - Uses JavaScriptCore (not V8)
// - Native bundler/transpiler
// - Native test runner

// Bun dev server:
bun --hot src/index.tsx

// Bun's advantages:
// ✓ Single binary (no npm install)
// ✓ Faster npm install
// ✓ Native TypeScript
// ✓ Built-in test runner
// ✓ Built-in bundler

// Bun's tradeoffs:
// ✗ Less mature ecosystem
// ✗ Some Node.js incompatibilities
// ✗ Smaller plugin ecosystem
```

### Modern Alternatives Comparison

```
┌────────────┬──────────┬────────────┬─────────────┬─────────┐
│            │ Vite     │ Turbopack  │ Bun         │ Parcel  │
├────────────┼──────────┼────────────┼─────────────┼─────────┤
│ Language   │ TS/JS    │ Rust       │ Zig         │ Rust/JS │
├────────────┼──────────┼────────────┼─────────────┼─────────┤
│ Approach   │ Unbundled│ Incremental│ All-in-one  │ Zero-cfg│
├────────────┼──────────┼────────────┼─────────────┼─────────┤
│ Dev Mode   │ ESM      │ Bundled    │ ESM         │ ESM     │
├────────────┼──────────┼────────────┼─────────────┼─────────┤
│ Framework  │ Agnostic │ Next.js    │ Agnostic    │ Agnostic│
├────────────┼──────────┼────────────┼─────────────┼─────────┤
│ Plugins    │ Rollup   │ Webpack*   │ Limited     │ Own     │
├────────────┼──────────┼────────────┼─────────────┼─────────┤
│ Maturity   │ Stable   │ Beta       │ Stable      │ Stable  │
└────────────┴──────────┴────────────┴─────────────┴─────────┘

* Turbopack aims for Webpack plugin compatibility
```

---

## Deep Dive: Building a Minimal Dev Server

### Core Implementation

```javascript
// A SIMPLIFIED DEV SERVER (educational)

import { createServer } from 'node:http';
import { readFile } from 'node:fs/promises';
import { WebSocketServer } from 'ws';
import chokidar from 'chokidar';
import { transform } from 'esbuild';

class DevServer {
  constructor(root = process.cwd()) {
    this.root = root;
    this.moduleGraph = new Map();
    this.clients = new Set();
  }
  
  async start(port = 3000) {
    // HTTP server for files
    this.httpServer = createServer(this.handleRequest.bind(this));
    
    // WebSocket for HMR
    this.wss = new WebSocketServer({ server: this.httpServer });
    this.wss.on('connection', (ws) => {
      this.clients.add(ws);
      ws.on('close', () => this.clients.delete(ws));
    });
    
    // File watcher
    this.watcher = chokidar.watch(this.root, {
      ignored: /node_modules/,
      ignoreInitial: true,
    });
    this.watcher.on('change', this.handleFileChange.bind(this));
    
    this.httpServer.listen(port, () => {
      console.log(`Dev server running at http://localhost:${port}`);
    });
  }
  
  async handleRequest(req, res) {
    let url = req.url;
    
    // Serve index.html for root
    if (url === '/') {
      url = '/index.html';
    }
    
    try {
      if (url.endsWith('.html')) {
        await this.serveHtml(url, res);
      } else if (url.endsWith('.ts') || url.endsWith('.tsx')) {
        await this.serveTypeScript(url, res);
      } else if (url.endsWith('.css')) {
        await this.serveCss(url, res);
      } else {
        await this.serveStatic(url, res);
      }
    } catch (error) {
      res.writeHead(500);
      res.end(error.message);
    }
  }
  
  async serveHtml(url, res) {
    let html = await readFile(this.root + url, 'utf-8');
    
    // Inject HMR client
    html = html.replace(
      '</head>',
      `<script type="module" src="/@hmr-client"></script></head>`
    );
    
    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(html);
  }
  
  async serveTypeScript(url, res) {
    const filePath = this.root + url.split('?')[0];
    const code = await readFile(filePath, 'utf-8');
    
    // Transform with esbuild
    const result = await transform(code, {
      loader: url.endsWith('.tsx') ? 'tsx' : 'ts',
      sourcemap: 'inline',
      target: 'esnext',
    });
    
    // Rewrite imports and inject HMR
    const transformed = this.transformImports(result.code, url);
    
    // Track in module graph
    this.moduleGraph.set(url, {
      code: transformed,
      timestamp: Date.now(),
    });
    
    res.writeHead(200, { 'Content-Type': 'application/javascript' });
    res.end(transformed);
  }
  
  transformImports(code, importer) {
    // Simple import rewriting (real impl uses es-module-lexer)
    code = code.replace(
      /from ['"](.+)['"]/g,
      (match, path) => {
        if (path.startsWith('.')) {
          return `from '${path}?t=${Date.now()}'`;
        }
        return match;
      }
    );
    
    // Inject HMR
    code += `
if (import.meta.hot) {
  import.meta.hot.accept();
}`;
    
    return code;
  }
  
  handleFileChange(file) {
    const url = file.replace(this.root, '');
    
    // Invalidate cache
    this.moduleGraph.delete(url);
    
    // Notify clients
    this.broadcast({
      type: 'update',
      path: url,
      timestamp: Date.now(),
    });
  }
  
  broadcast(message) {
    const data = JSON.stringify(message);
    for (const client of this.clients) {
      client.send(data);
    }
  }
}

// HMR Client (served at /@hmr-client)
const hmrClient = `
const socket = new WebSocket('ws://' + location.host);

socket.onmessage = async (event) => {
  const data = JSON.parse(event.data);
  
  if (data.type === 'update') {
    // Fetch new module
    const newModule = await import(data.path + '?t=' + data.timestamp);
    console.log('[HMR] Updated:', data.path);
  }
};

// Expose to modules
import.meta.hot = {
  accept(cb) {
    // Register callback
  },
};
`;
```

---

## For Framework Authors: Building DX Systems

> **Implementation Note**: The patterns and code examples below represent one proven approach to building developer experience systems. Different tools take different approaches—Vite uses native ESM, Turbopack uses incremental compilation, and Bun uses a unified runtime. The direction shown here is based on the unbundled dev server pattern popularized by Vite. Adapt based on your framework's bundler integration, target audience, and whether you need custom transform pipelines.

### Implementing an HMR Runtime

```javascript
// CLIENT-SIDE HMR RUNTIME

class HMRRuntime {
  constructor() {
    this.socket = null;
    this.registeredModules = new Map();
    this.pendingUpdates = [];
    this.isConnected = false;
  }
  
  connect(url) {
    this.socket = new WebSocket(url);
    
    this.socket.onopen = () => {
      this.isConnected = true;
      console.log('[HMR] Connected');
      this.flushPendingUpdates();
    };
    
    this.socket.onmessage = (event) => {
      const message = JSON.parse(event.data);
      this.handleMessage(message);
    };
    
    this.socket.onclose = () => {
      this.isConnected = false;
      console.log('[HMR] Disconnected, attempting reconnect...');
      setTimeout(() => this.connect(url), 1000);
    };
  }
  
  handleMessage(message) {
    switch (message.type) {
      case 'connected':
        console.log('[HMR] Server connected');
        break;
        
      case 'update':
        this.handleUpdate(message.updates);
        break;
        
      case 'full-reload':
        console.log('[HMR] Full reload required');
        location.reload();
        break;
        
      case 'error':
        this.showErrorOverlay(message.error);
        break;
        
      case 'prune':
        // Remove stale modules
        for (const path of message.paths) {
          this.registeredModules.delete(path);
        }
        break;
    }
  }
  
  async handleUpdate(updates) {
    const modulesToUpdate = [];
    
    for (const update of updates) {
      if (update.type === 'js-update') {
        modulesToUpdate.push(update);
      } else if (update.type === 'css-update') {
        this.updateCSS(update.path, update.timestamp);
      }
    }
    
    // Batch JavaScript updates
    if (modulesToUpdate.length > 0) {
      await this.applyJSUpdates(modulesToUpdate);
    }
  }
  
  async applyJSUpdates(updates) {
    const toApply = [];
    
    for (const { path, timestamp, acceptedPath } of updates) {
      const mod = this.registeredModules.get(acceptedPath);
      
      if (!mod) {
        // Module not registered, needs full reload
        console.warn(`[HMR] ${path} is not HMR-compatible, reloading`);
        location.reload();
        return;
      }
      
      toApply.push({ mod, path, timestamp });
    }
    
    // Apply updates
    for (const { mod, path, timestamp } of toApply) {
      try {
        // Dispose old module if needed
        if (mod.disposeCallback) {
          await mod.disposeCallback();
        }
        
        // Fetch new module
        const newUrl = `${path}?t=${timestamp}`;
        const newModule = await import(/* @vite-ignore */ newUrl);
        
        // Call accept callback
        if (mod.acceptCallback) {
          await mod.acceptCallback(newModule);
        }
        
        console.log(`[HMR] Updated ${path}`);
      } catch (error) {
        console.error(`[HMR] Failed to apply update for ${path}`, error);
        this.showErrorOverlay({
          message: error.message,
          stack: error.stack,
          file: path,
        });
      }
    }
  }
  
  updateCSS(path, timestamp) {
    // Find existing link
    const links = document.querySelectorAll('link[rel="stylesheet"]');
    for (const link of links) {
      if (link.href.includes(path)) {
        // Update href to bust cache
        const url = new URL(link.href);
        url.searchParams.set('t', timestamp);
        link.href = url.toString();
        console.log(`[HMR] Updated CSS ${path}`);
        return;
      }
    }
    
    // Link not found, insert new one
    const newLink = document.createElement('link');
    newLink.rel = 'stylesheet';
    newLink.href = `${path}?t=${timestamp}`;
    document.head.appendChild(newLink);
  }
  
  // Called by modules via import.meta.hot.accept()
  registerModule(path, callbacks) {
    this.registeredModules.set(path, callbacks);
  }
  
  // Create import.meta.hot API for a module
  createHotContext(ownerPath) {
    const hot = {
      accept: (deps, callback) => {
        if (typeof deps === 'function' || !deps) {
          // Self-accepting
          this.registerModule(ownerPath, {
            acceptCallback: deps || (() => {}),
            selfAccepting: true,
          });
        } else {
          // Accepting specific deps
          const callbacks = Array.isArray(deps) ? deps : [deps];
          // Implementation for dep-accepting...
        }
      },
      
      dispose: (callback) => {
        const mod = this.registeredModules.get(ownerPath);
        if (mod) {
          mod.disposeCallback = callback;
        }
      },
      
      invalidate: () => {
        // Force parent to update
        this.socket?.send(JSON.stringify({
          type: 'invalidate',
          path: ownerPath,
        }));
      },
      
      // Access to current module's data
      data: {},
    };
    
    return hot;
  }
}

// Global runtime
const hmr = new HMRRuntime();
hmr.connect(`ws://${location.host}/__hmr`);

// Export for modules
export { hmr };
```

### Building an Error Overlay

```javascript
// ERROR OVERLAY COMPONENT

class ErrorOverlay extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
  }
  
  static get styles() {
    return `
      :host {
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        z-index: 99999;
        background: rgba(0, 0, 0, 0.85);
        font-family: 'SF Mono', Monaco, Consolas, monospace;
      }
      
      .container {
        max-width: 800px;
        margin: 50px auto;
        padding: 30px;
        background: #1a1a1a;
        border-radius: 8px;
        border: 1px solid #333;
      }
      
      .header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 20px;
      }
      
      .title {
        color: #ff5555;
        font-size: 18px;
        font-weight: bold;
      }
      
      .close {
        background: none;
        border: none;
        color: #888;
        cursor: pointer;
        font-size: 24px;
      }
      
      .message {
        color: #ff8888;
        font-size: 16px;
        margin-bottom: 20px;
        white-space: pre-wrap;
      }
      
      .file {
        color: #888;
        margin-bottom: 15px;
      }
      
      .file a {
        color: #4dabf7;
        text-decoration: none;
      }
      
      .code-frame {
        background: #0d0d0d;
        border-radius: 4px;
        padding: 15px;
        overflow-x: auto;
        margin-bottom: 20px;
      }
      
      .line {
        display: flex;
        line-height: 1.6;
      }
      
      .line-number {
        color: #666;
        width: 50px;
        flex-shrink: 0;
        user-select: none;
      }
      
      .line-content {
        color: #ccc;
        white-space: pre;
      }
      
      .line.highlight {
        background: rgba(255, 85, 85, 0.15);
      }
      
      .line.highlight .line-content {
        color: #ff8888;
      }
      
      .stack {
        color: #888;
        font-size: 13px;
        line-height: 1.8;
      }
      
      .stack-frame {
        color: #ccc;
      }
      
      .stack-file {
        color: #4dabf7;
      }
    `;
  }
  
  show(error) {
    const { message, file, line, column, frame, stack } = error;
    
    this.shadowRoot.innerHTML = `
      <style>${ErrorOverlay.styles}</style>
      <div class="container">
        <div class="header">
          <span class="title">${this.escapeHtml(error.type || 'Error')}</span>
          <button class="close" onclick="this.getRootNode().host.close()">×</button>
        </div>
        
        <div class="message">${this.escapeHtml(message)}</div>
        
        ${file ? `
          <div class="file">
            <a href="vscode://file${file}:${line}:${column}">
              ${file}:${line}:${column}
            </a>
          </div>
        ` : ''}
        
        ${frame ? `
          <div class="code-frame">
            ${this.renderCodeFrame(frame, line)}
          </div>
        ` : ''}
        
        ${stack ? `
          <div class="stack">
            ${this.renderStack(stack)}
          </div>
        ` : ''}
      </div>
    `;
    
    document.body.appendChild(this);
    
    // Close on escape
    this.handleKeydown = (e) => {
      if (e.key === 'Escape') this.close();
    };
    document.addEventListener('keydown', this.handleKeydown);
    
    // Close on click outside
    this.addEventListener('click', (e) => {
      if (e.target === this) this.close();
    });
  }
  
  renderCodeFrame(frame, errorLine) {
    const lines = frame.split('\n');
    return lines.map((content, i) => {
      const lineNum = errorLine - Math.floor(lines.length / 2) + i;
      const isError = lineNum === errorLine;
      return `
        <div class="line ${isError ? 'highlight' : ''}">
          <span class="line-number">${lineNum}</span>
          <span class="line-content">${this.highlightSyntax(content)}</span>
        </div>
      `;
    }).join('');
  }
  
  renderStack(stack) {
    return stack
      .split('\n')
      .filter(line => line.trim())
      .map(line => {
        // Parse stack frame
        const match = line.match(/at (.+) \((.+):(\d+):(\d+)\)/);
        if (match) {
          const [, fn, file, line, col] = match;
          return `
            <div class="stack-frame">
              at <span class="stack-fn">${fn}</span>
              (<a class="stack-file" href="vscode://file${file}:${line}:${col}">${file}:${line}:${col}</a>)
            </div>
          `;
        }
        return `<div>${this.escapeHtml(line)}</div>`;
      })
      .join('');
  }
  
  highlightSyntax(code) {
    // Simple syntax highlighting
    return this.escapeHtml(code)
      .replace(/\b(const|let|var|function|return|if|else|for|while|import|export|from|class|extends)\b/g, 
        '<span style="color:#c678dd">$1</span>')
      .replace(/(['"`]).*?\1/g, 
        '<span style="color:#98c379">$&</span>')
      .replace(/\b(\d+)\b/g, 
        '<span style="color:#d19a66">$1</span>');
  }
  
  escapeHtml(str) {
    return str
      .replace(/&/g, '&amp;')
      .replace(/</g, '&lt;')
      .replace(/>/g, '&gt;')
      .replace(/"/g, '&quot;');
  }
  
  close() {
    document.removeEventListener('keydown', this.handleKeydown);
    this.remove();
  }
}

customElements.define('error-overlay', ErrorOverlay);

// Usage
function showError(error) {
  // Remove existing overlay
  document.querySelector('error-overlay')?.remove();
  
  const overlay = new ErrorOverlay();
  overlay.show(error);
}
```

### Implementing a Transform Pipeline

```javascript
// PLUGGABLE TRANSFORM PIPELINE

class TransformPipeline {
  constructor() {
    this.transformers = [];
    this.cache = new Map();
  }
  
  // Register a transformer
  use(transformer) {
    this.transformers.push(transformer);
    // Sort by enforce order
    this.transformers.sort((a, b) => {
      const order = { pre: -1, normal: 0, post: 1 };
      return (order[a.enforce] || 0) - (order[b.enforce] || 0);
    });
    return this;
  }
  
  // Transform a module
  async transform(code, id, options = {}) {
    // Check cache
    const cacheKey = `${id}:${code.length}:${code.slice(0, 100)}`;
    if (this.cache.has(cacheKey) && !options.force) {
      return this.cache.get(cacheKey);
    }
    
    let result = { code, map: null };
    
    for (const transformer of this.transformers) {
      // Check if transformer handles this file
      if (transformer.filter && !transformer.filter(id)) {
        continue;
      }
      
      try {
        const output = await transformer.transform(result.code, id);
        
        if (output) {
          if (typeof output === 'string') {
            result.code = output;
          } else {
            result.code = output.code;
            result.map = this.combineSourceMaps(result.map, output.map);
          }
        }
      } catch (error) {
        throw new TransformError(error.message, {
          id,
          transformer: transformer.name,
          originalError: error,
        });
      }
    }
    
    this.cache.set(cacheKey, result);
    return result;
  }
  
  // Invalidate cache for a file
  invalidate(id) {
    for (const key of this.cache.keys()) {
      if (key.startsWith(id)) {
        this.cache.delete(key);
      }
    }
  }
  
  combineSourceMaps(map1, map2) {
    if (!map1) return map2;
    if (!map2) return map1;
    // Use a library like @ampproject/remapping
    // to properly combine source maps
    return map2;
  }
}

// Built-in transformers

// TypeScript/JSX transformer
const esbuildTransformer = {
  name: 'esbuild',
  enforce: 'pre',
  filter: (id) => /\.[jt]sx?$/.test(id),
  
  async transform(code, id) {
    const { transform } = await import('esbuild');
    
    const loader = id.endsWith('.tsx') ? 'tsx' :
                   id.endsWith('.ts') ? 'ts' :
                   id.endsWith('.jsx') ? 'jsx' : 'js';
    
    return transform(code, {
      loader,
      sourcemap: true,
      sourcefile: id,
      target: 'esnext',
    });
  },
};

// React Fast Refresh transformer
const reactRefreshTransformer = {
  name: 'react-refresh',
  enforce: 'post',
  filter: (id) => /\.[jt]sx$/.test(id) && !id.includes('node_modules'),
  
  async transform(code, id) {
    // Check if file exports React components
    if (!hasReactComponents(code)) {
      return null;
    }
    
    // Inject refresh runtime
    const header = `
import RefreshRuntime from '/@react-refresh';
const prevRefreshReg = window.$RefreshReg$;
const prevRefreshSig = window.$RefreshSig$;
window.$RefreshReg$ = (type, id) => RefreshRuntime.register(type, ${JSON.stringify(id)} + ' ' + id);
window.$RefreshSig$ = RefreshRuntime.createSignatureFunctionForTransform;
`;
    
    const footer = `
window.$RefreshReg$ = prevRefreshReg;
window.$RefreshSig$ = prevRefreshSig;
import.meta.hot?.accept();
RefreshRuntime.performReactRefresh();
`;
    
    return {
      code: header + code + footer,
      map: null,
    };
  },
};

function hasReactComponents(code) {
  // Simple heuristic - check for function components or JSX
  return /\bfunction\s+[A-Z]/.test(code) || 
         /<[A-Z]/.test(code) ||
         /export\s+(default\s+)?function\s+[A-Z]/.test(code);
}

// Import rewriter
const importRewriter = {
  name: 'import-rewriter',
  enforce: 'post',
  
  async transform(code, id) {
    const { parse } = await import('es-module-lexer');
    const [imports] = await parse(code);
    
    let result = code;
    
    // Process imports in reverse order to preserve positions
    for (const imp of imports.reverse()) {
      const specifier = code.slice(imp.s, imp.e);
      
      // Rewrite bare imports to pre-bundled deps
      if (!specifier.startsWith('.') && 
          !specifier.startsWith('/') && 
          !specifier.startsWith('http')) {
        const resolved = `/node_modules/.dev/${specifier}.js`;
        result = result.slice(0, imp.s) + resolved + result.slice(imp.e);
      }
      
      // Add timestamp to local imports
      if (specifier.startsWith('.') || specifier.startsWith('/')) {
        const timestamped = `${specifier}?t=${Date.now()}`;
        result = result.slice(0, imp.s) + timestamped + result.slice(imp.e);
      }
    }
    
    return result;
  },
};

// Usage
const pipeline = new TransformPipeline();
pipeline.use(esbuildTransformer);
pipeline.use(reactRefreshTransformer);
pipeline.use(importRewriter);

const result = await pipeline.transform(sourceCode, '/src/App.tsx');
```

### Building a Dev Server with HMR

```javascript
// FULL DEV SERVER IMPLEMENTATION

import { createServer } from 'node:http';
import { WebSocketServer } from 'ws';
import { readFile, stat } from 'node:fs/promises';
import { join, extname } from 'node:path';
import chokidar from 'chokidar';

class DevServer {
  constructor(options = {}) {
    this.root = options.root || process.cwd();
    this.port = options.port || 5173;
    this.plugins = options.plugins || [];
    
    this.moduleGraph = new ModuleGraph();
    this.transformPipeline = new TransformPipeline();
    this.clients = new Set();
    
    // Setup plugins
    for (const plugin of this.plugins) {
      if (plugin.transform) {
        this.transformPipeline.use(plugin);
      }
    }
  }
  
  async start() {
    // Initialize plugins
    for (const plugin of this.plugins) {
      if (plugin.buildStart) {
        await plugin.buildStart();
      }
    }
    
    // Create HTTP server
    this.server = createServer(this.handleRequest.bind(this));
    
    // Setup WebSocket for HMR
    this.wss = new WebSocketServer({ noServer: true });
    this.server.on('upgrade', (req, socket, head) => {
      if (req.url === '/__hmr') {
        this.wss.handleUpgrade(req, socket, head, (ws) => {
          this.wss.emit('connection', ws, req);
        });
      }
    });
    
    this.wss.on('connection', (ws) => {
      this.clients.add(ws);
      ws.send(JSON.stringify({ type: 'connected' }));
      ws.on('close', () => this.clients.delete(ws));
    });
    
    // Setup file watcher
    this.watcher = chokidar.watch(this.root, {
      ignored: [/node_modules/, /\.git/],
      ignoreInitial: true,
    });
    
    this.watcher.on('change', this.handleFileChange.bind(this));
    this.watcher.on('add', this.handleFileAdd.bind(this));
    this.watcher.on('unlink', this.handleFileDelete.bind(this));
    
    // Start listening
    return new Promise((resolve) => {
      this.server.listen(this.port, () => {
        console.log(`\n  Dev server running at:\n`);
        console.log(`  > Local:   http://localhost:${this.port}/\n`);
        resolve(this);
      });
    });
  }
  
  async handleRequest(req, res) {
    const url = new URL(req.url, `http://localhost:${this.port}`);
    let pathname = url.pathname;
    
    // Let plugins handle custom routes
    for (const plugin of this.plugins) {
      if (plugin.configureServer) {
        const result = await plugin.configureServer(req, res);
        if (result === false) return; // Plugin handled it
      }
    }
    
    try {
      // Serve HMR client
      if (pathname === '/@hmr-client') {
        return this.serveHMRClient(res);
      }
      
      // Serve react-refresh runtime
      if (pathname === '/@react-refresh') {
        return this.serveReactRefresh(res);
      }
      
      // Serve index.html for SPA
      if (pathname === '/' || !pathname.includes('.')) {
        pathname = '/index.html';
      }
      
      // Determine file path
      const filePath = join(this.root, pathname);
      
      // Check if file exists
      try {
        await stat(filePath);
      } catch {
        res.writeHead(404);
        res.end('Not found');
        return;
      }
      
      // Serve based on file type
      const ext = extname(filePath);
      
      if (ext === '.html') {
        await this.serveHTML(filePath, res);
      } else if (['.ts', '.tsx', '.js', '.jsx'].includes(ext)) {
        await this.serveJS(filePath, pathname, res);
      } else if (ext === '.css') {
        await this.serveCSS(filePath, pathname, res);
      } else {
        await this.serveStatic(filePath, res);
      }
      
    } catch (error) {
      console.error('Request error:', error);
      res.writeHead(500);
      res.end(JSON.stringify({
        type: 'error',
        message: error.message,
        stack: error.stack,
      }));
    }
  }
  
  async serveHTML(filePath, res) {
    let html = await readFile(filePath, 'utf-8');
    
    // Inject HMR client before </head>
    html = html.replace(
      '</head>',
      `<script type="module" src="/@hmr-client"></script></head>`
    );
    
    res.writeHead(200, { 'Content-Type': 'text/html' });
    res.end(html);
  }
  
  async serveJS(filePath, pathname, res) {
    const code = await readFile(filePath, 'utf-8');
    
    // Transform
    const result = await this.transformPipeline.transform(code, filePath);
    
    // Update module graph
    this.moduleGraph.ensureEntry(pathname, filePath);
    
    res.writeHead(200, {
      'Content-Type': 'application/javascript',
      'Cache-Control': 'no-cache',
    });
    res.end(result.code);
  }
  
  async serveCSS(filePath, pathname, res) {
    const css = await readFile(filePath, 'utf-8');
    
    // For CSS modules, transform to JS
    if (pathname.includes('.module.css')) {
      const js = this.transformCSSModule(css, pathname);
      res.writeHead(200, { 'Content-Type': 'application/javascript' });
      res.end(js);
      return;
    }
    
    res.writeHead(200, { 'Content-Type': 'text/css' });
    res.end(css);
  }
  
  transformCSSModule(css, id) {
    // Simple CSS Modules implementation
    const classMap = {};
    const transformed = css.replace(/\.([a-zA-Z_][\w-]*)/g, (match, name) => {
      const hash = this.hash(id + name).slice(0, 8);
      classMap[name] = `${name}_${hash}`;
      return `.${classMap[name]}`;
    });
    
    return `
const css = ${JSON.stringify(transformed)};
const style = document.createElement('style');
style.textContent = css;
document.head.appendChild(style);
export default ${JSON.stringify(classMap)};
if (import.meta.hot) {
  import.meta.hot.accept();
}
    `;
  }
  
  async handleFileChange(filePath) {
    const pathname = filePath.replace(this.root, '');
    console.log(`[HMR] File changed: ${pathname}`);
    
    // Invalidate transform cache
    this.transformPipeline.invalidate(filePath);
    
    // Find affected modules
    const modules = this.moduleGraph.getModulesByFile(filePath);
    
    if (modules.length === 0) {
      return; // Not a tracked module
    }
    
    // Find HMR boundaries
    const boundaries = this.moduleGraph.findBoundaries(modules);
    
    if (boundaries.length > 0) {
      // Can do HMR
      this.broadcast({
        type: 'update',
        updates: boundaries.map(mod => ({
          type: pathname.endsWith('.css') ? 'css-update' : 'js-update',
          path: mod.url,
          timestamp: Date.now(),
        })),
      });
    } else {
      // Need full reload
      this.broadcast({ type: 'full-reload' });
    }
  }
  
  broadcast(message) {
    const data = JSON.stringify(message);
    for (const client of this.clients) {
      if (client.readyState === 1) { // OPEN
        client.send(data);
      }
    }
  }
  
  hash(str) {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash |= 0;
    }
    return Math.abs(hash).toString(36);
  }
  
  serveHMRClient(res) {
    res.writeHead(200, { 'Content-Type': 'application/javascript' });
    res.end(`
// HMR Client Runtime
${HMR_CLIENT_CODE}
    `);
  }
  
  async close() {
    this.watcher.close();
    this.wss.close();
    this.server.close();
    
    for (const plugin of this.plugins) {
      if (plugin.buildEnd) {
        await plugin.buildEnd();
      }
    }
  }
}

// Module Graph for HMR
class ModuleGraph {
  constructor() {
    this.urlToModule = new Map();
    this.fileToModules = new Map();
  }
  
  ensureEntry(url, file) {
    if (!this.urlToModule.has(url)) {
      const mod = {
        url,
        file,
        importers: new Set(),
        importedModules: new Set(),
        acceptsSelf: false,
      };
      this.urlToModule.set(url, mod);
      
      if (!this.fileToModules.has(file)) {
        this.fileToModules.set(file, new Set());
      }
      this.fileToModules.get(file).add(mod);
    }
    return this.urlToModule.get(url);
  }
  
  getModulesByFile(file) {
    return [...(this.fileToModules.get(file) || [])];
  }
  
  findBoundaries(modules) {
    const boundaries = [];
    const visited = new Set();
    
    const traverse = (mod) => {
      if (visited.has(mod)) return;
      visited.add(mod);
      
      if (mod.acceptsSelf) {
        boundaries.push(mod);
        return;
      }
      
      for (const importer of mod.importers) {
        traverse(importer);
      }
    };
    
    for (const mod of modules) {
      traverse(mod);
    }
    
    return boundaries;
  }
}

// Usage
const server = new DevServer({
  root: process.cwd(),
  port: 5173,
  plugins: [
    esbuildTransformer,
    reactRefreshTransformer,
  ],
});

await server.start();
```

## Related Skills

- See [build-pipelines-bundling](../build-pipelines-bundling/SKILL.md) for bundler architecture
- See [meta-frameworks-overview](../meta-frameworks-overview/SKILL.md) for framework integration
- See [universal-javascript-runtimes](../universal-javascript-runtimes/SKILL.md) for server architecture
