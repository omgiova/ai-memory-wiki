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
| Geral | 1 | `telegram:-1003870518428:1` |
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

## Como enviar para cada destino — comandos validados

### Geral (via curl / Python urllib direto na VPS)

Validado em 2026-06-27 com 4 testes (message_ids 793–796). Enviar para `chat_id` **sem** `message_thread_id`:

```bash
curl -s -X POST "https://api.telegram.org/bot${TOKEN}/sendMessage" \
  -H "Content-Type: application/json" \
  -d "{\"chat_id\":\"-1003870518428\",\"text\":\"mensagem aqui\"}"
```

### Outros tópicos (Substack, Skills, wiki_review)

Usar `chat_id` + `message_thread_id` com o ID correspondente da tabela acima:

```bash
curl -s -X POST "https://api.telegram.org/bot${TOKEN}/sendMessage" \
  -H "Content-Type: application/json" \
  -d "{\"chat_id\":\"-1003870518428\",\"message_thread_id\":112,\"text\":\"mensagem aqui\"}"
```

---

## Observações por contexto

### Geral (thread_id=1) — comportamento via Python urllib / curl direto na VPS

**Contexto:** curator-teste1 (`/root/curator-teste1.sh`), 2026-06-27, chamada via `python3` com `urllib.request` diretamente na VPS, sem passar pelo Hermes gateway.

**Comportamento observado:** ao incluir `"message_thread_id": 1` no payload JSON, a API retornou `HTTP 400: Bad Request: message thread not found`. Ao omitir o campo `message_thread_id`, a mensagem foi entregue ao Geral com sucesso (message_ids 793–796 confirmados via 5 testes curl).

**O que isso não invalida:** o `thread_id=1` documentado acima é correto e validado em outros ambientes (ex: Hermes gateway). A diferença pode estar no método de envio, autenticação, ou configuração do cliente. Requer investigação futura para entender por que o comportamento difere neste contexto específico.

## Conexões

- [[infraestrutura/hermes.md]] — stack e configuração do Hermes
- [[automacao/wiki-review.md]] — plugin que escreve no diário e notifica o tópico wiki_review
