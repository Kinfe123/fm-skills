# Framework Skills (fm-skills)

A collection of framework-agnostic AI skills for understanding modern web development fundamentals.

## Purpose

These skills help AI assistants understand:
- How web applications are architected (SPA, MPA, hybrid)
- Different rendering strategies and when to use each
- SEO implications of architectural decisions
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

## Learning Path

For best understanding, read the skills in this order:

```
1. web-app-architectures     (Foundation: SPA vs MPA)
        ↓
2. rendering-patterns        (When/where HTML is generated)
        ↓
3. hydration-patterns        (How static becomes interactive)
        ↓
4. routing-patterns          (How navigation works)
        ↓
5. data-fetching-patterns    (How to load data)
        ↓
6. state-management-patterns (Where to store data)
        ↓
7. seo-fundamentals          (Search engine optimization)
        ↓
8. meta-frameworks-overview  (How frameworks implement all of the above)
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

