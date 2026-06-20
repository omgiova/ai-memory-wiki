---
type: procedure
tags: [ai-memory, firecrawl, hermes, procedure, web, workflow]
title: Firecrawl — Busca e Scraping Multi-Plataforma
description: Usar firecrawl search com sintaxe site: para buscar em plataformas específicas.
timestamp: 2026-06-19T00:00:00+00:00
status: draft
---

# Firecrawl — Busca Multi-Plataforma

Usar `firecrawl search` com sintaxe `site:` para buscar em plataformas específicas.

```bash
# GitHub
firecrawl search "site:github.com <query>" --scrape --limit 3

# Reddit
firecrawl search "site:reddit.com <query>" --scrape --limit 3

# X/Twitter
firecrawl search "site:x.com OR site:twitter.com <query>" --scrape --limit 3
```

Sempre usar `--scrape` para ver o conteúdo completo.
