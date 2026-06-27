---
type: procedure
tags: [telegram, bot-api, reacoes, set-message-reaction]
title: Telegram — Reações via setMessageReaction
description: Procedimento para aplicar reações a mensagens do Telegram usando setMessageReaction e HERMES_SESSION_MESSAGE_ID — status de validação incluído
timestamp: 2026-06-27T17:48:00-03:00
status: draft
---

# Telegram — Reações via setMessageReaction

## Endpoint

```
POST https://api.telegram.org/bot{TOKEN}/setMessageReaction
```

## Procedimento

### 1. Obter o ID da mensagem

```bash
source ~/.hermes/.env
echo "HERMES_SESSION_MESSAGE_ID: $HERMES_SESSION_MESSAGE_ID"
echo "HERMES_SESSION_CHAT_ID: $HERMES_SESSION_CHAT_ID"
```

> ⚠️ **HERMES_SESSION_MESSAGE_ID não está validado de forma confiável** — todos os testes até 2026-06-27 não confirmaram funcionamento consistente. Se a variável retornar vazio ou ID incorreto, o passo 2 detectará o problema. Ver [[infraestrutura/telegram-bot-api.md]].

### 2. Validar com reação benigna (CRÍTICO)

Antes de aplicar qualquer reação, validar que o ID está correto:

```bash
source ~/.hermes/.env
curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/setMessageReaction" \
  -H "Content-Type: application/json" \
  -d '{
    "chat_id": '$HERMES_SESSION_CHAT_ID',
    "message_id": '$HERMES_SESSION_MESSAGE_ID',
    "reaction": [{"type": "emoji", "emoji": "👍"}]
  }'
```

Resposta esperada: `{"ok":true,"result":true}`

### 3. Aplicar a reação desejada

```bash
source ~/.hermes/.env
curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/setMessageReaction" \
  -H "Content-Type: application/json" \
  -d '{
    "chat_id": '$HERMES_SESSION_CHAT_ID',
    "message_id": '$HERMES_SESSION_MESSAGE_ID',
    "reaction": [{"type": "emoji", "emoji": "SEU_EMOJI_AQUI"}]
  }'
```

Após aplicar, pedir confirmação explícita ao usuário: "Reagi a tal mensagem com tal emoji. Essa é a mensagem que você queria ou errei?"

## Parâmetros do payload

| Campo | Tipo | Descrição |
|---|---|---|
| `chat_id` | integer | ID do chat |
| `message_id` | integer | ID da mensagem a reagir |
| `reaction` | array | Lista de objetos `{"type": "emoji", "emoji": "🎯"}` |
| `is_big` | boolean | (opcional) Reação grande — efeito visual diferente |

## Status de validação

Este procedimento foi identificado e documentado mas **não foi validado de ponta a ponta** até 2026-06-27. O gargalo atual é a confiabilidade do `HERMES_SESSION_MESSAGE_ID` — sem um ID correto, o `setMessageReaction` falha. Os testes realizados não confirmaram que a variável retorna o ID certo de forma consistente.

O padrão test-then-act (validar com 👍 antes da reação real) é o protocolo de segurança para qualquer operação de reação.

## Conexões

- [[infraestrutura/telegram-bot-api.md|Telegram Hub]] — hub central; seção HERMES_SESSION_MESSAGE_ID com mecanismo de injeção e status
- [[infraestrutura/telegram-send-rich-message.md|sendRichMessage]] — endpoint de envio de mensagens (padrão similar de documentação)
