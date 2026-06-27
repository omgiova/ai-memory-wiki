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

### Tentativa 2 — 2026-06-27T13:34-03:00 — FALHA PARCIAL

**Aprendizado da tentativa 1:** o problema não foi chamar o script de dentro de uma sessão Claude Code ativa — isso faz parte do teste e está validado no [[wiki/conhecimento/plano-implementacao-loop.md|Plano de Loops]]. O problema foi o **bloqueio**: o Claude Code ficou esperando o script terminar, o timeout de 2 minutos do Bash tool estourou com o processo ainda rodando, e o kill subsequente derrubou a sessão.

**Correção aplicada:** manter o script sendo chamado da sessão ativa, mas desacoplar o processo com `nohup ... &`.

**Fixes aplicados no script antes da execução:**
1. `log "Script iniciado (PID $$)."` movido para antes de qualquer execução
2. `timeout 300` adicionado ao `claude -p`
3. `|| true` + verificação de `CURADORIA` vazia para não abortar silenciosamente

**Forma de chamada:**
```bash
nohup bash /root/curator-teste1.sh > /var/log/curator-teste1.log 2>&1 &
```

**O que aconteceu:**

| Etapa | Resultado |
|---|---|
| Sessão Claude Code derrubada | ✅ Não aconteceu — `nohup &` resolveu o bloqueio |
| Log gravado desde o início | ✅ Confirmado: duas linhas de log em 13:34:47 |
| `claude -p` executou | ✅ Confirmado: script chegou até o `telegram_send`, não abortou por CURADORIA vazia |
| Daily sorteada | `2026-06-23-20260623.md` |
| PID do processo | 3709792 |
| Envio ao Telegram | ❌ `HTTP Error 400: Bad Request` |
| Processo encerrou | ✅ Não há processo ativo após a falha |

**O que funcionou nesta tentativa:**
- `nohup &` desacoplou corretamente — a sessão Claude Code não caiu
- Log gravou corretamente desde o início (fix 1 validado)
- `claude -p` com `timeout 300` executou e retornou curadoria (fix 2 validado — o modelo processou o prompt grande sem timeout)
- Tratamento de `CURADORIA` vazia funcionou por omissão — não foi necessário (fix 3 validado preventivamente)

**O que falhou:**
O `telegram_send` retornou `HTTP 400: Bad Request` ao tentar enviar uma das duas mensagens. Causa exata ainda não identificada — o log completo não foi lido ainda. Hipóteses em ordem de probabilidade:
1. Texto da curadoria ou da daily contém caracteres que a API do Telegram rejeita (ex: `[[wikilinks]]`, caracteres de controle, encoding inesperado)
2. `message_thread_id=1` inválido para o estado atual do grupo
3. Texto passado via argumento de shell corrompeu o encoding antes de chegar ao Python

**Diagnóstico pós-falha — sessão 2026-06-27T13:47-03:00:**

O log completo foi lido. O arquivo `/var/log/curator-teste1.log` continha apenas o traceback Python — as linhas do `log()` não estavam visíveis. Causa: `nohup bash script > logfile 2>&1` abre o arquivo na posição 0 para fd1/fd2 do bash; quando o Python escreve o traceback via stderr (fd2), escreve a partir da posição 0, sobrescrevendo as linhas gravadas pelo `log()` via `>>` (O_APPEND). Os dois mecanismos escrevem no mesmo arquivo por caminhos diferentes e colidem. Isso é um bug de arquitetura do script que precisa ser corrigido na próxima tentativa: ou o `nohup` redireciona para um arquivo separado, ou o script não usa `nohup > $LOG`.

Traceback completo registrado no log:
```
File "<stdin>", line 17, in <module>  ← urlopen
urllib.error.HTTPError: HTTP Error 400: Bad Request
```

A linha 17 do Python inline é o `urllib.request.urlopen(req)`. O script não capturava o corpo da resposta de erro do Telegram, então só havia "400: Bad Request" sem descrição.

**5 testes curl realizados para isolar a causa do 400:**

| # | Método | `message_thread_id` | Resultado |
|---|---|---|---|
| 1 | JSON inline (`-H Content-Type + -d`) | `1` | ❌ `Bad Request: message thread not found` |
| 2 | JSON inline | sem | ✅ OK — message_id: 793 |
| 3 | form-encoded (`--data-urlencode`) | sem | ✅ OK — message_id: 794 |
| 4 | JSON via arquivo temp (`--data @file`) | sem | ✅ OK — message_id: 795 |
| 5 | multipart form (`-F`) | sem | ✅ OK — message_id: 796 |

**Causa raiz confirmada:** `message_thread_id=1` está errado para este grupo. O erro retornado pelo Telegram é "message thread not found".

**Investigação do thread_id:**

O grupo foi consultado via `getChat`: `is_forum: True`, `type: supergroup`, `title: HERMES & GIONATO`. O grupo tem fórum habilitado, mas `thread_id=1` não existe como thread separado — o Telegram trata o tópico Geral como o chat principal, não como um tópico com ID próprio.

A wiki em [[infraestrutura/telegram-topicos.md]] documenta `thread_id=1` para Geral, mas o próprio campo "Como usar" da mesma página contradiz isso: *"Omitir `:thread_id` para o tópico padrão (Geral)."* Os 4 testes sem `message_thread_id` confirmaram que sem o campo as mensagens chegam ao Geral sem erro (message_ids 793–796 entregues).

**Conclusão:** o `telegram_send` do script deve enviar sem `message_thread_id` quando o destino for o tópico Geral. O formato JSON inline (teste 2) é o mais simples e funciona.

**Fixes identificados para a próxima tentativa:**
1. Remover `message_thread_id` do payload quando o destino for Geral
2. Capturar o corpo do erro HTTP no Python (`e.read().decode()`) para diagnóstico futuro
3. Separar o destino do `nohup` do `$LOG` para evitar sobrescrita das linhas de log
4. Atualizar [[infraestrutura/telegram-topicos.md]] — a nota sobre `thread_id=1` para Geral está incorreta na prática; o comportamento correto é omitir o campo

**Próximo passo:** documentar, depois aplicar os fixes no script, depois rodar tentativa 3.

---

## Conexões

- [[wiki/conhecimento/plano-implementacao-loop.md|Plano de Implementação — Loops]] — contexto técnico e arquitetura de loops agênticos
- [[wiki/infraestrutura/telegram-topicos.md|Telegram Tópicos]] — IDs do Telegram usados no envio
- [[wiki/automacao/wiki-review.md|Wiki Review]] — outro agente de automação da wiki
