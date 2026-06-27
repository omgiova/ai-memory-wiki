---
type: concept
tags: [telegram, infraestrutura, bot-api, hub]
title: Telegram — Hub Central
description: Ponto de entrada único para tudo relacionado ao Telegram nesta wiki — Bot API, tópicos, envio de mensagens, reações e integrações com o Hermes
timestamp: 2026-06-27T17:00:00-03:00
status: draft
---

# Telegram — Hub Central

Referência central para tudo sobre o Telegram neste setup. Navegue pelas páginas específicas ou consulte as seções abaixo para comportamentos transversais.

## Páginas específicas

| Página | O que cobre |
|---|---|
| [[infraestrutura/telegram-topicos.md]] | IDs de chats e tópicos do grupo HERMES & GIONATO |
| [[infraestrutura/telegram-send-rich-message.md]] | Endpoint `sendRichMessage` (Bot API 10.1) — parâmetros, payload, limites |
| [[infraestrutura/telegram-reacoes.md]] | Endpoint `setMessageReaction` — procedimento e status de validação |

## Integrações com o Hermes

| Página | Como usa o Telegram |
|---|---|
| [[infraestrutura/hermes.md]] | Gateway principal — Telegram é o canal de entrada do Hermes |
| [[infraestrutura/hermes-api.md]] | Endpoints REST de messaging (`/api/messaging/telegram/`) |
| [[automacao/curador-wiki.md]] | Entrega curadorias e dailies ao Geral via `sendRichMessage` |
| [[automacao/wiki-review.md]] | Notifica o tópico `wiki_review` após cada revisão |

## HERMES_SESSION_MESSAGE_ID

O gateway injeta o `message_id` da mensagem que disparou o turno atual na variável de contexto `HERMES_SESSION_MESSAGE_ID`.

**Acesso via Python (código interno):**
```python
from gateway.session_context import get_session_env
msg_id = get_session_env("HERMES_SESSION_MESSAGE_ID")
chat_id = get_session_env("HERMES_SESSION_CHAT_ID")
```

**Acesso via bash (ambiente de sessão):**
```bash
source ~/.hermes/.env
echo "HERMES_SESSION_MESSAGE_ID: $HERMES_SESSION_MESSAGE_ID"
echo "HERMES_SESSION_CHAT_ID: $HERMES_SESSION_CHAT_ID"
```

**Mecanismo esperado:** o gateway consome updates via long polling e injeta `message_id`, `chat_id` e `thread_id` da mensagem que disparou o turno no contexto da sessão antes de repassar ao agente — mesmo que `getUpdates` retorne vazio para o agente.

> ⚠️ **Não validado de forma confiável.** Todos os testes realizados até 2026-06-27 não confirmaram funcionamento consistente. Os métodos de acesso acima são os mais promissores identificados, mas nenhum foi verificado como funcional em uso real. Ver [[pendencias/proximos-passos.md]] e [[infraestrutura/telegram-reacoes.md]].

## Conexões

- [[infraestrutura/hermes.md]]
- [[infraestrutura/telegram-topicos.md]]
- [[infraestrutura/telegram-send-rich-message.md]]
- [[infraestrutura/telegram-reacoes.md]]
