---
type: concept
tags: [ai-memory, hermes, obsidian, wiki]
title: Wiki como Fonte da Verdade
description: A wiki markdown (+ SQLite + FTS5 + Git) é a memória durável. Conhecimento vai pra wiki, não pra memory() do Hermes.
timestamp: 2026-06-17T00:00:00+00:00
---

# Wiki como Fonte da Verdade

**Criado:** 2026-06-17

## Contexto

A `memory()` do Hermes tem limite de 2.200 chars e é uma caixa preta sem estrutura. Sem taxonomia (decisão, gotcha, procedimento, regra ficam tudo misturado), sem versionamento, sem portabilidade.

A solução: **ai-memory** — um servidor Rust que indexa markdown puro em SQLite + FTS5, versionado por Git.

| Camada | Tecnologia | Função |
|---|---|---|
| Fonte da verdade | **Markdown puro** | Editável no Obsidian, grep, diff, git |
| Índice | **SQLite + FTS5** | Busca rápida, embeddings opcionais |
| Versionamento | **Git** | Histórico, diff, rollback |
| Servidor | **Rust** | Rápido, baixo consumo |
| Acesso | **MCP tools** | Hermes consulta/escreve automaticamente |
| Visual | **Web UI** | Browser + Obsidian (Windows) |

## Estrutura real do vault

```
wiki-home.md              → índice raiz
hermes/                   → Hermes Agent (config, skills, regras)
  conceitos/              → conceitos transversais
  decisions/              → decisões de arquitetura
  gotchas/                → armadilhas
  procedures/             → procedimentos
  rules/                  → regras
  sessoes/                → registro de sessões
  todo/                   → próximos passos
hermes-config/            → configuração do Hermes
  notes/                  → notas gerais
  _rules/                 → regras do agente
infra-vps/                → Docker, Swarm, Traefik, VPS
  decisions/
  gotchas/
  procedures/
  rules/
geral/                    → conteúdo não categorizado
  procedures/
  sessoes/
```

## Como funciona

1. **Escrever:** agente cria/atualiza markdown diretamente no filesystem (`/root/ai-memory-wiki/wiki/<projeto>/<tipo>/<nome>.md`)
2. **Indexar:** `docker exec ai-memory ai-memory write-page --path <path> --body "<markdown>"`
3. **Consultar:** Hermes usa MCP tools (`memory_query`, `memory_read_page`, etc.)
4. **Versionar:** Git. O vault tem sync com GitHub (repo `omgiova/ai-memory-wiki`) e Obsidian como IDE visual

## Regras

1. **Conhecimento durável vai pra wiki, não pra `memory()`** — toda decisão, gotcha, procedimento, regra vira markdown
2. **Memory() do Hermes** é cache de sessão (2.200 chars), não fonte da verdade
3. **Uma página por conceito** — seguir OKF (Open Knowledge Format): `type`, `title`, `description`, `tags`, `timestamp`
4. **Cross-links** entre páginas relacionadas usando wikilinks (`[[path/to/page.md|display]]`)
5. **Todo arquivo** tem frontmatter OKF com `type`, `title`, `description`, `tags`, `timestamp`
