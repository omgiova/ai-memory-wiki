---
type: concept
tags: [curador, automacao, diario, curadoria, wiki]
title: Curador da Wiki
description: Agente read-only que analisa daily notes e produz recomendações de curadoria estruturadas, entregues ao Telegram Geral.
timestamp: 2026-06-28T00:00:00-03:00
status: stable
---

# Curador da Wiki

Agente especializado que processa daily notes e recomenda ao Giovani o que fazer com cada item: migrar para página existente, criar página nova ou descartar.

**Papel no ecossistema da wiki:** o diário é a inbox — captura bruta de sessões. O curador é o filtro: identifica o que tem valor permanente e propõe como integrá-lo à base.

## Comportamento

- **Read-only** — só sugere, nunca executa alterações na wiki
- **Dailies são temporárias** — nunca sugere outra daily como destino; só páginas permanentes (`automacao/`, `conhecimento/`, `infraestrutura/`, `pendencias/`)
- **Lê antes de decidir** — lê as páginas de destino a partir do `index.md` antes de sugerir MIGRAR; nunca decide por nome de arquivo
- **Rastreabilidade** — declara `↳ LIDOS:` ao final de cada análise com todos os arquivos consultados

## Critério de curadoria

Pergunta central: **"daqui a 6 meses, um agente novo vai precisar disso?"**

- **Migrar** — complementa algo já documentado; exige leitura da página de destino antes de decidir
- **Criar** — identidade própria, substância para consulta independente; define type OKF, pasta e nome do arquivo
- **Descartar** — volátil, duplicata ou ruído; descarte bem justificado vale tanto quanto uma boa sugestão

## Arquitetura atual (v6 / prompt v5)

```
[bash curator-teste6.sh]
        │
        ├── recebe daily como argumento (ou sorteia aleatória)
        ├── envia daily ao Telegram via sendRichMessage (script direto)
        ├── chama claude com:
        │     --system-prompt-file /root/curator-v5-system.md
        │     --allowedTools "Read"
        │     --output-format json
        │     -p "<index.md + daily>"
        ├── agente lê páginas relevantes e produz tabelas por pasta
        └── envia curadoria + footer de métricas ao Telegram
```

## Como executar

```bash
# daily específica
bash /root/curator-teste6.sh 2026-06-23-20260623.md

# aleatória
bash /root/curator-teste6.sh
```

Log: `/var/log/curator-teste6.log`

## Formato de output

Tabelas Markdown organizadas por pasta da wiki. Para cada pasta com ao menos um item positivo:

```
### automacao/

| Nº | Tópico | Já existe na wiki? | Injetar? | Por que | Onde exatamente + conexões |
|---|---|---|---|---|---|
| 1  | ...    | Sim/Não/Parcialmente | Sim — editar arquivo.md | Por que vale a longo prazo | Seção + [[wikilinks]] |

### DESCARTAR

| Nº | Tópico | Por que descartar |
|---|---|---|
| 2  | ...    | Motivo objetivo |

↳ LIDOS: [[arquivo1]], [[arquivo2]], ...
```

Numeração contínua através de todas as tabelas.

## Sistema de tickets

Cada execução gera um ticket progressivo. Contador em `/var/log/curator-ticket.count`.

Outputs locais salvos em `/var/log/curator-outputs/ticket-NNN.md` — consultáveis por agentes via `Read`.

## Destino: Telegram Geral

- **Chat ID:** `-1003870518428`
- **Thread ID:** omitido (Geral = chat principal do fórum; sem `message_thread_id`)
- **Formato:** `sendRichMessage` com `rich_message.markdown`
- **Chunking:** 32.768 chars por mensagem

Cada execução envia 3 mensagens: daily completa → curadoria → footer de métricas.

## Arquivos do agente

| Arquivo | Papel |
|---|---|
| `/root/curator-teste6.sh` | Script principal (v6) |
| `/root/curator-v5-system.md` | System prompt do agente (v5) |
| `/var/log/curator-teste6.log` | Log de execução |
| `/var/log/curator-ticket.count` | Contador de tickets |
| `/var/log/curator-outputs/` | Outputs locais por ticket |

## Conexões

- [[wiki/automacao/curador-wiki-historico.md|Histórico de desenvolvimento]] — todas as tentativas, scripts e decisões de design
- [[wiki/infraestrutura/telegram-send-rich-message.md|Telegram sendRichMessage]] — endpoint usado para entrega
- [[wiki/infraestrutura/telegram-topicos.md|Telegram Tópicos]] — IDs do Telegram
- [[wiki/conhecimento/plano-implementacao-loop.md|Plano de Loops]] — contexto técnico do pipeline agêntico
