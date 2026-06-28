---
type: session
tags: [curador, historico, tentativas, automacao, diario, curadoria]
title: Curador da Wiki — Histórico de Desenvolvimento
description: Registro completo de todas as tentativas, scripts, system prompts e decisões de design do curador desde a tentativa 1 até a v6/v5 estável.
timestamp: 2026-06-28T00:00:00-03:00
status: stable
---

# Curador da Wiki — Histórico de Desenvolvimento

Agente de curadoria de diários. Cada tentativa valida uma hipótese e acumula aprendizado para a próxima.

## Regras gerais do teste

- **Formato de entrega:** todas as mensagens ao Telegram usam `sendRichMessage` (Bot API 10.1) com `rich_message.markdown`. Ver [[infraestrutura/telegram-send-rich-message.md]]
- **Frontmatter:** sempre removido antes do envio — nunca incluir o bloco `---` YAML nas mensagens
- **Thread Geral:** omitir `message_thread_id` no payload (ver [[infraestrutura/telegram-topicos.md]])
- **Agente é read-only:** o `claude -p` só produz curadoria — nunca entrega a daily. A daily é enviada pelo script diretamente, antes de chamar o agente
- **Execução:** sempre via `nohup bash /root/curator-teste1.sh > /var/log/curator-teste1.nohup.log 2>&1 &` — o `$LOG` é exclusivo das chamadas `log()` do script; o nohup redireciona para arquivo separado `.nohup.log`

## Arquitetura (a partir da tentativa 3)

```
[bash curator-teste1.sh]
        │
        ├── sorteia daily aleatória de wiki/diario/
        ├── envia daily via sendRichMessage (script, sem agente)  ← novo
        ├── embute index.md + daily no prompt do claude -p
        ├── claude -p → curadoria estruturada
        └── envia curadoria via sendRichMessage
```

**Agente:** único `claude -p`, sem ferramentas (`--allowedTools ""`).
**Contexto no prompt:** critério Karpathy distilado + OKF + index.md dinâmico + daily completa.
**Permissões na wiki:** nenhuma — read-only via variáveis de shell, sem tool calls.

## Arquitetura original (tentativas 1 e 2) — descontinuada

```
[bash curator-teste1.sh]
        │
        ├── sorteia daily aleatória de wiki/diario/
        ├── embute index.md + daily no prompt (sem tool calls)
        ├── claude -p → curadoria estruturada
        └── envia 2 mensagens ao Telegram Geral
              msg 1: conteúdo completo da daily  ← era responsabilidade do agente
              msg 2: bloco de curadoria
```

**Motivo da mudança:** não faz sentido passar a daily pelo agente para entregá-la ao Telegram. A daily é selecionada pelo script, que já tem o conteúdo — o envio deve acontecer diretamente, sem intermediário.

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

**Nota:** os fixes identificados para a tentativa 3 estão documentados na seção seguinte.

**Validação adicional — formato de saída (2026-06-27T14:02-03:00):**

Antes de alterar o script, foi validado o formato de entrega da daily ao Telegram. Três formatos testados:

| Formato | Método | Resultado |
|---|---|---|
| Texto puro | `sendMessage` sem parse_mode | ✅ chegou, mas markdown visível em texto puro (##, **, etc.) |
| HTML | `sendMessage` com `parse_mode: HTML` | ✅ chegou, mas formatação parcial e inconsistente |
| Rich Message | `sendRichMessage` com `rich_message.markdown` | ✅ chegou com formatação nativa completa — aprovado |

O `sendRichMessage` (Bot API 10.1) aceita markdown raw e renderiza nativamente no Telegram. Payload correto:
```json
{
  "chat_id": -1003870518428,
  "rich_message": {"markdown": "conteúdo markdown aqui"}
}
```

O frontmatter YAML (`---`) é removido antes do envio — não faz parte do conteúdo legível.

Documentação completa do endpoint: [[infraestrutura/telegram-send-rich-message.md]]

### Tentativa 3 — 2026-06-27 — SUCESSO ✅

**Primeira execução bem-sucedida. Pipeline completo funcionou como configurado.**

**Feedback do Giovani:** "foi a primeira que funcionou e chegou o output como configurado."

**Fixes aplicados (acumulados das tentativas 1 e 2 + novo insight):**

| # | Fix | Origem |
|---|---|---|
| 1 | `nohup` redireciona para `.nohup.log` separado; `$LOG` é exclusivo do `log()` | Bug tentativa 2: traceback sobrescrevia log |
| 2 | `message_thread_id` removido do payload do Telegram | Bug tentativa 2: `400 message thread not found` |
| 3 | Python captura `e.read().decode()` no `except HTTPError` | Diagnóstico: erro 400 sem corpo não ajuda |
| 4 | Daily enviada via `sendRichMessage` diretamente pelo script, antes do agente | Insight: agente não deve ser intermediário da daily |
| 5 | Curadoria (msg 2) também enviada via `sendRichMessage` | Validação: formato aprovado na sessão 2026-06-27 |
| 6 | Frontmatter YAML removido antes do envio da daily | Regra: `---` não é markdown válido para renderização |

**Forma de chamada:**
```bash
bash /root/curator-teste1.sh
```

Rodado direto da sessão Claude Code (sem `nohup`) — script terminou dentro do timeout de 360s configurado na ferramenta Bash.

**O que aconteceu:**

| Etapa | Resultado |
|---|---|
| Sessão Claude Code derrubada | ✅ Não aconteceu |
| Daily sorteada | aleatória — via `shuf -n1` |
| Daily enviada via `sendRichMessage` | ✅ message_id: 800 |
| `claude -p` executou e retornou curadoria | ✅ sem timeout, sem vazio |
| Curadoria enviada via `sendRichMessage` | ✅ message_id: 801 |
| Mensagens chegaram ao Telegram Geral | ✅ confirmado pelo Giovani |

**O que esta tentativa validou:**
- Pipeline completo: sorteia daily → envia daily como rich message → agente gera curadoria → envia curadoria como rich message
- Separação correta de responsabilidades: script entrega a daily, agente entrega só a análise
- Logs sem colisão entre `nohup` e `log()`
- Formatação rich message aprovada pelo Giovani em ambas as mensagens
- `claude -p` com timeout 300 e prompt grande (index.md + daily completa) funciona dentro do limite

**Próximos ajustes (a definir com o Giovani):** pipeline validado — próxima fase é avaliar a qualidade da curadoria e decidir ajustes no prompt ou na seleção de dailies.

## Regra de fluxo obrigatória para o agente

Quando o script falha ou apresenta erro:

1. **Documentar** o erro na wiki (aqui, nesta seção de histórico)
2. **Perguntar** ao Giovani se quer corrigir, mudar abordagem ou descartar
3. **Só então** aplicar qualquer alteração — e apenas o que for autorizado

**Nunca editar o script (ou qualquer arquivo validado) sem autorização explícita.** Esta regra vale mesmo quando a correção parece óbvia.

---

### Tentativa 4 — 2026-06-27 — FALHA PARCIAL

**Script:** `curator-teste2.sh` (v2)
**Mudança central:** agente deixa de ser one-shot cego e passa a ter acesso de leitura à wiki.

**O problema da tentativa 3:** o agente só via o `index.md` (títulos e uma linha por página). Sem ler o conteúdo real das páginas, ele não conseguia saber com certeza se um item já estava documentado — decidia com base em títulos, não em conteúdo.

**Solução:** `--allowedTools "Read"` — o agente lê as páginas que julgar relevantes antes de decidir.

#### Arquitetura v2

```
[bash curator-teste2.sh]
        │
        ├── sorteia daily aleatória de wiki/diario/
        ├── envia daily via sendRichMessage (script, sem agente)
        ├── chama claude com:
        │     -s /root/curator-v2-system.md   ← system prompt separado
        │     --allowedTools "Read"            ← leitura da wiki habilitada
        │     -p "<index.md + daily + formato>"
        ├── agente lê pages relevantes via Read tool
        ├── agente produz curadoria
        └── envia curadoria via sendRichMessage
```

#### Separação de responsabilidades

| Camada | Conteúdo | Por quê |
|---|---|---|
| System prompt | Expertise de curadoria + caminho da wiki | Estável — define o raciocínio do agente |
| User prompt | index.md + daily + formato de output | Dinâmico — muda a cada execução |
| Tools (Read) | Conteúdo real das páginas wiki | Sob demanda — agente decide o que precisa ler |

#### System prompt (`/root/curator-v2-system.md`)

~750 tokens. Não resume nem reescreve a documentação — ensina o raciocínio para casos onde o agente precisa julgar.

```
Você é o Curador da Wiki — agente especializado nesta wiki específica,
com acesso de leitura a todos os arquivos em /root/wiki/.
Seu trabalho é analisar daily notes e produzir recomendações precisas
sobre o que fazer com cada item encontrado.

## Esta wiki e o padrão que ela segue

Esta wiki segue o padrão LLM Wiki de Karpathy: uma base de conhecimento
composta e persistente, mantida por agentes de IA. O princípio central
não é acumular — é compor. A wiki deve ficar mais densa e útil com o
tempo, não apenas maior.

O diário (wiki/diario/) é a inbox do sistema: captura bruta de sessões,
descobertas, pendências e decisões. É o ponto de partida, não o destino.
Seu trabalho é processar esse material e identificar o que merece sair
do diário e entrar na base permanente.

## A distinção central: durável vs volátil

Esta é a decisão mais difícil. Use o critério: "daqui a 6 meses,
alguém vai precisar saber disso?"

**Durável** — fatos técnicos, decisões de arquitetura, causas raiz
identificadas, procedimentos validados, regras de comportamento do
sistema. Exemplos desta wiki:

- "sendRichMessage não precisa de message_thread_id para o tópico Geral"
- "timeout do Bash tool do Claude Code é ~2 minutos"
- "deploy key deve ser individual por dispositivo no Obsidian Git"

**Volátil** — contexto de execução, estados transitórios, tentativas
sem conclusão, o que estava rodando naquele momento. Exemplos:

- "tentei enviar com thread_id=1 e deu 400" → volátil
- "processo estava rodando há 2min18s quando verificado" → volátil
- "aguardando resposta do Giovani" → volátil

Atenção: uma tentativa que falhou pode ser durável se gerou uma causa
raiz documentada. O que importa não é o sucesso — é se o aprendizado
se aplica além daquela sessão.

## Tipos de página (OKF)

**concept** — descreve o que algo É ou como algo FUNCIONA. Sistemas,
componentes, padrões. Ex: o que é o Hermes, como funciona o sendRichMessage.

**procedure** — passo a passo executável com começo, meio e fim.
Alguém pode seguir para realizar uma tarefa.

**session** — o incidente em si é o conhecimento. Uma crise, uma
migração, uma descoberta importante. O valor está no registro do
que aconteceu e do que foi aprendido.

**todo** — pendências ativas. Coisas que ainda vão acontecer.

**raw** — fontes externas imutáveis. Nunca sugerir edição.

## Como decidir

**Antes de qualquer decisão, leia as páginas relevantes.** O index.md
no prompt dá o mapa. Os arquivos completos estão em /root/wiki/.
Não decida com base só no título de uma página — leia o conteúdo real.

**Migrar para página existente** quando o item complementa, corrige
ou atualiza algo já documentado. Diga exatamente o que adicionar
e em qual seção.

**Criar página nova** quando o item tem identidade própria, não existe
nada na wiki que o cubra mesmo parcialmente, e tem substância
suficiente para ser consultado de forma independente. Defina tipo
OKF, pasta e nome do arquivo sugerido. Você pode sugerir nova pasta
se nenhuma existente encaixar.

**Descartar** quando é volátil, duplicata ou ruído. Um descarte
bem justificado é tão valioso quanto uma boa sugestão de criação.

Casos difíceis: se dois fragmentos da daily juntos justificam uma
página que nenhum deles justificaria sozinho, sugira a consolidação
em CRIAR com essa observação.

## Regra absoluta

Você é read-only. Só sugere — nunca executa alterações na wiki.
```

#### Script (`/root/curator-teste2.sh`)

```bash
#!/bin/bash
# curator-teste2.sh — Curador da Wiki: v2 / Tentativa 4
# Melhorias vs v1: tools (Read), system prompt separado, tickets, footer com métricas
# Uso: bash /root/curator-teste2.sh

set -euo pipefail

WIKI_DIR="/root/wiki"
WIKI_DIARIO="$WIKI_DIR/wiki/diario"
WIKI_INDEX="$WIKI_DIR/index.md"
TELEGRAM_BOT_TOKEN="..."
CHAT_ID="-1003870518428"
LOG="/var/log/curator-teste2.log"
TICKET_COUNT_FILE="/var/log/curator-ticket.count"
OUTPUT_DIR="/var/log/curator-outputs"
SYSTEM_PROMPT="/root/curator-v2-system.md"

log() { echo "[$(TZ='America/Sao_Paulo' date '+%Y-%m-%dT%H:%M:%S-03:00')] $*" >> "$LOG"; }

log "Script iniciado (PID $$)."

mkdir -p "$OUTPUT_DIR"

# Ticket progressivo
if [[ -f "$TICKET_COUNT_FILE" ]]; then
    TICKET=$(( $(cat "$TICKET_COUNT_FILE") + 1 ))
else
    TICKET=1
fi
echo "$TICKET" > "$TICKET_COUNT_FILE"
TICKET_FMT=$(printf "%03d" "$TICKET")

log "Ticket: #$TICKET_FMT"

# Envia markdown via sendRichMessage, removendo frontmatter e quebrando em chunks de 32768 chars
send_rich() {
    local text="$1"
    python3 - "$TELEGRAM_BOT_TOKEN" "$CHAT_ID" "$text" <<'PYEOF'
import sys, json, urllib.request, urllib.error, re

token, chat_id, text = sys.argv[1], sys.argv[2], sys.argv[3]

text = re.sub(r'^---\n.*?\n---\n', '', text, flags=re.DOTALL).strip()

chunks = [text[i:i+32768] for i in range(0, max(len(text), 1), 32768)]

for chunk in chunks:
    payload = json.dumps({
        "chat_id": int(chat_id),
        "rich_message": {"markdown": chunk}
    }).encode()
    req = urllib.request.Request(
        f"https://api.telegram.org/bot{token}/sendRichMessage",
        data=payload,
        headers={"Content-Type": "application/json"}
    )
    try:
        resp = urllib.request.urlopen(req)
        print(json.loads(resp.read().decode())["result"]["message_id"], flush=True)
    except urllib.error.HTTPError as e:
        print(f"ERRO {e.code}: {e.read().decode()}", flush=True)
        sys.exit(1)
PYEOF
}

# 1. Escolher daily aleatória
DAILY_PATH=$(ls "$WIKI_DIARIO"/*.md | shuf -n1)
DAILY_NAME=$(basename "$DAILY_PATH")
DAILY_CONTENT=$(cat "$DAILY_PATH")
INDEX_CONTENT=$(cat "$WIKI_INDEX")
TOTAL_DAILIES=$(ls "$WIKI_DIARIO"/*.md | wc -l | tr -d ' ')

log "Daily sorteada: $DAILY_NAME | Total no diário: $TOTAL_DAILIES"

# Estimativa de tokens injetados (index + daily)
INDEX_BYTES=$(wc -c < "$WIKI_INDEX")
DAILY_BYTES=$(wc -c < "$DAILY_PATH")
INJECT_TOKENS=$(( (INDEX_BYTES + DAILY_BYTES) / 4 ))

# 2. Enviar daily ao Telegram
MSG_DAILY="📅 **CURADOR DA WIKI — #$TICKET_FMT**
**Daily sorteada:** $DAILY_NAME

📄 **CONTEÚDO COMPLETO:**

$DAILY_CONTENT"

send_rich "$MSG_DAILY"
log "Daily enviada ao Telegram."

# 3. Rodar curador com tools + system prompt separado
START_TIME=$SECONDS

CLAUDE_JSON=$(timeout 600 claude \
    -s "$SYSTEM_PROMPT" \
    --allowedTools "Read" \
    --output-format json \
    -p "## Mapa da wiki

$INDEX_CONTENT

## Daily a analisar

Arquivo: $DAILY_NAME

$DAILY_CONTENT

---

Leia as páginas que precisar antes de decidir. Quando tiver contexto
suficiente, produza APENAS o bloco abaixo, sem texto fora dele.

🔍 CURADORIA — $DAILY_NAME

**MIGRAR PARA ARQUIVO EXISTENTE:**
- [item] → [[wiki/pasta/arquivo.md]] — o que adicionar e onde

**CRIAR PÁGINA NOVA:**
- [conceito] → wiki/[pasta]/[arquivo].md | type: [tipo] — motivo

**DESCARTAR:**
- [item] — motivo") || true

DURATION=$(( SECONDS - START_TIME ))

if [[ -z "$CLAUDE_JSON" ]]; then
    log "ERRO: claude retornou vazio ou timeout (600s). Abortando."
    exit 1
fi

# Extrair curadoria e tokens do JSON
CURADORIA=$(echo "$CLAUDE_JSON" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('result',''))")
INPUT_TOKENS=$(echo "$CLAUDE_JSON" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('usage',{}).get('input_tokens','?'))")
OUTPUT_TOKENS=$(echo "$CLAUDE_JSON" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('usage',{}).get('output_tokens','?'))")

if [[ -z "$CURADORIA" ]]; then
    log "ERRO: campo 'result' vazio no JSON. Abortando."
    exit 1
fi

log "Curadoria concluída. Tokens: ${INPUT_TOKENS}in / ${OUTPUT_TOKENS}out. Duração: ${DURATION}s."

# 4. Salvar output em arquivo local (para consulta futura por agentes)
OUTPUT_FILE="$OUTPUT_DIR/ticket-$TICKET_FMT.md"
cat > "$OUTPUT_FILE" <<EOF
# Ticket #$TICKET_FMT — $DAILY_NAME
Data: $(TZ='America/Sao_Paulo' date '+%Y-%m-%dT%H:%M:%S-03:00')

$CURADORIA

---
Tokens injetados: ~$INJECT_TOKENS | Input: $INPUT_TOKENS | Output: $OUTPUT_TOKENS | Duração: ${DURATION}s
EOF

log "Output salvo em $OUTPUT_FILE"

# 5. Enviar curadoria ao Telegram
send_rich "$CURADORIA"

# 6. Enviar footer com métricas
FOOTER="📊 **TICKET #$TICKET_FMT** — $DAILY_NAME
Dailies no diário: $TOTAL_DAILIES arquivos
Tokens injetados: ~$INJECT_TOKENS
Tokens totais: ${INPUT_TOKENS} input / ${OUTPUT_TOKENS} output
Duração: ${DURATION}s"

send_rich "$FOOTER"

log "Enviado com sucesso."
```

#### Formato de output (3 seções fixas)

```
🔍 CURADORIA — {DAILY_NAME}

**MIGRAR PARA ARQUIVO EXISTENTE:**
- [item] → [[wiki/pasta/arquivo.md]] — o que adicionar e onde

**CRIAR PÁGINA NOVA:**
- [conceito] → wiki/[pasta]/[arquivo].md | type: [tipo] — motivo

**DESCARTAR:**
- [item] — motivo
```

Sugestões estruturais (nova pasta, mudança de status, consolidações) entram dentro das seções relevantes — CRIAR ou MIGRAR — sem seção separada.

#### Sistema de tickets

Cada execução gera um número de ticket progressivo começando em `#001`.

**Armazenamento do contador:** `/var/log/curator-ticket.count` — arquivo com um inteiro, incrementado a cada run.

**Por que ticket é útil:** permite referenciar um output específico em qualquer ambiente ou sessão futura. "Me fala sobre o ticket 5" é suficiente para um agente localizar o arquivo.

**Persistência do output:** o output de curadoria é salvo em dois lugares:
1. **Telegram** — entrega em tempo real para o Giovani
2. **Arquivo local** — `/var/log/curator-outputs/ticket-NNN.md` — registro permanente consultável por qualquer agente via `Read`

O ticket aparece no header da daily enviada ao Telegram e no nome do arquivo salvo.

#### Footer de execução

Enviado ao Telegram após o output de curadoria, em mensagem separada:

```
📊 TICKET #NNN — {DAILY_NAME}
Dailies no diário:  XX arquivos
Tokens injetados:   ~XXXX  (index.md + daily)
Tokens totais:      XXXX input / XXXX output
Duração:            Xs
```

- **Dailies no diário:** `ls wiki/diario/*.md | wc -l` — total de arquivos no momento da execução
- **Tokens injetados:** estimativa por contagem de caracteres (`len / 4`)
- **Tokens totais:** via `--output-format json` no `claude -p`, que retorna `usage.input_tokens` e `usage.output_tokens`
- **Duração:** `$SECONDS` do bash, calculado entre início e fim da chamada ao agente

#### O que esta tentativa valida

_(planejado — não validado ainda devido à falha)_

#### O que aconteceu — 2026-06-27

| Etapa | Resultado |
|---|---|
| Daily enviada ao Telegram | ✅ message_id: 802 |
| `claude -p` executou | ❌ `error: unknown option '-s'` |
| Curadoria gerada | ❌ não chegou |
| Mensagem de curadoria enviada | ❌ não enviada |

**Causa raiz:** a flag `-s` usada no script para passar o system prompt não existe nesta versão do `claude` CLI. O comando correto é `--system-prompt-file` (ou `--system-prompt` para texto inline). O script foi projetado com base em uma flag que não existe.

**Erro exato:**
```
error: unknown option '-s'
```

**Nota:** o Claude Code (agente) editou o script sem autorização ao ver o erro, depois reverteu a alteração ao ser alertado. Esse comportamento é incorreto — ver **Regra de fluxo obrigatória** acima.

**Fix aplicado (autorizado pelo Giovani):**
- Flag `-s` substituída por `--system-prompt-file` no `curator-teste2.sh`
- Tentativa 4 re-executada após o fix: pipeline completo ✅ (message_ids 803, 804, 805)

**Erro de comportamento identificado no output:** o agente leu a daily analisada e sugeriu editar/migrar conteúdo para outra daily — o que é errado. Dailies são imutáveis e temporárias.

**Fix aplicado no system prompt (`curator-v2-system.md`):**
- Adicionada seção "Natureza das daily notes" — explica que dailies são temporárias, serão apagadas pelo usuário, e não devem ser destino de nenhuma sugestão
- Adicionada seção "Pastas proibidas para leitura" — proíbe explicitamente ler `wiki/diario/` e `wiki/historico/`; define as pastas permanentes como destino exclusivo de MIGRAR
- Reforçado que a daily a analisar já está no prompt — o agente não precisa ler outras dailies

**Próximo passo:** rodar tentativa 5 para validar se o comportamento foi corrigido.

---

### Tentativa 5 — 2026-06-27 — aguardando execução

**Script:** `curator-teste2.sh` (v2 — mesmo script da tentativa 4)
**Mudança central:** correção de comportamento no system prompt — agente proibido de ler `diario/` e `historico/`, e instruído sobre a natureza temporária das dailies.

**Problema identificado na tentativa 4:** o agente leu a daily sorteada e sugeriu migrar conteúdo para outra daily — destino proibido. Dailies são imutáveis, temporárias e serão apagadas pelo usuário após revisão. O agente não deve ler outras dailies nem sugerir dailies como destino de migração.

**Fixes aplicados no `curator-v2-system.md` antes desta tentativa:**

| # | Fix | Motivo |
|---|---|---|
| 1 | Seção "Natureza das daily notes" adicionada | Agente não entendia que dailies são temporárias e serão apagadas |
| 2 | Seção "Pastas proibidas para leitura" adicionada | Proibição explícita de ler `wiki/diario/` e `wiki/historico/` |
| 3 | Destino de MIGRAR definido explicitamente | Só páginas permanentes: `automacao/`, `conhecimento/`, `infraestrutura/`, `pendencias/` |
| 4 | Instrução: a daily já está no prompt | Agente não precisa ler outras dailies — o conteúdo a analisar já é fornecido |

**O que esta tentativa valida:**
- Agente respeita a proibição de ler `diario/` e `historico/`
- Agente nunca sugere outra daily como destino de MIGRAR
- Agente entende o papel da daily como inbox temporária
- System prompt com restrições explícitas de pasta é suficiente (sem whitelist técnica no `--allowedTools`)

**Forma de execução:**
```bash
bash /root/curator-teste2.sh
```

**Resultado:** SUCESSO ✅ — pipeline completo, message_ids 806 (daily), 807 (curadoria), 808 (footer).

**O que foi validado:**
- Agente não leu `diario/` nem `historico/`
- Agente não sugeriu outra daily como destino de MIGRAR
- Proibições explícitas no system prompt foram respeitadas

**Problemas identificados no output para a próxima iteração:**
- Numeração reinicia em cada bloco (1 no MIGRAR, 1 no CRIAR, 1 no DESCARTAR) — deveria ser contínua
- Sem tags de prioridade nos itens
- Formato pouco legível — falta estrutura visual por item

---

### Tentativa 6 — 2026-06-27 — aguardando execução

**Script:** `curator-teste3.sh` (v3 — novo script; v2 preservado como baseline estável da tentativa 5)
**System prompt:** `curator-v3-system.md` (v3 — inclui seção de prioridade)
**Mudanças:** formato de output redesenhado — numeração contínua, tags de prioridade, estrutura por item mais legível.

**Problemas da tentativa 5 a corrigir:**

| # | Problema | Fix |
|---|---|---|
| 1 | Numeração reinicia em cada bloco | Numeração contínua através dos 3 blocos |
| 2 | Sem indicação de prioridade | Tag `[ALTA]`, `[MÉDIA]` ou `[BAIXA]` em cada item |
| 3 | Formato compacto dificulta leitura | Estrutura expandida por item com linhas `→` separadas |

**Novo formato de output:**

```
🔍 CURADORIA — {DAILY_NAME}

**MIGRAR PARA ARQUIVO EXISTENTE:**
1. [ALTA] [nome do conceito/item]
→ inserir em [[wiki/pasta/arquivo.md]]
→ [explicação detalhada de onde inserir e por quê]

2. [MÉDIA] [nome do conceito/item]
→ inserir em [[wiki/pasta/arquivo.md]]
→ [explicação detalhada]

**CRIAR PÁGINA NOVA:**
3. [ALTA] [nome do conceito]
→ wiki/pasta/arquivo.md | type: [tipo]
→ [motivo e o que a página deve conter]

**DESCARTAR:**
4. [BAIXA] [item]
→ [motivo do descarte]
```

**Critério de prioridade:**
- `[ALTA]` — conhecimento durável que pode ser perdido se não registrado agora; afeta decisões futuras
- `[MÉDIA]` — útil, mas existe em outro lugar ou pode ser reconstruído
- `[BAIXA]` — complementar, contexto de apoio, baixo risco de perda

**Onde foram aplicados os fixes:**
- Formato de output no `-p` do `curator-teste3.sh`
- Critério de prioridade no `curator-v3-system.md`

**O que esta tentativa valida:**
- Numeração contínua funciona com o modelo
- Tags de prioridade são aplicadas de forma consistente e útil
- Novo formato é mais legível para o Giovani

**Resultado:** SUCESSO ✅ — pipeline completo, message_ids 809 (daily), 810 (curadoria), 811 (footer).

**O que foi validado:**
- Numeração contínua através dos 3 blocos funcionou
- Tags de prioridade `[ALTA]`, `[MÉDIA]`, `[BAIXA]` aplicadas pelo modelo
- Novo formato expandido por item entregue corretamente
- Pipeline idêntico ao da tentativa 5 — sem regressões

---

### Tentativa 7 — 2026-06-27 — SUCESSO PARCIAL ⚠️

**Script:** `curator-teste3.sh` (v3 — mesmo da tentativa 6)
**Ticket gerado:** #005 — daily `2026-06-22-20260622.md`
**Message-ids:** 813 (daily), 814 (curadoria), 815 (footer)

**Problemas identificados no output via revisão manual com o Giovani:**

| # | Problema | Detalhe |
|---|---|---|
| 1 | Destino errado sem verificação | Agente sugeriu `telegram-topicos.md` para conteúdo sobre reações sem ler a página — ela cobre IDs de tópicos, não Bot API |
| 2 | Conteúdo não validado tratado como fato | Técnica ±5 de IDs vizinhos era aspiração do wiki-review, não procedimento testado — agente sugeriu migrar como se fosse durável |
| 3 | Não declarou o que leu | Sem rastreabilidade das decisões — impossível saber se o agente realmente verificou as páginas antes de escolher destinos |

**Aprendizados para próxima iteração:**
1. Agente DEVE ler a página de destino antes de sugerir MIGRAR — não decidir por nome de arquivo
2. Cada item deve declarar explicitamente o que foi lido e o que foi encontrado (linha `↳`)
3. Footer deve incluir lista completa de todos os arquivos lidos durante a análise

---

### Tentativa 8 — aguardando execução

**Script:** `curator-teste4.sh` (v4 — novo script)
**System prompt:** `curator-v4-system.md` (v4 — inclui regra de leitura obrigatória antes de MIGRAR)
**Mudanças centrais:**

| # | Mudança | Onde |
|---|---|---|
| 1 | Regra explícita: ler página antes de sugerir MIGRAR | `curator-v4-system.md` |
| 2 | Linha `↳` por item: arquivos lidos + o que encontrou | prompt `-p` em `curator-teste4.sh` |
| 3 | `LIDOS:` ao final do output do agente | prompt `-p` em `curator-teste4.sh` |
| 4 | Footer inclui lista completa de arquivos lidos | `curator-teste4.sh` — extrai `LIDOS:` e adiciona ao footer |

**O que esta tentativa valida:**
- Agente declara rastreabilidade de cada decisão via linha `↳`
- Destinos de MIGRAR são verificados antes de sugeridos
- Footer com arquivos lidos permite auditoria rápida da análise

**Resultado:** SUCESSO ✅ — ticket #014, daily `2026-06-22-reacoes-telegram.md`, message-ids 835 (daily), 836 (curadoria), 837 (footer). Avaliação do output pendente.

---

### Tentativa 9 — 2026-06-28 — SUCESSO ✅

**Script:** `curator-teste5.sh` (v5 — novo script; v4 preservado como baseline estável)
**System prompt:** `curator-v4-system.md` (mesmo do v4 — sem alteração)
**Mudança central:** script aceita nome de daily como argumento opcional.

**Problema da tentativa 8:** com poucas dailies no diário, a seleção aleatória repetia arquivos já processados. Não havia como escolher qual daily injetar sem editar o script.

**Fix aplicado em `curator-teste5.sh`:**

```bash
# 1. Escolher daily (argumento ou aleatória)
if [[ -n "${1:-}" ]]; then
    DAILY_PATH="$WIKI_DIARIO/$1"
    if [[ ! -f "$DAILY_PATH" ]]; then
        log "ERRO: daily não encontrada: $DAILY_PATH"
        echo "ERRO: daily não encontrada: $DAILY_PATH" >&2
        exit 1
    fi
    MODO="escolhida"
else
    DAILY_PATH=$(ls "$WIKI_DIARIO"/*.md | shuf -n1)
    MODO="sorteada"
fi
```

**Uso:**
```bash
# daily específica
bash /root/curator-teste5.sh 2026-06-20.md

# aleatória (comportamento anterior)
bash /root/curator-teste5.sh
```

**O que esta tentativa valida:**
- Argumento opcional funciona sem quebrar o comportamento padrão
- Validação de arquivo inexistente com erro claro no log

| Etapa | Resultado |
|---|---|
| Daily escolhida | `2026-06-20.md` (argumento explícito) |
| Daily enviada ao Telegram | ✅ message_id: 838 |
| `claude -p` executou e retornou curadoria | ✅ |
| Curadoria enviada via `sendRichMessage` | ✅ message_id: 839 |
| Footer enviado | ✅ message_id: 840 |
| Ticket gerado | #015 |

**Avaliação do output:** pendente.

---

### Tentativa 10 — 2026-06-28 — aguardando execução

**Script:** `curator-teste6.sh` (v6 — novo script; v5 preservado como baseline estável)
**System prompt:** `curator-v5-system.md` (v5 — redesenho estrutural)
**Mudanças centrais:** output em tabelas por pasta (vs 3 blocos de prosa); pastas proibidas sobem para o topo do prompt; três seções de julgamento fundidas em uma; prioridade restrita a MIGRAR e CRIAR; template de formato removido do `-p` e centralizado no system prompt.

#### System prompt (`/root/curator-v5-system.md`)

```
Você é o Curador Sênior da Wiki — agente especializado nesta wiki específica,
com acesso de leitura a todos os arquivos em /root/wiki/.
Seu trabalho é analisar daily notes e produzir análise estruturada em tabelas,
organizadas por pasta da wiki, com raciocínio explícito por item.

## Pastas proibidas

É estritamente proibido ler arquivos em:

- `wiki/diario/` — o conteúdo da daily a analisar já chega no prompt
- `wiki/historico/` — registros históricos imutáveis, não são base de decisão para curadoria

Dailies são **imutáveis** — nunca sugira editar, complementar ou migrar conteúdo para outra daily.
O destino de qualquer MIGRAR é sempre uma página permanente: `automacao/`, `conhecimento/`,
`infraestrutura/`, `pendencias/` — jamais `diario/` ou `historico/`.

Você tem acesso livre a toda a wiki permanente: `automacao/`, `conhecimento/`,
`infraestrutura/`, `pendencias/`, e ao `index.md`.

## Esta wiki e o padrão que ela segue

Esta wiki segue o padrão LLM Wiki de Karpathy: uma base de conhecimento
composta e persistente, mantida por agentes de IA. O princípio central
não é acumular — é compor. Cada página deve ser densa, interligada e
consultável de forma independente. A wiki cresce em profundidade, não
em volume: conceitos se conectam via wikilinks, formando um grafo de
conhecimento que fica mais útil à medida que é editado, não apenas
adicionado.

O diário é a inbox do sistema: captura bruta de sessões, descobertas,
pendências e decisões. É o ponto de partida, não o destino. Seu trabalho
é processar esse material e identificar o que merece sair do diário e
entrar na base permanente.

## Julgamento

**Critério central:** "daqui a 6 meses, um agente novo vai precisar disso?"

**Durável** — fato técnico, decisão de arquitetura, causa raiz, procedimento
validado, regra de comportamento. Uma tentativa que falhou pode ser durável
se gerou aprendizado aplicável além daquela sessão.

**Volátil** — estado transitório, contexto de execução, tentativas sem conclusão.
Exemplos: "tentei enviar com thread_id=1 e deu 400", "aguardando resposta do Giovani".

Antes de qualquer decisão, leia as páginas relevantes pelo index.md. Nunca decida só pelo título.

**MIGRAR** quando o item complementa algo já documentado. Leia a página de
destino antes de decidir — nunca escolha pelo nome do arquivo.

**CRIAR** quando o item tem identidade própria e substância para ser consultado
de forma independente. Defina type OKF, pasta e nome. Pode sugerir pasta nova.

**DESCARTAR** quando é volátil, duplicata ou ruído. Um descarte bem justificado
é tão valioso quanto uma boa sugestão de criação.

Caso difícil: dois fragmentos que juntos justificam uma página que nenhum
justificaria sozinho → CRIAR com essa observação.

Itens MIGRAR e CRIAR recebem nível de prioridade:
**Alta** — durável, risco real de perda, afeta decisões futuras. Registrar agora.
**Média** — útil, mas parcialmente coberto ou reconstruível com esforço.
**Baixa** — complementar, apoio, baixo valor a longo prazo.

## Tipos de página (OKF)

**concept** — descreve o que algo É ou como algo FUNCIONA. Sistemas,
componentes, padrões. Ex: o que é o Hermes, como funciona o sendRichMessage.

**procedure** — passo a passo executável com começo, meio e fim.
Alguém pode seguir para realizar uma tarefa.

**session** — o incidente em si é o conhecimento. Uma crise, uma
migração, uma descoberta importante. O valor está no registro do
que aconteceu e do que foi aprendido.

**todo** — pendências ativas. Coisas que ainda vão acontecer.

**raw** — fontes externas imutáveis. Nunca sugerir edição.

## Formato de output obrigatório

Produza APENAS tabelas Markdown — sem texto introdutório, sem conclusão,
sem explicações fora das tabelas. Apenas as seções abaixo.

Para cada pasta da wiki que tiver ao menos um item positivo (MIGRAR ou CRIAR),
produza uma seção. Inclua apenas as pastas com itens. Ordene os itens dentro
de cada tabela do mais para o menos importante.

### [pasta]/

| Nº | Tópico | Já existe na wiki? | Injetar? | Por que | Onde exatamente + conexões |
|---|---|---|---|---|---|
| N | Descrição detalhada do que é o conteúdo — não apenas um título, o suficiente para entender o item sem ler a daily | Sim / Não / Parcialmente — e o que existe | Sim — editar `arquivo.md` / Sim — criar `arquivo.md` | Por que isso vale daqui a 6 meses | Seção exata onde inserir + [[wikilinks]] para páginas relacionadas |

Pastas disponíveis: automacao/, conhecimento/, infraestrutura/, pendencias/
Se nenhuma servir, sugira o nome da nova pasta.

Ao final dos itens positivos, tabela de descartes:

### DESCARTAR

| Nº | Tópico | Por que descartar |
|---|---|---|
| N | Descrição do item | Motivo objetivo (volátil, duplicata, ruído) |

Numeração Nº contínua através de TODAS as tabelas (não reinicia por seção).

Última linha do output, obrigatória:
↳ LIDOS: [[arquivo1]], [[arquivo2]], ...

## Regra absoluta

Você é read-only. Só sugere — nunca executa alterações na wiki.
```

#### Script (`/root/curator-teste6.sh`)

```bash
#!/bin/bash
# curator-teste6.sh — Curador da Wiki: v6 / Tentativa 10+
# Melhorias vs v5: output em tabelas por pasta (curator-v5-system.md)
# Uso: bash /root/curator-teste6.sh [nome-da-daily.md]
#   Sem argumento → daily aleatória
#   Com argumento → usa a daily especificada (ex: 2026-06-20.md)

set -euo pipefail

WIKI_DIR="/root/wiki"
WIKI_DIARIO="$WIKI_DIR/wiki/diario"
WIKI_INDEX="$WIKI_DIR/index.md"
TELEGRAM_BOT_TOKEN="8712644255:AAFEOBZOBNBLSB05YVT9rbG2lhNF-83nehc"
CHAT_ID="-1003870518428"
LOG="/var/log/curator-teste6.log"
TICKET_COUNT_FILE="/var/log/curator-ticket.count"
OUTPUT_DIR="/var/log/curator-outputs"
SYSTEM_PROMPT="/root/curator-v5-system.md"

log() { echo "[$(TZ='America/Sao_Paulo' date '+%Y-%m-%dT%H:%M:%S-03:00')] $*" >> "$LOG"; }

log "Script iniciado (PID $$)."

mkdir -p "$OUTPUT_DIR"

# Ticket progressivo
if [[ -f "$TICKET_COUNT_FILE" ]]; then
    TICKET=$(( $(cat "$TICKET_COUNT_FILE") + 1 ))
else
    TICKET=1
fi
echo "$TICKET" > "$TICKET_COUNT_FILE"
TICKET_FMT=$(printf "%03d" "$TICKET")

log "Ticket: #$TICKET_FMT"

# Envia markdown via sendRichMessage, removendo frontmatter e quebrando em chunks de 32768 chars
send_rich() {
    local text="$1"
    python3 - "$TELEGRAM_BOT_TOKEN" "$CHAT_ID" "$text" <<'PYEOF'
import sys, json, urllib.request, urllib.error, re

token, chat_id, text = sys.argv[1], sys.argv[2], sys.argv[3]

text = re.sub(r'^---\n.*?\n---\n', '', text, flags=re.DOTALL).strip()

chunks = [text[i:i+32768] for i in range(0, max(len(text), 1), 32768)]

for chunk in chunks:
    payload = json.dumps({
        "chat_id": int(chat_id),
        "rich_message": {"markdown": chunk}
    }).encode()
    req = urllib.request.Request(
        f"https://api.telegram.org/bot{token}/sendRichMessage",
        data=payload,
        headers={"Content-Type": "application/json"}
    )
    try:
        resp = urllib.request.urlopen(req)
        print(json.loads(resp.read().decode())["result"]["message_id"], flush=True)
    except urllib.error.HTTPError as e:
        print(f"ERRO {e.code}: {e.read().decode()}", flush=True)
        sys.exit(1)
PYEOF
}

# 1. Escolher daily (argumento ou aleatória)
if [[ -n "${1:-}" ]]; then
    DAILY_PATH="$WIKI_DIARIO/$1"
    if [[ ! -f "$DAILY_PATH" ]]; then
        log "ERRO: daily não encontrada: $DAILY_PATH"
        echo "ERRO: daily não encontrada: $DAILY_PATH" >&2
        exit 1
    fi
    MODO="escolhida"
else
    DAILY_PATH=$(ls "$WIKI_DIARIO"/*.md | shuf -n1)
    MODO="sorteada"
fi

DAILY_NAME=$(basename "$DAILY_PATH")
DAILY_CONTENT=$(cat "$DAILY_PATH")
INDEX_CONTENT=$(cat "$WIKI_INDEX")
TOTAL_DAILIES=$(ls "$WIKI_DIARIO"/*.md | wc -l | tr -d ' ')

log "Daily $MODO: $DAILY_NAME | Total no diário: $TOTAL_DAILIES"

# Estimativa de tokens injetados (index + daily)
INDEX_BYTES=$(wc -c < "$WIKI_INDEX")
DAILY_BYTES=$(wc -c < "$DAILY_PATH")
INJECT_TOKENS=$(( (INDEX_BYTES + DAILY_BYTES) / 4 ))

# 2. Enviar daily ao Telegram
MSG_DAILY="📅 **CURADOR DA WIKI — #$TICKET_FMT**
**Daily $MODO:** $DAILY_NAME

📄 **CONTEÚDO COMPLETO:**

$DAILY_CONTENT"

send_rich "$MSG_DAILY"
log "Daily enviada ao Telegram."

# 3. Rodar curador com tools + system prompt separado
START_TIME=$SECONDS

CLAUDE_JSON=$(timeout 600 claude \
    --system-prompt-file "$SYSTEM_PROMPT" \
    --allowedTools "Read" \
    --output-format json \
    -p "## Mapa da wiki

$INDEX_CONTENT

## Daily a analisar

Arquivo: $DAILY_NAME

$DAILY_CONTENT

---

Leia as páginas que precisar antes de decidir. Produza a curadoria no formato especificado.") || true

DURATION=$(( SECONDS - START_TIME ))

if [[ -z "$CLAUDE_JSON" ]]; then
    log "ERRO: claude retornou vazio ou timeout (600s). Abortando."
    exit 1
fi

# Extrair curadoria e tokens do JSON
CURADORIA_RAW=$(echo "$CLAUDE_JSON" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('result',''))")
INPUT_TOKENS=$(echo "$CLAUDE_JSON" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('usage',{}).get('input_tokens','?'))")
OUTPUT_TOKENS=$(echo "$CLAUDE_JSON" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('usage',{}).get('output_tokens','?'))")

if [[ -z "$CURADORIA_RAW" ]]; then
    log "ERRO: campo 'result' vazio no JSON. Abortando."
    exit 1
fi

# Extrair linha LIDOS: e separar do corpo da curadoria
LIDOS=$(echo "$CURADORIA_RAW" | python3 -c "
import sys, re
text = sys.stdin.read()
m = re.search(r'^↳ LIDOS:(.+)$', text, re.MULTILINE)
print(m.group(1).strip() if m else '(não informado)')
")

CURADORIA=$(echo "$CURADORIA_RAW" | python3 -c "
import sys, re
text = sys.stdin.read()
text = re.sub(r'\n↳ LIDOS:.*$', '', text, flags=re.MULTILINE).strip()
print(text)
")

log "Curadoria concluída. Tokens: ${INPUT_TOKENS}in / ${OUTPUT_TOKENS}out. Duração: ${DURATION}s."

# 4. Salvar output em arquivo local
OUTPUT_FILE="$OUTPUT_DIR/ticket-$TICKET_FMT.md"
cat > "$OUTPUT_FILE" <<EOF
# Ticket #$TICKET_FMT — $DAILY_NAME
Data: $(TZ='America/Sao_Paulo' date '+%Y-%m-%dT%H:%M:%S-03:00')

$CURADORIA_RAW

---
Tokens injetados: ~$INJECT_TOKENS | Input: $INPUT_TOKENS | Output: $OUTPUT_TOKENS | Duração: ${DURATION}s
EOF

log "Output salvo em $OUTPUT_FILE"

# 5. Enviar curadoria ao Telegram
send_rich "$CURADORIA"

# 6. Enviar footer com métricas + arquivos lidos
FOOTER="📊 **TICKET #$TICKET_FMT** — $DAILY_NAME
Dailies no diário: $TOTAL_DAILIES arquivos
Tokens injetados: ~$INJECT_TOKENS
Tokens totais: ${INPUT_TOKENS} input / ${OUTPUT_TOKENS} output
Duração: ${DURATION}s

📚 **Arquivos lidos:** $LIDOS"

send_rich "$FOOTER"

log "Enviado com sucesso."
```

**O que esta tentativa valida:**
- Agente produz tabelas por pasta sem template no `-p`
- Prioridade ausente na tabela DESCARTAR
- `↳ LIDOS:` extraído corretamente para o footer

---

## Pendências de curadoria (estado em 2026-06-28)

Duas dailies ainda sem ticket:

- `wiki/diario/2026-06-22-reacoes-telegram.md`
- `wiki/diario/2026-06-23-20260623.md`

**Comando para processar as duas com intervalo de 5 minutos:**
```bash
bash /root/curator-teste4.sh && sleep 300 && bash /root/curator-teste4.sh
```

Rodar do terminal ou de uma sessão Claude Code. Após cada execução, avaliar a curadoria no Telegram e aplicar/descartar conforme revisão do Giovani.

---

## Conexões

- [[wiki/conhecimento/plano-implementacao-loop.md|Plano de Implementação — Loops]] — contexto técnico e arquitetura de loops agênticos
- [[wiki/infraestrutura/telegram-topicos.md|Telegram Tópicos]] — IDs do Telegram usados no envio
- [[wiki/automacao/wiki-review.md|Wiki Review]] — outro agente de automação da wiki
