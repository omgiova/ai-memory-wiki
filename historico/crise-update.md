---
type: session
tags: [github, hermes, sessoes, telegram]
title: Sessão: 2026-06-18 — Crise de update + Recuperação de Sessões
description: Múltiplos /update corromperam state.db. Backup automático salvou. hermes import restaurou em 10s.
timestamp: 2026-06-18T00:00:00+00:00
status: stable
---

# Sessão: 2026-06-18 — Crise de update + Recuperação de Sessões

## O que aconteceu

Múltiplos `/update` no Hermes corromperam o `state.db` (WAL mode) — sessões sumiram. Backup automático `pre_update_backup: true` salvou antes de cada update. `hermes import --force ~/.hermes/backups/pre-update-*.zip` restaurou tudo em 10s.

Total final: **138 sessões, 11.005 mensagens**.

## Configurações alteradas

```yaml
updates:
  pre_update_backup: true
  non_interactive_local_changes: stash
```

## Causa raiz suspeita

`state.db` usa `journal_mode=WAL` — gateway morre durante update → WAL truncado → database malformed. Suspeita: trocar pra `journal_mode=DELETE` evitaria a corrupção.

## Procedimento de recovery (comprovado)

1. Update roda, state.db some
2. Verificar: `ls ~/.hermes/backups/pre-update-*.zip`
3. Restaurar: `hermes import --force ~/.hermes/backups/pre-update-*.zip`
4. Verificar: `sqlite3 ~/.hermes/state.db "PRAGMA integrity_check;"`
5. Pronto — 10 segundos, zero perda

## Resolução definitiva (2026-06-20)

Causa raiz identificada e corrigida. O problema não era só corrupção de WAL — o agente havia modificado 3 skills bundled (git-tracked), causando conflito no stash apply durante o update, que por sua vez rodava `git reset --hard HEAD` e apagava todos os untracked.

Três fixes aplicados em `/root/.hermes/`:
1. `.git/info/exclude` — arquivos críticos de runtime nunca entram no stash
2. `non_interactive_local_changes: stash` — stash é restaurado em vez de dropado
3. `SOUL.md` — regra: verificar `git ls-files` antes de editar qualquer skill

Ver detalhes completos em [[diario/2026-06-20.md|sessão 2026-06-20]].

## Pendente

- Abrir issue no Hermes GitHub sobre corrupção de state.db durante update do gateway
- Testar `journal_mode=DELETE` quando gateway puder ser parado
