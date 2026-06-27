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

## Integrações com o Hermes

| Página | Como usa o Telegram |
|---|---|
| [[infraestrutura/hermes.md]] | Gateway principal — Telegram é o canal de entrada do Hermes |
| [[infraestrutura/hermes-api.md]] | Endpoints REST de messaging (`/api/messaging/telegram/`) |
| [[automacao/curador-wiki.md]] | Entrega curadorias e dailies ao Geral via `sendRichMessage` |
| [[automacao/wiki-review.md]] | Notifica o tópico `wiki_review` após cada revisão |

## HERMES_SESSION_MESSAGE_ID — limitação de timing

> ⚠️ **Validação pendente** — comportamento observado em sessão mas não testado sistematicamente.

A variável de contexto `HERMES_SESSION_MESSAGE_ID` injeta o `message_id` da mensagem que disparou o turno atual. Porém, pode estar defasada: ela reflete a mensagem que *iniciou* o turno, não necessariamente a última mensagem do usuário. A latência entre o gateway e o agente pode causar defasagem — em testes de sessão, o ID observado estava errado por pelo menos uma mensagem.

**Acesso:**
```python
from gateway.session_context import get_session_env
msg_id = get_session_env("HERMES_SESSION_MESSAGE_ID")
chat_id = get_session_env("HERMES_SESSION_CHAT_ID")
```

**Recomendação atual:** não confiar no valor sem verificação prévia. Validar com uma reação benigna antes de usar.

## Conexões

- [[infraestrutura/hermes.md]]
- [[infraestrutura/telegram-topicos.md]]
- [[infraestrutura/telegram-send-rich-message.md]]
