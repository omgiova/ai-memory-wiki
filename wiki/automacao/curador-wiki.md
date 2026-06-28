---
type: procedure
tags: [curador, automacao, diario, curadoria, wiki]
title: Curador da Wiki
description: Automação bash que processa uma daily note e entrega curadoria estruturada ao Telegram Geral via claude -p com acesso de leitura à wiki.
timestamp: 2026-06-28T00:00:00-03:00
status: stable
---

# Curador da Wiki

Automação bash que seleciona uma daily note, injeta o `index.md` e a daily em um `claude -p` com `--allowedTools "Read"`, e entrega três mensagens ao Telegram Geral: a daily completa, a curadoria em tabelas e um footer de métricas.

**Papel no ecossistema da wiki:** o diário é a inbox — captura bruta de sessões. O curador é o filtro: identifica o que tem valor permanente e propõe como integrá-lo à base.

## Como rodar

```bash
# daily específica
bash /root/curador-wiki-script-v1.sh nome-da-daily.md

# aleatória
bash /root/curador-wiki-script-v1.sh
```

Log de execução: `/var/log/curador-wiki.log`

## O que acontece em cada execução

1. Sorteia ou usa a daily passada como argumento
2. Envia a daily completa ao Telegram Geral via `sendRichMessage` (frontmatter removido)
3. Chama `claude --system-prompt-file /root/curador-wiki-prompt-v1.md --allowedTools "Read" --output-format json -p "<index.md + daily>"`
4. Extrai `result` e `usage` do JSON retornado
5. Salva output em `/var/log/curator-outputs/ticket-NNN.md`
6. Envia curadoria ao Telegram
7. Envia footer com métricas (tokens, duração, arquivos lidos)

## Resultado esperado

3 mensagens no Telegram Geral (`chat_id: -1003870518428`, sem `message_thread_id`):

**Msg 1 — Daily:**
```
📅 CURADOR DA WIKI — #NNN
Daily escolhida: 2026-06-XX.md

📄 CONTEÚDO COMPLETO:
[conteúdo sem frontmatter]
```

**Msg 2 — Curadoria (tabelas por pasta):**
```
### infraestrutura/

| Nº | Tópico | Já existe na wiki? | Injetar? | Por que | Onde exatamente + conexões |
|---|---|---|---|---|---|
| 1  | ...    | ...                | ...      | ...     | ...                        |

### DESCARTAR

| Nº | Tópico | Por que descartar |
|---|---|---|
| 2  | ...    | ...               |
```

Numeração contínua. Sem texto fora das tabelas.

**Msg 3 — Footer:**
```
📊 TICKET #NNN — nome-da-daily.md
Dailies no diário: X arquivos
Tokens injetados: ~XXXX
Tokens totais: XXXX input / XXXX output
Duração: Xs

📚 Arquivos lidos: [[arquivo1]], [[arquivo2]], ...
```

## Outputs locais

| Path | Conteúdo |
|---|---|
| `/var/log/curator-outputs/ticket-NNN.md` | Output completo de curadoria (inclui `↳ LIDOS:`) |
| `/var/log/curator-ticket.count` | Contador de tickets (inteiro, incrementado a cada run) |
| `/var/log/curador-wiki.log` | Log de execução com timestamps BRT — acumula todas as runs |

## Arquivos do agente

| Arquivo | Papel |
|---|---|
| `/root/curador-wiki-script-v1.sh` | Script principal — v1 validada (ex `curator-teste6.sh`) |
| `/root/curador-wiki-prompt-v1.md` | System prompt do agente curador — v1 validada (ex `curator-v5-system.md`) |

## Conexões

- [[wiki/automacao/curador-wiki-historico.md|Histórico de desenvolvimento]] — todas as tentativas, scripts completos e decisões de design
- [[wiki/infraestrutura/telegram.md|Telegram]] — endpoint sendRichMessage (seção 3.2) e IDs de tópicos (seção 1)
