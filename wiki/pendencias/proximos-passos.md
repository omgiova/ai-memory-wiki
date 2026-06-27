---
type: todo
tags: [github, hermes, obsidian, todo, wiki]
title: Próximos Passos
description: Pendências ativas da wiki — migração de conhecimento, AGENTS.md, revisão do SOUL.md e estrutura de diário/raw
timestamp: 2026-06-18T21:00:00-03:00
status: stable
---

## Próximos Passos

### Pendente

1. **Backup GitHub privado do `.hermes`** — criar repo privado `omgiova/hermes-config` com `agent/`, `skills/`, `plugins/`, `scripts/`, `cron/`, `SOUL.md`, `AGENTS.md`, `config.yaml` (excluindo `.env`, `backups/`, `logs/`, `node_modules/`, `node/`, `venv/`). Configurar auto-push via hook ou cron.

2. **Backup geral da VPS** — ativar snapshot automático no painel da Hostinger (KVM 2) ou configurar restic/borg para storage externo. Cobre tudo que o GitHub não cobre (binários, databases, OS).

3. **Migrar conhecimento acumulado** — revisar sessões passadas do Hermes e capturar decisões, gotchas, procedimentos e regras que estão perdidos na memory() ou só na cabeça do usuário

6. **[RASCUNHO] Unificar skills na wiki** — pesquisar documentação técnica de sistemas existentes (ex: MCP tools + LLM Wiki, skills como páginas `type: procedure`) para que skills do Hermes também vivam na wiki e sejam acessíveis a qualquer agente (Claude Code, Codex, Manus). Não implementar sem validação externa. *Registrado em 2026-06-24.*

5. **Print visual do wiki_review no terminal** — o plugin atualmente só escreve `logger.info`, que não aparece na tela do usuário. Giovani quer um print visível quando o wiki_review dispara e quando termina. Explorar depois que o sistema estiver rodando estável: opções são `on_session_start` hook, gateway callback, ou acesso ao objeto `agent` via algum mecanismo futuro. *Registrado 2026-06-24.*

4. **Limpar source tree do Hermes** — gatilho do wiki_review removido de `agent/turn_finalizer.py` (2026-06-24). Ainda há `AGENTS.md` (redirect) em `/usr/local/lib/hermes-agent/` que pode precisar de limpeza. Confirmar com Giovani antes de marcar como concluído.

8. **Resolver prompt de aprovação invisível no Remote Control** — quando Claude pede aprovação de ferramenta, prompt aparece no Termux (pts/0) mas não no app. Sessão fica aparentemente travada. Investigar configurar permissões automáticas para reduzir aprovações manuais. Ver [[infraestrutura/termux-ssh-claude.md]]. *Registrado 2026-06-26.*

9. **Cleanup automático de processos zumbi do Claude** — sessões canceladas pelo app deixam processo Claude vivo no Termux. Implementar script que mata processos órfãos ao iniciar nova sessão. Ver [[infraestrutura/termux-ssh-claude.md]]. *Registrado 2026-06-26.*

10. **Detecção periódica de processos zumbi do Claude** — criar mecanismo que monitore com certa frequência processos Claude acumulados em background, evitando consumo silencioso de memória sem o usuário saber. Diferente do item 9 (que age só no início de sessão), este deve rodar de forma contínua ou agendada. Ver [[infraestrutura/termux-ssh-claude.md]]. *Registrado 2026-06-26.*

11. **Validar fix do Obsidian Git (merge.autostash)** — `git config merge.autostash true` configurado no Windows em 2026-06-26. Confirmar que Pull funciona consistentemente sem erros de "would be overwritten by merge". Quando validado, mover [[infraestrutura/obsidian-git.md]] de `status: draft` para `status: stable`. *Registrado 2026-06-26.*

### Concluído

- ~~**Criar `wiki_review.py`**~~ — feito (2026-06-23), arquivo em `/root/.hermes/agent/wiki_review.py`

- ~~**Sincronizar wiki com GitHub**~~ — feito (repo `omgiova/wiki`)
- ~~**Conectar Obsidian**~~ — feito (espelhado no Windows via clone)
- ~~**Criar AGENTS.md**~~ — feito ([[AGENTS.md]])
- ~~**Criar estrutura `diario/`**~~ — feito
- ~~**Criar pasta `raw/`**~~ — feito
- ~~**Revisar SOUL.md**~~ — feito (regra bundled skills + referência à wiki em `/root/wiki/`)
- ~~**Migrar de ai-memory para wiki Karpathy**~~ — feito (Docker parado, MCP desabilitado, hook auto-push configurado)
- ~~**Corrigir curl sem --max-time no script de startup**~~ — resolvido em 2026-06-26: hooks do ai-memory (legado) removidos do `settings.json` e pasta `/root/.local/share/ai-memory/` deletada. Problema não se aplicava (curls já tinham timeout), mas limpeza feita.

## 🔗 Conexões entre projetos
- [[wiki/historico/crise-update.md|2026-06-18-crise-update]]
- [[wiki/conhecimento/wiki.md|wiki (histórico + conceito)]]
- [[wiki/infraestrutura/vps.md|vps (IPVS, hardware, stack)]]
- [[wiki/conhecimento/orquestrador.md|orquestrador — panorama de saúde do vault e débito técnico]]
