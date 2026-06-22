---
type: daily
tags: [telegram, reacoes, bot-api, setmessagereaction, hermes]
title: Reações no Telegram via Bot API
description: Como usar setMessageReaction para reagir a mensagens e como receber reações de usuários
timestamp: 2026-06-22T00:10:00+00:00
status: stable
---

# Reações no Telegram via Bot API

## Descoberta

O Hermes pode **enviar reações** em mensagens do Telegram via `setMessageReaction` usando curl direto na API. Também pode **receber reações** de usuários se configurado corretamente.

## Como enviar uma reação

### Endpoint

```
POST https://api.telegram.org/bot<TOKEN>/setMessageReaction
```

### Parâmetros

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|:-----------:|-----------|
| `chat_id` | Integer ou String | ✅ | ID do chat ou @username |
| `message_id` | Integer | ✅ | ID da mensagem alvo |
| `reaction` | Array de `ReactionType` | ❌ | Lista de reações (bots: **no máximo 1**) |
| `is_big` | Boolean | ❌ | `true` = animação grande |

### Exemplo com curl

```bash
source ~/.hermes/.env && curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/setMessageReaction" \
  -H "Content-Type: application/json" \
  -d '{
    "chat_id": -1003870518428,
    "message_id": 537,
    "reaction": [{"type": "emoji", "emoji": "🔥"}],
    "is_big": false
  }'
```

### Como obter o message_id

O link do Telegram `https://t.me/c/3870518428/1/537` revela:
- `3870518428` → chat_id = `-1003870518428` (prefixo `-100`)
- `1` → thread/topic_id
- `537` → **message_id**

⚠️ O gateway do Hermes consome os updates via long polling, então `getUpdates` retorna vazio. O message_id **não** está disponível no contexto do agente — precisa vir do link da mensagem ou de outra fonte.

## Emojis suportados como reação

O Telegram tem ~70 emojis disponíveis para reações. Lista completa:

❤️ 👍 👎 🔥 🥰 👏 😁 🤔 🤯 😱 🤬 😢 🎉 🤩 🤮 💩 🙏 👌 🕊 🤡 🥱 🥴 😍 🐳 ❤‍🔥 🌚 🌭 💯 🤣 ⚡ 🍌 🏆 💔 🤨 😐 🍓 🍾 💋 🖕 😈 😴 😭 🤓 👻 👨‍💻 👀 🎃 🙈 😇 😨 🤝 ✍ 🤗 🫡 🎅 🎄 ☃ 💅 🤪 🗿 🆒 💘 🙉 🦄 😘 💊 🙊 😎 👾 🤷‍♂ 🤷 🤷‍♀ 😡

**🐙 polvo NÃO está na lista.** A API retorna `REACTION_INVALID` se usar emoji fora da lista.

## Como receber reações de usuários

Para o Hermes **ver** quando alguém reage a uma mensagem:

1. **Bot precisa ser admin** do grupo
2. Config `telegram.reactions` precisa estar `true` (hoje está `false`)
3. O gateway precisa registrar `"message_reaction"` nos `allowed_updates`

O update chega como `MessageReactionUpdated`:

| Campo | Tipo | Info |
|-------|------|------|
| `chat` | Chat | O chat da mensagem |
| `message_id` | Integer | Qual mensagem |
| `user` | User | Quem reagiu (se não anônimo) |
| `old_reaction` | ReactionType[] | Reações anteriores |
| `new_reaction` | ReactionType[] | Reações novas |

⚠️ Reações de bots **não** disparam esse evento.

## Pendências

- [ ] Ativar `telegram.reactions: true` no config.yaml
- [ ] Testar recebimento de reações (Giovani reage → Hermes vê)
- [ ] Verificar se o gateway precisa de restart após mudar config

## Conexões

- [[infraestrutura/hermes.md|Hermes Config]] — config do gateway Telegram
- [[diario/2026-06-20.md|Daily 2026-06-20]] — sessão anterior
