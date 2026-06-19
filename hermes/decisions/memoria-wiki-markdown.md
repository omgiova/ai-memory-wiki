---
type: decision
tags: [ai-memory, alexa, decision, docker, github, hermes, mcp, n8n, obsidian, projetos, remotion, traefik, vps, wiki]
title: Decisão: Memória Baseada em Markdown + FTS5 + SQLite + Git
description: Camada Tecnologia Função --------- Fonte da verdade Markdown puro Editável no Obsidian, grep, diff, git Índice SQLite + FTS5 Busca全文 rápida, embeddings opcionais Versionamento Git Histórico, diff, ...
timestamp: 2026-06-17T00:00:00+00:00
---


## Decisão: Memória Baseada em Markdown + FTS5 + SQLite + Git

**Data:** 2026-06-17  
**Contexto:** A memory() do Hermes tem limite de 2.200 chars e é uma caixa preta sem estrutura.

### O problema
- memory() do Hermes = cache de sessão, não é memória durável
- Sem taxonomia (decisão, gotcha, procedimento, regra ficam tudo misturado)
- Sem versionamento (não dá diff, não dá rollback)
- Sem portabilidade (preso ao Hermes)

### A solução: ai-memory + wiki markdown

| Camada | Tecnologia | Função |
|---|---|---|
| Fonte da verdade | **Markdown puro** | Editável no Obsidian, grep, diff, git |
| Índice | **SQLite + FTS5** | Busca全文 rápida, embeddings opcionais |
| Versionamento | **Git** | Histórico, diff, rollback |
| Servidor | **Rust** | Rápido, baixo consumo |
| Acesso | **MCP tools** | Hermes consulta/escreve automaticamente |
| Visual | **Web UI** | Browser + Obsidian (Windows) |

### Estrutura adotada
```
projetos/
  infra-vps/      → Docker, Swarm, Traefik
  n8n/            → workflows
  node-red/       → flows, Alexa
  remotion/       → vídeos
  hermes/         → config, skills, regras
_global/          → conhecimento geral
```

Dentro de cada projeto: `decisions/`, `gotchas/`, `procedures/`, `rules/`.

### Próximos passos
- Sincronizar wiki com GitHub (repo privado)
- Conectar Obsidian no Windows ao mesmo repo
- Popular com conhecimento acumulado

## 📂 Navegação
[[hermes/decisions/INDEX.md|📂 Voltar para decisions]]

## 🔗 Conexões entre projetos
- [[geral/sessoes/2026-06-18-crise-update-recuperacao.md|2026-06-18-crise-update-recuperacao]]
- [[hermes/procedures/implementar-memoria-arquivos.md|implementar-memoria-arquivos]]
- [[hermes/rules/conhecimento-na-wiki.md|conhecimento-na-wiki]]
- [[hermes/sessoes/2026-06-18-recuperacao-e-wiki.md|2026-06-18-recuperacao-e-wiki]]
- [[hermes/todo/proximos-passos.md|proximos-passos]]
- [[hermes-config/notes/arquitetura-vps.md|arquitetura-vps]]

