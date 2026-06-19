---
type: concept
tags: [ai-memory, hermes, obsidian, wiki]
title: Wiki como Fonte da Verdade
description: A wiki markdown (+ SQLite + FTS5 + Git) é a memória durável. Conhecimento vai pra wiki, não pra memory() do Hermes.
timestamp: 2026-06-17T00:00:00+00:00
---

# Wiki como Fonte da Verdade

**Criado:** 2026-06-17  
**Última atualização:** 2026-06-19

## Contexto

A `memory()` do Hermes tem limite de 2.200 chars e é uma caixa preta sem estrutura. Sem taxonomia (decisão, gotcha, procedimento, regra ficam tudo misturado), sem versionamento, sem portabilidade.

A solução: **ai-memory** — um servidor Rust que indexa markdown puro em SQLite + FTS5, versionado por Git.

| Camada | Tecnologia | Função |
|---|---|---|
| Fonte da verdade | **Markdown puro** | Editável no Obsidian, grep, diff, git |
| Índice | **SQLite + FTS5** | Busca rápida, embeddings opcionais |
| Versionamento | **Git** | Histórico, diff, rollback |
| Servidor | **Rust** | Rápido, baixo consumo |
| Acesso | **MCP tools** (Hermes) | Consulta/escreve automaticamente |
| Visual | **Web UI** (`:49374/web`) + Obsidian | Navegação visual |

## Estrutura atual do vault

```
wiki-fundacao.md               → este arquivo (conceito central)
hermes/                        → Hermes Agent
  conceitos/                   → conceitos transversais
  sessoes/                     → registro de sessões
  todo/                        → próximos passos
hermes-config/                 → configuração do Hermes
  notes/                       → notas gerais
  _rules/                      → regras do agente
infra-vps/vps.md               → Docker, Swarm, Traefik, IPVS, VPS
hermes/procedures/             → procedures do Hermes
```

## Como funciona

1. **Escrever:** agente cria/atualiza markdown no vault (`/root/ai-memory-wiki/wiki/`)
2. **Indexar:** o ai-memory server (Rust) indexa automaticamente em SQLite + FTS5
3. **Consultar:** Hermes usa MCP tools (`memory_query`, `memory_read_page`, etc.)
4. **Versionar:** Git. Vault sincronizado com GitHub (`omgiova/ai-memory-wiki`) e Obsidian no celular

## Regras

1. **Conhecimento durável vai pra wiki, não pra `memory()`** — toda decisão, gotcha, procedimento, regra vira markdown
2. **Memory() do Hermes** é cache de sessão (2.200 chars), não fonte da verdade
3. **Uma página por conceito** — seguir OKF (Open Knowledge Format): `type`, `title`, `description`, `tags`, `timestamp`
4. **Cross-links** entre páginas relacionadas usando wikilinks (`[[path/to/file.md|display]]`)
5. **Todo arquivo** tem frontmatter OKF com `type`, `title`, `description`, `tags`, `timestamp`

## Histórico

### Fundação (2026-06-18)

Na sessão de 2026-06-18, após uma série de comandos Docker Swarm errados que quebraram n8n e Node-RED (IPVS table corrompida → 502), foi criada a estrutura inicial da wiki com 7 páginas em `projetos/infra-vps/` e `projetos/hermes/` — o embrião do que virou este vault.

### 5 Pilares do objetivo final

1. **Memória de longo prazo** para agentes Hermes
2. **Markdown como fonte da verdade** — versionado, legível, buscável
3. **Padrão LLM Wiki do Karpathy** (Raw → Wiki → Schema)
4. **Loop de auto-aprendizado** (sessão → extração → memória)
5. **Handoff entre agentes** (trocar de ferramenta sem perder contexto)

### Pendente

- Migrar conhecimento acumulado de semanas de uso pra wiki
- Configurar auto-improve loop do ai-memory
- Garantir handoff entre agentes (Hermes ↔ Codex etc.)
- Revisar SOUL.md + criar AGENTS.md

## Navegação

- [[hermes/conceitos/wiki-fundacao.md|📄 Este arquivo (wiki-fundacao)]]
- [[infra-vps/vps.md|🖥 VPS (infraestrutura)]]
- [[hermes-config/hermes-config.md|📂 Config]]
- [[hermes/procedures/firecrawl-multi-platform-search.md|🔥 Firecrawl]]
- [[hermes/sessoes/2026-06-18-crise-update-recuperacao.md|🔄 Crise update]]
- [[hermes/todo/proximos-passos.md|📋 Próximos passos]]
