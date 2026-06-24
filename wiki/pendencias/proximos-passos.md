---
type: todo
tags: [github, hermes, obsidian, todo, wiki]
title: Próximos Passos
description: Pendências ativas da wiki — migração de conhecimento, AGENTS.md, revisão do SOUL.md e estrutura de diário/raw
timestamp: 2026-06-19T00:00:00+00:00
status: stable
---

## Próximos Passos

### Pendente

1. **Criar `wiki_review.py`** — módulo `/root/.hermes/agent/wiki_review.py` com `spawn_wiki_review_thread` e `_increment_and_check_counter`; o trigger já existe em `turn_finalizer.py` mas o import falha silenciosamente. Ver [[wiki/automacao/wiki-review.md]] para spec completa.

2. **Migrar conhecimento acumulado** — revisar sessões passadas do Hermes e capturar decisões, gotchas, procedimentos e regras que estão perdidos na memory() ou só na cabeça do usuário

### Concluído

- ~~**Sincronizar wiki com GitHub**~~ — feito (repo `omgiova/wiki`)
- ~~**Conectar Obsidian**~~ — feito (espelhado no Windows via clone)
- ~~**Criar AGENTS.md**~~ — feito ([[AGENTS.md]])
- ~~**Criar estrutura `diario/`**~~ — feito
- ~~**Criar pasta `raw/`**~~ — feito
- ~~**Revisar SOUL.md**~~ — feito (regra bundled skills + referência à wiki em `/root/wiki/`)
- ~~**Migrar de ai-memory para wiki Karpathy**~~ — feito (Docker parado, MCP desabilitado, hook auto-push configurado)

## 🔗 Conexões entre projetos
- [[wiki/historico/crise-update.md|2026-06-18-crise-update]]
- [[wiki/conhecimento/wiki.md|wiki (histórico + conceito)]]
- [[wiki/infraestrutura/vps.md|vps (IPVS, hardware, stack)]]
