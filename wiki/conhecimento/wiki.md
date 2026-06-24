---
type: concept
tags: [ai-memory, hermes, obsidian, wiki]
title: Wiki
description: A wiki markdown (+ SQLite + FTS5 + Git) é a memória durável. Conhecimento vai pra wiki, não pra memory() do Hermes.
timestamp: 2026-06-19T00:00:00+00:00
status: stable
---

# Wiki como Fonte da Verdade

**Criado:** 2026-06-17  
**Última atualização:** 2026-06-19

## Contexto

A `memory()` do Hermes tem limite de 2.200 chars e é uma caixa preta sem estrutura. Sem taxonomia (decisão, gotcha, procedimento, regra ficam tudo misturado), sem versionamento, sem portabilidade.

A solução: **wiki Git pura** — markdown puro versionado em Git, sincronizado automaticamente com GitHub via hook `post-commit`. Vault em `/root/wiki/`, espelhado em `omgiova/wiki`.

| Camada | Tecnologia | Função |
|---|---|---|
| Fonte da verdade | **Markdown puro** | Editável no Obsidian, grep, diff, git |
| Versionamento | **Git** | Histórico, diff, rollback — hook auto-push |
| Sync | **GitHub** (`omgiova/wiki`) | Acesso de qualquer agente, qualquer máquina |
| Acesso | **read_file / search_files** | Hermes lê/escreve direto, sem servidor |
| Visual | **Obsidian** (Windows/Android) | Navegação e edição visual |

## Estrutura atual do vault

Ver [[AGENTS.md]] — fonte da verdade da estrutura, sempre sincronizada com `git ls-files`.

## Como funciona

1. **Escrever:** agente cria/atualiza markdown em `/root/wiki/`
2. **Commitar:** `git commit` — hook `post-commit` faz pull --rebase + push automático
3. **Versionar:** Git → GitHub (`omgiova/wiki`) + Obsidian no Windows via clone
4. **Consultar:** Hermes lê direto via `read_file` / `search_files` (sem MCP intermediário)

## Regras

1. **Conhecimento durável vai pra wiki, não pra `memory()`** — toda decisão, gotcha, procedimento, regra vira markdown
2. **`memory()` do Hermes** é cache de sessão (2.200 chars), não fonte da verdade
3. **Uma página por conceito** — seguir OKF (Open Knowledge Format): `type`, `title`, `description`, `tags`, `timestamp`, `status`
4. **Cross-links** entre páginas relacionadas usando wikilinks (`[[path/to/file.md|display]]`)
5. **Todo arquivo** tem frontmatter OKF completo
6. **`status`** indica confiabilidade: `draft` (em construção), `stable` (confiável), `deprecated` (obsoleto)
7. **Estrutura do vault** em `AGENTS.md` deve refletir `git ls-files` — atualizar junto com qualquer mudança de estrutura

## Histórico

### Fundação (2026-06-18)

Na sessão de 2026-06-18, após uma série de comandos Docker Swarm errados que quebraram n8n e Node-RED (IPVS table corrompida → 502), foi criada a estrutura inicial da wiki — o embrião do que virou este vault.

### 5 Pilares do objetivo final

1. **Memória de longo prazo** para agentes Hermes
2. **Markdown como fonte da verdade** — versionado, legível, buscável
3. **Padrão LLM Wiki do Karpathy** (Raw → Wiki → Schema)
4. **Loop de auto-aprendizado** (sessão → extração → memória)
5. **Handoff entre agentes** (trocar de ferramenta sem perder contexto)

## Navegação

- [[index.md|🏠 Index]]
- [[wiki/infraestrutura/vps.md|🖥 VPS]]
- [[wiki/infraestrutura/hermes.md|🤖 Hermes Config]]
- [[wiki/automacao/firecrawl.md|🔥 Firecrawl]]
- [[wiki/historico/crise-update.md|🔄 Crise update]]
- [[wiki/pendencias/proximos-passos.md|📋 Próximos passos]]
