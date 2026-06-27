---
type: procedure
tags: [curador, loop, agentes, automacao, diario, curadoria]
title: Curador da Wiki — Teste 1
description: Agente read-only que analisa uma daily note aleatória e envia curadoria estruturada ao Telegram Geral. MVP do sistema de curadoria de diários.
timestamp: 2026-06-27T00:00:00-03:00
status: draft
---

# Curador da Wiki — Teste 1

Agente MVP que valida a hipótese central do sistema de curadoria: um único `claude -p` com contexto distilado consegue identificar corretamente o que tem valor numa daily note?

## Arquitetura (MVP)

```
[bash curator-teste1.sh]
        │
        ├── sorteia daily aleatória de wiki/diario/
        ├── embute index.md + daily no prompt (sem tool calls)
        ├── claude -p → curadoria estruturada
        └── envia 2 mensagens ao Telegram Geral
              msg 1: conteúdo completo da daily
              msg 2: bloco de curadoria
```

**Agente:** único `claude -p`, sem ferramentas (`--allowedTools ""`).
**Contexto no prompt:** critério Karpathy distilado + OKF + index.md dinâmico + daily completa.
**Permissões na wiki:** nenhuma — read-only via variáveis de shell, sem tool calls.

## Script

`/root/curator-teste1.sh`

```bash
bash /root/curator-teste1.sh
```

Log em `/var/log/curator-teste1.log`.

## Output esperado por execução

**Mensagem 1 — Daily completa:**
```
📅 CURADOR DA WIKI — TESTE 1
Daily sorteada: 2026-06-XX.md

📄 CONTEÚDO COMPLETO:
[conteúdo]
```

**Mensagem 2 — Curadoria:**
```
🔍 CURADORIA — 2026-06-XX.md

MIGRAR PARA ARQUIVO EXISTENTE:
- [item] → [[wiki/pasta/arquivo.md]] — o que fazer

CRIAR PÁGINA NOVA:
- [conceito] → wiki/[pasta]/[arquivo].md — motivo

DESCARTAR:
- [item] — motivo

OBSERVAÇÕES:
[ambiguidades ou decisões pendentes]
```

## Destino: Telegram Geral

- Chat ID: `-1003870518428`
- Thread ID: `1` (tópico Geral)
- Mensagens longas são quebradas automaticamente em chunks de 4096 chars

## Fase de validação

Cada execução sorteia uma daily diferente (aleatória, não necessariamente a menor). O Giovani avalia o output e decide:
- Curadoria correta → prompt validado → expandir para todas as dailies
- Curadoria incorreta → ajustar critérios no prompt e re-testar

Quando validado, o próximo passo é rodar em todas as 11 dailies atuais e montar o pipeline multi-agente.

## Decisões de design

| Decisão | Motivo |
|---|---|
| Contexto embutido no prompt (não via Read tools) | Mais simples, previsível, sem I/O no MVP |
| index.md dinâmico | Reflete o estado real da wiki sem requerer re-configuração |
| Daily aleatória (não a menor) | Evitar viés de validação com amostras fáceis |
| Um único agente no MVP | Validar o critério antes de separar responsabilidades |
| Sem permissão de escrita | Curador nunca altera a wiki; saída vai para Telegram para revisão humana |

## Histórico de execuções

### Tentativa 1 — 2026-06-27T13:18-03:00 — FALHA

**Resultado:** sessão Claude Code derrubada. Nenhuma mensagem enviada ao Telegram. Log vazio.

**O que foi feito:**
O Giovani autorizou a primeira execução do script dentro de uma sessão Claude Code ativa (chat "loop engineering", controle remoto). O Claude Code executou `bash /root/curator-teste1.sh` como ferramenta Bash.

**O que aconteceu passo a passo:**

1. O script iniciou e criou o arquivo de log em `/var/log/curator-teste1.log` (visível pelo timestamp 13:18 no `ls -la`).
2. O script sorteou uma daily, leu o conteúdo dela e do `index.md`, e chegou até o ponto de chamar `claude --allowedTools "" -p "..."` com o prompt enorme (index.md completo + daily completa embutidos).
3. O `claude -p` começou a processar — o processo era visível via `pgrep` e estava rodando há 2min18s quando verificado.
4. O Bash tool do Claude Code tem timeout de ~2 minutos. O timeout estourou antes do `claude -p` terminar. A ferramenta reportou timeout, mas o subprocesso `claude -p` continuou rodando em background.
5. O Claude Code da sessão vigente identificou que o processo ainda estava rodando e propôs matar o processo pendente para rodar em background.
6. O Giovani aprovou o kill.
7. A sessão caiu imediatamente após a aprovação.

**Por que a sessão caiu:**
O processo filho `claude -p` (PID 3708100) e a sessão Claude Code pai tinham PIDs vizinhos no mesmo grupo de processo. O kill provavelmente atingiu o processo pai ou ambos compartilhavam um recurso (terminal, pipe) que, ao ser encerrado no filho, derrubou o pai.

**Por que o log ficou vazio:**
O arquivo `/var/log/curator-teste1.log` foi criado com 0 bytes. A função `log()` do script usa `echo "..." >> "$LOG"` — o arquivo é criado pela redireção mas nada é escrito se o processo for interrompido antes da primeira chamada `log()`. Com `set -euo pipefail`, quando o subprocesso `claude -p` foi morto externamente, o script abortou sem executar nenhuma linha de log subsequente. Isso também indica que o `log "Iniciando Teste 1..."` (que vem antes do `claude -p`) também não foi gravado, sugerindo que o script pode ter iniciado e o arquivo de log sido criado pela própria sessão anterior de diagnóstico, não pela execução do curator.

**Evidências coletadas na sessão seguinte (13:22):**
- `pgrep -a claude` retornou PID 3708100 com uptime de 3min45s — este era a sessão Claude Code atual, não um resíduo do curator
- `ps -p 3708100 3700880` retornou vazio — os processos da tentativa anterior já haviam encerrado
- `/var/log/curator-teste1.log` existia com 0 bytes, timestamp 13:18
- Nenhuma mensagem recebida no Telegram Geral

**Causa raiz identificada:**
Chamar `claude -p` como subprocesso dentro de uma sessão Claude Code ativa é incompatível com o funcionamento do Bash tool: o timeout de 2 minutos é insuficiente para prompts grandes, e matar o subprocesso em condições de grupo de processo compartilhado derruba a sessão pai. O script foi projetado para ser chamado externamente (cron, terminal direto), não de dentro de uma sessão interativa do Claude Code.

**O que ainda não foi feito:**
Nenhum fix foi aplicado ao script. A documentação deste erro vem antes de qualquer alteração no código — conforme solicitado pelo Giovani.

### Tentativa 2 — planejada — aguardando execução

**Aprendizado da tentativa 1:** o problema não foi chamar o script de dentro de uma sessão Claude Code ativa — isso faz parte do teste e está validado no [[wiki/conhecimento/plano-implementacao-loop.md|Plano de Loops]]. O problema foi o **bloqueio**: o Claude Code ficou esperando o script terminar, o timeout de 2 minutos do Bash tool estourou com o processo ainda rodando, e o kill subsequente derrubou a sessão.

**Correção identificada:** manter o script sendo chamado da sessão ativa, mas desacoplar o processo com `nohup ... &` para que o Claude Code dispare e não fique bloqueado esperando. O script roda por conta própria e o Telegram sinaliza o fim.

**Fixes a aplicar no script antes da execução:**

1. **Mover o primeiro `log()` para antes de qualquer execução** — atualmente a primeira linha de log vem depois de `ls` e `shuf`, o que significa que uma falha cedo deixa o log vazio sem diagnóstico.
2. **Adicionar `timeout 300` no `claude -p`** — se o modelo demorar mais de 5 minutos, o script mata o processo e registra o erro em vez de ficar preso indefinidamente. Valor escolhido com margem: na tentativa 1 o processo já estava em 2min18s quando verificado e provavelmente ainda não havia terminado.
3. **Tratar `CURADORIA` vazia sem abortar** — com `set -euo pipefail`, se o `claude -p` for interrompido ou retornar vazio, o script aborta silenciosamente. Precisa de tratamento explícito para registrar o erro e encerrar com mensagem de falha no log.

**Forma de chamada:**
```bash
nohup bash /root/curator-teste1.sh > /var/log/curator-teste1.log 2>&1 &
```
Chamado de dentro de uma sessão Claude Code ativa. O `nohup &` desacopla o processo — a sessão fica livre, o script roda independente, o Telegram confirma o fim.

**O que esta tentativa valida (além da tentativa 1):**
- O padrão `nohup &` dentro de sessão Claude Code ativa resolve o problema de bloqueio
- O `timeout 300` é suficiente para o `claude -p` com prompt grande terminar com sucesso
- O log registra corretamente início, progresso e fim da execução

---

## Conexões

- [[wiki/conhecimento/plano-implementacao-loop.md|Plano de Implementação — Loops]] — contexto técnico e arquitetura de loops agênticos
- [[wiki/infraestrutura/telegram-topicos.md|Telegram Tópicos]] — IDs do Telegram usados no envio
- [[wiki/automacao/wiki-review.md|Wiki Review]] — outro agente de automação da wiki
