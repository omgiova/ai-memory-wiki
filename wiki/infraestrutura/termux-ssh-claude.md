---
type: session
tags: [hermes, claude, ssh, termux, remote-control, bug]
title: Problema SSH/Claude — Sessões Travando no Remote Control
description: Diagnóstico dos problemas que causavam travamento de sessões Claude via Remote Control (Termux/Android) — registro do que foi feito em 2026-06-26
timestamp: 2026-06-26T00:00:00-03:00
status: stable
---

# Problema SSH/Claude — Sessões Travando no Remote Control

Diagnóstico realizado em 2026-06-26. Explica por que sessões do Claude via Remote Control apareciam como "Executando..." indefinidamente.

---

## O que foi feito

### Etapa 1 — Processos Claude ativos

```bash
ps aux | grep -i claude | grep -v grep
ps -eo pid,etime,cmd | grep -i claude | grep -v grep
```

**Resultado:** 2 processos encontrados (após kill acidental de uma sessão VS Code no início do diagnóstico):
- PID 3584042 — sem terminal, 1h05min — daemon do VS Code extensionHost (~146 MB RAM)
- PID 3595909 — pts/1, ~17min — sessão Remote Control ativa (a do diagnóstico)

**Aprendizado — daemon do VS Code não é zumbi:**
O processo sem terminal com flags `--output-format stream-json` é iniciado automaticamente pelo VS Code extensionHost ao conectar via SSH, mesmo sem abrir nenhuma aba. Não é zumbi — morre com o VS Code.

**Regra:** nunca dar `kill` em processo claude sem confirmar antes qual `pts/N` é a sessão ativa.
- Processo sem terminal (`?`) com flags `--output-format stream-json` = daemon VS Code → não matar
- Processo com `pts/N` = sessão interativa ativa → só matar com certeza

---

### Etapa 2 — Hooks e curls

```bash
cat ~/.claude/settings.json
grep -rn "curl" ~/.claude/ 2>/dev/null
grep -rn "curl" ~/.claude/ 2>/dev/null | grep -v "max-time"
```

**Resultado:** 7 hooks do sistema `ai-memory` (legado) encontrados no `settings.json`, todos apontando para `http://127.0.0.1:49374` (servidor já down). Os curls tinham `--max-time 0.5s`, então não travavam sessões — mas disparavam em vão a cada evento.

**Ação:** removidos os 7 hooks do `settings.json` e deletada a pasta `/root/.local/share/ai-memory/`. Settings ficou com apenas `model` e `theme`.

---

### Etapa 3 — Logs

```bash
ls -lth ~/.claude/logs/ 2>/dev/null | head -10
grep -rih "timeout\|hang\|error\|killed\|signal\|stuck" ~/.claude/logs/ 2>/dev/null | tail -30
```

**Resultado:** sem pasta `~/.claude/logs/` — o Claude Code nesta versão não gera logs de texto. Busca nos JSONLs de sessões e journalctl não revelou erros de travamento. Os travamentos ficam invisíveis nos logs.

---

### Etapa 4 — Permissões automáticas

Não executada. O erro (prompt de aprovação aparecendo no terminal do Termux mas invisível no app) não se repetiu durante esta sessão de diagnóstico. Retomar se o problema voltar. Ver [[pendencias/proximos-passos.md]] item 8.

---

## Pendências abertas

- **Item 8** — Prompt invisível no app: investigar permissões automáticas para reduzir aprovações manuais
- **Item 9** — Cleanup de zumbi ao iniciar sessão
- **Item 10** — Detecção periódica de processos zumbi em background

Ver [[pendencias/proximos-passos.md]] para acompanhamento.

---

## Resumo

| # | Problema | Status |
|---|---|---|
| 1 | curl sem --max-time (ai-memory legado) | ✅ Resolvido — hooks removidos |
| 2 | Processo zumbi acumulando memória | ⏳ Pendente — itens 9 e 10 |
| 3 | Prompt de aprovação invisível no app | ⏳ Pendente — item 8 |
