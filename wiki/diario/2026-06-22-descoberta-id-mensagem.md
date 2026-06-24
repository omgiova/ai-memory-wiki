---
type: procedure
tags: [telegram, message-id, reacciones, hermes, bot-api]
title: Descoberta do ID da mensagem via HERMES_SESSION_MESSAGE_ID
description: Como descobrir o ID exato de uma mensagem do Telegram usando a variável de sessão do Hermes
timestamp: 2026-06-22T14:03:09Z
status: stable
---

# Descoberta: Como identificar o ID de qualquer mensagem do Telegram

## Problema
Antes desta descoberta, o agente estava supondo qual mensagem reagir baseado em adivinhação ou padrões incorretos, levando a erros onde reações eram aplicadas às mensagens erradas.

## Solução
O Hermes gateway já injeta o `message_id` da mensagem que disparou o turno atual no contexto da sessão via a variável de ambiente `HERMES_SESSION_MESSAGE_ID`.

## Procedimento passo a passo

### 1. Obter o ID da mensagem atual
```bash
source ~/.hermes/.env
echo "HERMES_SESSION_MESSAGE_ID: $HERMES_SESSION_MESSAGE_ID"
echo "HERMES_SESSION_CHAT_ID: $HERMES_SESSION_CHAT_ID"
```

### 2. Validar o ID antes de usar (CRÍTICO)
Sempre teste com uma reação benigna primeiro para confirmar que o ID está correto:
```bash
source ~/.hermes/.env
curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/setMessageReaction" \\
  -H "Content-Type: application/json" \\
  -d '{
    "chat_id": '$HERMES_SESSION_CHAT_ID',
    "message_id": '$HERMES_SESSION_MESSAGE_ID',
    "reaction": [{"type": "emoji", "emoji": "👍"}]
  }'
```
Se o retorno for `{"ok":true,"result":true}`, o ID está válido.

### 3. Aplicar a reação desejada
Após validação, aplique a reação que o usuário solicitou:
```bash
source ~/.hermes/.env
curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/setMessageReaction" \\
  -H "Content-Type: application/json" \\
  -d '{
    "chat_id": '$HERMES_SESSION_CHAT_ID',
    "message_id": '$HERMES_SESSION_MESSAGE_ID',
    "reaction": [{"type": "emoji", "emoji": "SEU_EMOJI_AQUI"}]
  }'
```

## Por que isso funciona
- O Hermes gateway consome updates do Telegram via long polling
- Apesar de `getUpdates` retornar vazio para o agente, o gateway já processa a mensagem
- O gateway injeta o `message_id`, `chat_id` e `thread_id` da mensagem que disparou o turno no contexto da sessão
- Essas variáveis estão disponíveis como variáveis de ambiente após `source ~/.hermes/.env`

## Verificação
Depois de aplicar qualquer reação, sempre peça confirmação explícita do usuário:
"Reagi a tal mensagem com tal emoji. Essa é a mensagem que você queria ou errei?"

## Histórico desta descoberta
Descobrido em 2026-06-22 durante sessão de teste de reações, após múltiplos erros onde o agente estava reagindo às mensagens erradas devido a suposições incorretas sobre numeração de mensagens.

## Conexões
- [[infraestrutura/hermes.md|Hermes Config]]
- [[2026-06-22-reacoes-telegram.md|Reações no Telegram via Bot API]]
