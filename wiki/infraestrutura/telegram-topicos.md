---
type: concept
tags: [telegram, infraestrutura, topicos]
title: Telegram — Tópicos do Grupo
description: Mapa completo de chats, tópicos e IDs do Telegram do Giovani para entrega de mensagens pelo Hermes
timestamp: 2026-06-24T17:09:56-03:00
status: stable
---

# Telegram — Tópicos do Grupo

Supergrupo com fórum (`is_forum=true`): `-1003870518428`

## Tópicos Confirmados

| Nome | thread_id | Target |
|---|---|---|
| Geral | *(omitir)* | `telegram:-1003870518428` — **não usar thread_id=1**, causa "message thread not found" |
| Substack | 112 | `telegram:-1003870518428:112` |
| Skills | 282 | `telegram:-1003870518428:282` |
| wiki_review | 749 | `telegram:-1003870518428:749` |

## DM

Giovani (principal): `142422888` — conversa privada com o bot.

## Como usar

```
telegram:<chat_id>:<thread_id>
```

Omitir `:thread_id` para o tópico padrão (Geral). Omitir tudo para DM (home).

## Conexões

- [[infraestrutura/hermes.md]] — stack e configuração do Hermes
- [[automacao/wiki-review.md]] — plugin que escreve no diário e notifica o tópico wiki_review
