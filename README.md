# pagination-independence-architecture

Open-source skill for **designing category and listing architecture so that a site's indexable value does not depend on paginated pages**, using demand-backed indexable subcategories as the alternative to deep pagination. Before pruning a class of URLs from the index, re-architect so the internal link graph no longer depends on them: pagination handling is the last step, not the first. The agent delivers a sequenced plan — audit → subcategory design → coverage verification → conditional pruning, rolled out per category — never a pagination directive as a standalone recommendation. Applies to e-commerce catalogues, editorial archives and directories. Load it into any MCP-capable AI agent (Claude, Cursor, Codex, n8n, etc.).

## When it triggers

When the user asks whether to noindex, canonicalise, block or prune paginated URLs (`?page=`, `?p=`, `/page/2/`); mentions pagination handling, page-2+ indexation, infinite scroll and crawlability, orphaned product or article pages, "products only reachable from page 7", crawl budget wasted on listing pages, thin or duplicate category pages, faceted navigation index bloat, or splitting a large category into subcategories. Also for "how do we cut paginated URLs without losing products" and listing-page taxonomy redesign for large catalogues.

## What this repo contains

- `SKILL.md` — the core principle, the five phases (the pagination-independence audit, demand-backed leaf subcategories, page-1 coverage verification, conditional pruning per category, governance), the anti-patterns, the pre-flight, and the pointers into the reference files.
- `references/measurement-recipes.md` — the Phase 1 measurement recipes: the queue guard before any absence diff, same-snapshot consistency and the depth bound, classifying inlink sources, the listing stability test, item-level page-1 coverage from validated grid extraction, reading an absence off a fetched page.
- `references/subcategory-sizing.md` — the Phase 2 sizing recipes: inventorying existing landing-page mechanisms, sizing and pricing the items-per-page lever, exact facet membership and greedy set cover, the demand gate and the two tracks.
- `references/pruning-evidence.md` — the Phase 4 preconditions and evidence: the rendered-page check and the evidence for pruning.
- `LICENSE` — CC BY 4.0.

## Install

Install the whole skill directory — `SKILL.md` and `references/` must travel together — into your MCP-capable agent's skills directory, following the client-specific path, and restart the client.

## Contributing

Open an issue or a PR at [github.com/rebelytics/pagination-independence-architecture](https://github.com/rebelytics/pagination-independence-architecture).

## Credits

- Original author: [Eoghan Henn](https://www.rebelytics.com) / [LinkedIn](https://www.linkedin.com/in/eoghanhenn)

## License

CC BY 4.0 — the licence text is in [LICENSE](./LICENSE): the full legal code, unmodified, as Creative Commons publishes it; the [human-readable summary](https://creativecommons.org/licenses/by/4.0/) says what it allows. Use it, fork it, adapt it, monetise it. Keep the attribution.

Copyright (c) 2026 Eoghan Henn / rebelytics.com
