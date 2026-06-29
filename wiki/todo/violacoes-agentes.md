---
type: todo
tags: [violacoes, agentes, regras, log]
title: Violações de Regras — Agentes
description: Registro de violações graves cometidas por agentes às regras explícitas do AGENTS.md; serve de memória para evitar reincidência.
timestamp: 2026-06-29T08:35:41-03:00
status: draft
---

# Violações de Regras — Agentes

Registro append-only de violações graves de regras explícitas do [[AGENTS.md]]. Cada item descreve o que foi feito, qual regra foi quebrada e o que deveria ter acontecido.

---

## V1 — Edição de entrada existente no log.md

**Regra violada:** `log.md` é append-only — "Nunca editar entradas existentes" (AGENTS.md)

**Ocorrência 1 — data desconhecida**
O agente editou o log.md para remover uma linha com variável `$(date)` não expandida (linha 326), alterando o histórico existente. Deveria ter appendado uma nota de correção sem tocar no que já estava escrito.

**Ocorrência 2 — 2026-06-29**
Ao documentar o Eval 2-B (3ª execução), o agente substituiu a entrada "APROVADO" por "REPROVADO" em vez de appendar uma nova entrada. Agravante: a instrução "não leia wiki nem outros arquivos" impedia o acesso ao AGENTS.md com as regras. Deveria ter mantido a entrada intacta e appendado correção abaixo.

**Correção aplicada — 2026-06-29**
Hook PreToolUse configurado globalmente em `/root/.claude/settings.json`. Script em `/root/.claude/hooks/log-guard.py`. Bloqueia qualquer Edit ou Write no log.md que modifique conteúdo existente em vez de apenas appendar.

**⏳ Pendente — validar nas próximas sessões:**
- [ ] Hook bloqueia edição de entrada existente (mensagem visível ao agente)
- [ ] Hook permite append legítimo (new_string começa com old_string intacto)
- [ ] Não interfere em outros arquivos .md
- [ ] Funciona em sessões onde AGENTS.md não é lido

**Limitação:** o hook cobre apenas sessões Claude Code — outros processos na VPS não são protegidos.

---

---

## V2 — Não atualizar o log.md após editar a wiki

**Regra violada:** log.md deve ser atualizado após qualquer edição na wiki (AGENTS.md)

**Ocorrência 1 — 2026-06-29**
Após editar `wiki/todo/violacoes-agentes.md` (documentar V1 reincidência + hook), o agente não appendou entrada no log.md. Só foi percebido quando o usuário questionou explicitamente.

---

## Conexões

- [[AGENTS.md]] — fonte das regras violadas
- [[log.md]] — arquivo afetado no V1
