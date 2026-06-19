---
type: session
tags: [github, hermes, projetos, sessoe, telegram]
title: Sessão: 2026-06-18 — Crise de update + Recuperação de Sessões
description: preupdatebackup: true noninteractivelocalchanges: discard
timestamp: 2026-06-18T00:00:00+00:00
---


# Sessão: 2026-06-18 — Crise de update + Recuperação de Sessões

## O que aconteceu
- Múltiplos `/update` no Hermes corromperam o `state.db` (WAL mode) — sessões sumiram
- Backup automático `pre_update_backup: true` salvou antes de cada update (comprovado 3x)
- `hermes import --force ~/.hermes/backups/pre-update-*.zip` restaurou tudo em 10s (comprovado 3x)
- Importamos 76 sessões antigas dos JSONs + 61 sessões do export do Telegram
- Total final: **138 sessões, 11.005 mensagens**

## Configurações alteradas
```yaml
updates:
  pre_update_backup: true
  non_interactive_local_changes: discard
```

## Causa raiz suspeita
- `state.db` usa `journal_mode=WAL` — gateway morre de repente durante update → WAL truncado → database malformed
- Suspeita: trocar pra `journal_mode=DELETE` evitaria a corrupção, mas não testado

## Procedimento de recovery (comprovado)
1. Update rodeia, state.db some
2. Verificar: `ls ~/.hermes/backups/pre-update-*.zip`
3. Restaurar: `hermes import --force ~/.hermes/backups/pre-update-*.zip`
4. Verificar: `sqlite3 ~/.hermes/state.db "PRAGMA integrity_check;"`
5. Pronto — 10 segundos, zero perda

## Arquivos de import disponíveis
- `/root/ChatExport_2026-06-15/result_fixed.json` (6.200 msgs, mai-jun 2026)
- `/root/ChatExport_2026-06-15/ChatExport_2026-06-15 (1)/result.json` (997 msgs, jun 2026)
- Script: `/root/import_telegram_export.py`

## Pendente
- Abrir issue no Hermes GitHub sobre corrupção de state.db durante update do gateway
- Testar `journal_mode=DELETE` quando gateway puder ser parado

## 📂 Navegação
[[geral/sessoes/INDEX.md|📂 Voltar para sessoes]]

## 🔗 Conexões entre projetos
- [[hermes/decisions/memoria-wiki-markdown.md|memoria-wiki-markdown]]
- [[hermes/procedures/implementar-memoria-arquivos.md|implementar-memoria-arquivos]]
- [[hermes/rules/conhecimento-na-wiki.md|conhecimento-na-wiki]]
- [[hermes/sessoes/2026-06-18-recuperacao-e-wiki.md|2026-06-18-recuperacao-e-wiki]]
- [[hermes/todo/proximos-passos.md|proximos-passos]]

