---
tags:
- hermes
- procedure
- workflow
- web
- search
tier: semantic
---
# Procedure: Como usar web search e firecrawl

**Data:** 2026-06-18

## Ferramentas disponíveis

### Firecrawl CLI (✅ funciona)
Firecrawl está instalado no VPS (`/usr/local/bin/firecrawl`) e autenticado. Usar **sempre** o CLI, nunca curl na API.

**Buscar na web:**
```bash
firecrawl search "query" --limit N
```

**Extrair conteúdo de URL:**
```bash
firecrawl scrape <url>
```

### web_search tool (❌ NÃO usar)
A tool `web_search` do Hermes retorna erro "No web search provider configured". Isso NÃO significa que não temos busca — o firecrawl CLI funciona. Se uma tool falha, verificar alternativas antes de aceitar que algo não está disponível.

## Regra importante
Nunca aceitar "não configurado" ou "não funciona" como resposta sem investigar. Verificar:
1. Se existe CLI instalado (`which <ferramenta>`)
2. Se tem API key no `.env` (mesmo comentada, pode estar configurada de outra forma)
3. Se tem alternativa funcional (ex: firecrawl CLI ao invés da tool web_search)

## Referência
- CLI: `firecrawl --help` para comandos completos
- Status: `firecrawl --status` (mostra créditos e auth)
- Autenticação via credenciais armazenadas (não precisa de env var)
