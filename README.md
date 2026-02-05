# Framework Skills

A collection of framework-agnostic AI skills for understanding modern web development fundamentals.

## Purpose

These skills help AI assistants understand:
- How web applications are architected (SPA, MPA, hybrid)
- Build pipelines, bundling, code splitting, and optimization
- Different rendering strategies and when to use each
- SEO implications of architectural decisions
- Universal JavaScript and cross-platform deployment (Nitro, H3, edge runtimes)
- Middleware patterns for request/response processing (auth, CORS, rate limiting)
- How meta-frameworks solve common problems
- Hydration, routing, and other core concepts

Rather than framework-specific instructions, these skills provide the **conceptual foundation** needed to make informed decisions and understand any meta-framework.

## Skills Overview

| Skill | Description |
|-------|-------------|
| [web-app-architectures](./web-app-architectures/SKILL.md) | SPA vs MPA fundamentals, hybrid architectures |
| [rendering-patterns](./rendering-patterns/SKILL.md) | CSR, SSR, SSG, ISR, Streaming |
| [seo-fundamentals](./seo-fundamentals/SKILL.md) | SEO for web apps, Core Web Vitals, crawling |
| [hydration-patterns](./hydration-patterns/SKILL.md) | Hydration, islands architecture, resumability |
| [meta-frameworks-overview](./meta-frameworks-overview/SKILL.md) | Next.js, Nuxt, SvelteKit, Astro, Remix, Qwik |
| [routing-patterns](./routing-patterns/SKILL.md) | Client vs server routing, file-based routing |
| [state-management-patterns](./state-management-patterns/SKILL.md) | Client state, server state, URL state, caching |
| [data-fetching-patterns](./data-fetching-patterns/SKILL.md) | Fetch patterns, caching, loading states |
| [build-pipelines-bundling](./build-pipelines-bundling/SKILL.md) | Bundling, code splitting, tree shaking, build optimization |
| [universal-javascript-runtimes](./universal-javascript-runtimes/SKILL.md) | Nitro, H3, unenv, web standards, cross-platform deployment |
| [middleware-patterns](./middleware-patterns/SKILL.md) | Server/edge middleware, request pipelines, auth, CORS, rate limiting |

## Learning Path

For best understanding, read the skills in this order:

```
1. web-app-architectures       (Foundation: SPA vs MPA)
        ↓
2. build-pipelines-bundling    (How code is transformed and bundled)
        ↓
3. rendering-patterns          (When/where HTML is generated)
        ↓
4. hydration-patterns          (How static becomes interactive)
        ↓
5. routing-patterns            (How navigation works)
        ↓
6. data-fetching-patterns      (How to load data)
        ↓
7. state-management-patterns   (Where to store data)
        ↓
8. seo-fundamentals              (Search engine optimization)
        ↓
9. universal-javascript-runtimes (Deploy anywhere: edge, serverless, Node)
        ↓
10. middleware-patterns          (Request/response pipelines, auth, CORS)
        ↓
11. meta-frameworks-overview     (How frameworks implement all of the above)
```

## Installation

### For Cursor IDE

Copy to your personal skills directory:

```bash
# Personal skills (available in all projects)
cp -r fm-skills ~/.cursor/skills/

# Or symlink
ln -s /path/to/fm-skills ~/.cursor/skills/fm-skills
```

### For Project-Specific Use

Copy to your project:

```bash
cp -r fm-skills .cursor/skills/
```

## Usage

Once installed, Cursor will automatically suggest these skills when relevant topics are discussed. For example:

- Asking "Should I use SSR or SSG?" will trigger rendering-patterns
- Asking "How do SPAs handle routing?" will trigger web-app-architectures and routing-patterns
- Asking "How do I improve SEO for my React app?" will trigger seo-fundamentals

## Skill Structure

Each skill follows the standard format:

```
skill-name/
└── SKILL.md          # Main content with YAML frontmatter
```

## Contributing

Contributions welcome! Each skill should be:

1. **Framework-agnostic**: Explain concepts, not framework-specific APIs
2. **Concise**: Under 500 lines, link to external resources for depth
3. **Practical**: Include decision matrices and code examples
4. **Connected**: Reference related skills for deeper understanding

## License

MIT License - Feel free to use, modify, and distribute.

