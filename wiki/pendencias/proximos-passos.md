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

1. **Backup GitHub privado do `.hermes`** — criar repo privado `omgiova/hermes-config` com `agent/`, `skills/`, `plugins/`, `scripts/`, `cron/`, `SOUL.md`, `AGENTS.md`, `config.yaml` (excluindo `.env`, `backups/`, `logs/`, `node_modules/`, `node/`, `venv/`). Configurar auto-push via hook ou cron.

2. **Backup geral da VPS** — ativar snapshot automático no painel da Hostinger (KVM 2) ou configurar restic/borg para storage externo. Cobre tudo que o GitHub não cobre (binários, databases, OS).

3. **Migrar conhecimento acumulado** — revisar sessões passadas do Hermes e capturar decisões, gotchas, procedimentos e regras que estão perdidos na memory() ou só na cabeça do usuário

6. **[RASCUNHO] Unificar skills na wiki** — pesquisar documentação técnica de sistemas existentes (ex: MCP tools + LLM Wiki, skills como páginas `type: procedure`) para que skills do Hermes também vivam na wiki e sejam acessíveis a qualquer agente (Claude Code, Codex, Manus). Não implementar sem validação externa. *Registrado em 2026-06-24.*

5. **Print de início de sessão no wiki_review** — adicionar via `on_session_start` (ou hook equivalente) um print `"📓 Ligando o wiki_review"` que aparece quando uma nova sessão começa, indicando que o wiki_review está ativo. Diferente do "iniciando..." atual (que dispara a cada 10 turnos antes de analisar).

4. **Limpar source tree do Hermes** — wiki_review foi reimplementado como plugin em `~/.hermes/plugins/wiki-review/` (2026-06-24), então o source tree já está limpo para esse item. Mas ainda há `AGENTS.md` (redirect) em `/usr/local/lib/hermes-agent/` que pode precisar de limpeza. Confirmar com Giovani antes de marcar como concluído.

### Concluído

- ~~**Criar `wiki_review.py`**~~ — feito (2026-06-23), arquivo em `/root/.hermes/agent/wiki_review.py`

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
