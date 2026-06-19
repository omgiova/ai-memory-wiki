---
type: procedure
tags: [ai-memory, firecrawl, hermes, procedure, web, workflow]
title: Usar Firecrawl para Busca e Scraping Multi-Plataforma
description: Usar firecrawl search com sintaxe site: para buscar em plataformas específicas.
timestamp: 2026-06-19T00:00:00+00:00
---


# Firecrawl — Busca Multi-Plataforma

Usar `firecrawl search` com sintaxe `site:` para buscar em plataformas específicas.

## Comandos

```bash
# GitHub
firecrawl search "site:github.com <query>" --scrape --limit 3

# Reddit
firecrawl search "site:reddit.com <query>" --scrape --limit 3

# X/Twitter
firecrawl search "site:x.com OR site:twitter.com <query>" --scrape --limit 3
```

Sempre usar `--scrape` para ver o conteúdo completo. Uma única ferramenta cobre as 3 plataformas, sem custo extra.

## 📂 Navegação
[[geral/procedures/INDEX.md|📂 Voltar para procedures]]

