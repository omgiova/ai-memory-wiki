---
type: procedure
tags: [ai-memory, firecrawl, hermes, procedure, web, workflow]
title: Firecrawl — Busca e Scraping Multi-Plataforma
description: Usar firecrawl search com sintaxe site: para buscar em plataformas específicas. Preferir sobre buscas genéricas quando a fonte importa.
timestamp: 2026-06-19T00:00:00+00:00
status: stable
---

# Firecrawl — Busca Multi-Plataforma

## Quando usar

- Quando a **fonte importa** (ex: "quero só resultados do GitHub, não do Stack Overflow")
- Para buscar issues, PRs, discussões, posts ou documentação em plataformas específicas
- Quando a busca genérica retorna ruído demais

## Quando NÃO usar

- Buscas gerais sem restrição de plataforma — use a ferramenta de busca padrão
- Sites que bloqueiam scraping agressivamente (pode retornar HTML vazio ou erro 403)

## Comandos

```bash
# GitHub — issues, repos, code
firecrawl search "site:github.com <query>" --scrape --limit 3

# Reddit — discussões, experiências
firecrawl search "site:reddit.com <query>" --scrape --limit 3

# X/Twitter — posts e threads
firecrawl search "site:x.com OR site:twitter.com <query>" --scrape --limit 3

# Documentação oficial de um projeto
firecrawl search "site:docs.exemplo.com <query>" --scrape --limit 5
```

Sempre usar `--scrape` para ver o conteúdo completo da página, não só o snippet.

## Troubleshooting

| Sintoma | Causa provável | Ação |
|---|---|---|
| Retorna 0 resultados | Query muito específica ou site bloqueado | Simplificar query ou trocar plataforma |
| HTML vazio no `--scrape` | Site usa JS dinâmico (SPA) | Tentar sem `--scrape` para ver o snippet |
| Timeout | Site lento ou bloqueando | Reduzir `--limit` ou tentar novamente |

## Gotcha: web_search do Hermes não funciona

A tool `web_search` do Hermes retorna erro "No web search provider configured". Isso **não significa que não há busca** — o firecrawl CLI funciona e é o método correto. Nunca aceitar "não configurado" sem verificar se existe alternativa CLI.

```bash
which firecrawl        # deve retornar /usr/local/bin/firecrawl
firecrawl --status     # mostra créditos e auth
```

## Conexões

- [[infraestrutura/hermes.md|Hermes Config]] — onde firecrawl está configurado na stack
