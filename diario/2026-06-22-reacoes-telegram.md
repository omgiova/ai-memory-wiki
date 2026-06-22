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

---

## Preferência de nomenclatura do diário — 00:15

O usuário pediu para renomear o arquivo de `2026-06-22.md` para `2026-06-22-reacoes-telegram.md`. Motivo: ele cria **múltiplas daily notes por dia**, então o padrão `YYYY-MM-DD.md` não serve. O padrão correto é `YYYY-MM-DD-sufixo-descritivo.md`.

**Ação tomada:** renomeei via `git mv` e commitei. Salvei na memory() do Hermes para não esquecer.

**Insight do usuário:** Giovani é detalhista com organização. Não aceita soluções genéricas quando a realidade é mais granular. Preferir perguntar o padrão antes de criar arquivos.

---

## Estilo de trabalho do Giovani — 00:20

Dois momentos marcantes nesta sessão revelam o estilo esperado:

1. **"era pra fazer com um comando só"** — quando o Hermes gastou múltiplas tentativas para encontrar o message_id, Giovani reclamou da complexidade. Ele espera eficiência: uma tentativa, resultado. Se não der, perguntar direto em vez de explorar por 10 turnos.

2. **"vc já tem o token do bot, reaja a essa mensagem minha"** — Giovani assume que o Hermes já tem tudo que precisa. Não quer fornecer contexto extra se a ferramenta deveria saber. Expectativa: o agente resolve com o que tem, ou explica em uma frase o que falta.

**Regra derivada:** quando uma ação parece simples mas esbarra em falta de dado (ex: message_id), perguntar ao usuário imediatamente em vez de investigar por múltiplos turnos. A pesquisa profunda só vale quando o dado realmente não existe no contexto.

---

## Descoberta técnica: HERMES_SESSION_MESSAGE_ID — 00:25

Investigando o código-fonte do gateway (`gateway/session_context.py`), descobri que o Hermes **já injeta o message_id** do Telegram no contexto da sessão via variável de contexto `HERMES_SESSION_MESSAGE_ID`.

**Resultado do teste:**
```
MESSAGE_ID: 527
CHAT_ID: -1003870518428
THREAD_ID: 1
```

**Acesso via Python:**
```python
from gateway.session_context import get_session_env
msg_id = get_session_env('HERMES_SESSION_MESSAGE_ID', '')
```

**Ressalva:** o valor retornado foi `527`, mas o link que o usuário mandou era `537`. Possíveis explicações:
- A variável pode refletir a mensagem anterior, não a atual
- Ou há timing issue entre o gateway e o agente

**Status:** funcional mas precisa de validação. Se confirmado que sempre reflete a mensagem atual, o Hermes pode reagir a qualquer mensagem sem pedir o link ao usuário.

**Correção na wiki:** a seção "Como obter o message_id" diz que "o message_id não está disponível no contexto do agente" — isso está parcialmente errado. A variável existe, mas precisa ser validada.

---

## Próximos passos — 00:30

1. **Validar `HERMES_SESSION_MESSAGE_ID`** — testar se o valor da variável corresponde ao message_id real da mensagem atual. Se sim, criar uma skill ou snippet para reagir automaticamente.

2. **Ativar `telegram.reactions: true`** — Giovani é admin do grupo e quer testar se o Hermes vê reações. Pendente desde o início da sessão.

3. **Criar skill `telegram-reactions`** — encapsular o curl do setMessageReaction + acesso à variável de contexto em uma skill reutilizável.

4. **Corrigir wiki** — atualizar a seção "Como obter o message_id" para mencionar `HERMES_SESSION_MESSAGE_ID` como fonte primária.
