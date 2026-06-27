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

## Conexões

- [[wiki/conhecimento/plano-implementacao-loop.md|Plano de Implementação — Loops]] — contexto técnico e arquitetura de loops agênticos
- [[wiki/infraestrutura/telegram-topicos.md|Telegram Tópicos]] — IDs do Telegram usados no envio
- [[wiki/automacao/wiki-review.md|Wiki Review]] — outro agente de automação da wiki
