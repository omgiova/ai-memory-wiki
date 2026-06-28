---
type: concept
tags: [telegram, infraestrutura, bot-api, topicos, rich-message, reacoes]
title: Telegram
description: Referência completa de tudo sobre Telegram neste setup — IDs, tópicos, HERMES_SESSION_MESSAGE_ID, envio de mensagens, reações e integrações com o Hermes
timestamp: 2026-06-28T00:00:00-03:00
status: draft
---

# Telegram

## 1. Chats e IDs

Supergrupo com fórum (`is_forum=true`): `-1003870518428`

### Tópicos Confirmados

| Nome | thread_id | Target |
|---|---|---|
| Geral | 1 | `telegram:-1003870518428:1` |
| Substack | 112 | `telegram:-1003870518428:112` |
| Skills | 282 | `telegram:-1003870518428:282` |
| wiki_review | 749 | `telegram:-1003870518428:749` |

### DM

Giovani (principal): `142422888` — conversa privada com o bot.

### Como usar

```
telegram:<chat_id>:<thread_id>
```

Omitir `:thread_id` para o tópico padrão (Geral). Omitir tudo para DM (home). *(informação a validar)*

---

## 2. HERMES_SESSION_MESSAGE_ID

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

> ⚠️ **Não validado de forma confiável.** Todos os testes realizados até 2026-06-27 não confirmaram funcionamento consistente. Os métodos de acesso acima são os mais promissores identificados, mas nenhum foi verificado como funcional em uso real. Ver [[pendencias/proximos-passos.md]] e a seção 4 (Reações) abaixo.

---

## 3. Envio de Mensagens

### 3.1 sendMessage (básico)

#### Como enviar para cada destino — comandos validados

##### Geral (via curl / Python urllib direto na VPS)

Validado em 2026-06-27 com 4 testes (message_ids 793–796). Enviar para `chat_id` **sem** `message_thread_id`:

```bash
curl -s -X POST "https://api.telegram.org/bot${TOKEN}/sendMessage" \
  -H "Content-Type: application/json" \
  -d "{\"chat_id\":\"-1003870518428\",\"text\":\"mensagem aqui\"}"
```

##### Outros tópicos (Substack, Skills, wiki_review)

Usar `chat_id` + `message_thread_id` com o ID correspondente da tabela acima:

```bash
curl -s -X POST "https://api.telegram.org/bot${TOKEN}/sendMessage" \
  -H "Content-Type: application/json" \
  -d "{\"chat_id\":\"-1003870518428\",\"message_thread_id\":112,\"text\":\"mensagem aqui\"}"
```

#### Observações por contexto

##### Geral (thread_id=1) — comportamento via Python urllib / curl direto na VPS

**Contexto:** curator-teste1 (`/root/curator-teste1.sh`), 2026-06-27, chamada via `python3` com `urllib.request` diretamente na VPS, sem passar pelo Hermes gateway.

**Comportamento observado:** ao incluir `"message_thread_id": 1` no payload JSON, a API retornou `HTTP 400: Bad Request: message thread not found`. Ao omitir o campo `message_thread_id`, a mensagem foi entregue ao Geral com sucesso (message_ids 793–796 confirmados via 5 testes curl).

**O que isso não invalida:** o `thread_id=1` documentado acima é correto e validado em outros ambientes (ex: Hermes gateway). A diferença pode estar no método de envio, autenticação, ou configuração do cliente. Requer investigação futura para entender por que o comportamento difere neste contexto específico.

---

### 3.2 sendRichMessage (Bot API 10.1)

#### Endpoint

```
POST https://api.telegram.org/bot{TOKEN}/sendRichMessage
```

#### Payload

```json
{
  "chat_id": -1003870518428,
  "rich_message": {
    "markdown": "conteúdo em markdown raw aqui"
  }
}
```

#### Parâmetros obrigatórios

| Campo | Tipo | Descrição |
|---|---|---|
| `chat_id` | integer | ID do chat de destino |
| `rich_message` | objeto | Objeto `InputRichMessage` |
| `rich_message.markdown` | string | Conteúdo em markdown raw |

#### Parâmetros opcionais

| Campo | Tipo | Descrição |
|---|---|---|
| `message_thread_id` | integer | ID do tópico (forum). Omitir para enviar ao Geral |
| `reply_parameters` | objeto | `{"message_id": N}` para responder a uma mensagem. **Não usar** `reply_to_message_id` — é ignorado silenciosamente |
| `rich_message.skip_entity_detection` | boolean | Pula detecção de entidades no conteúdo |

#### Limites

| Limite | Valor |
|---|---|
| Tamanho máximo do conteúdo | 32.768 caracteres |
| Encoding | UTF-8 |

#### Comportamento do markdown

O campo `rich_message.markdown` aceita **markdown raw** — não MarkdownV2, não HTML. Exemplos de sintaxe renderizada nativamente:

| Sintaxe | Renderização |
|---|---|
| `## Título` | Cabeçalho |
| `**texto**` | Negrito |
| `*texto*` | Itálico |
| `` `código` `` | Código inline |
| ` ```bloco``` ` | Bloco de código |
| `\| col1 \| col2 \|` | Tabela |
| `- [ ] item` | Task list |
| `<details>` | Seção colapsável |

Blocos de matemática também são suportados.

#### Erros conhecidos

| Código | Descrição | Causa |
|---|---|---|
| 400 | `rich message must be non-empty` | Campo `rich_message` ausente ou `markdown` vazio |
| 400 | `message thread not found` | `message_thread_id` inválido para o grupo |
| 400 | `chat not found` | `chat_id` inválido |

#### Exemplo funcional validado

Validado em 2026-06-27 (message_id 799, grupo `-1003870518428`):

```python
import json, urllib.request

TOKEN = "..."
CHAT  = "-1003870518428"

payload = json.dumps({
    "chat_id": int(CHAT),
    "rich_message": {"markdown": "## Título\n\n**Negrito** e *itálico*\n\n- item 1\n- item 2"}
}).encode()

req = urllib.request.Request(
    f"https://api.telegram.org/bot{TOKEN}/sendRichMessage",
    data=payload,
    headers={"Content-Type": "application/json"}
)
resp = urllib.request.urlopen(req)
r = json.loads(resp.read().decode())
print(r["result"]["message_id"])
```

#### Regras de uso

- O conteúdo deve ser passado em `rich_message.markdown`, **não** em `text`
- Passar `text` em vez de `rich_message` resulta em erro 400 `rich message must be non-empty`
- Frontmatter YAML (`---`) deve ser removido antes do envio — não é markdown válido para renderização
- Para o tópico Geral do grupo `-1003870518428`, omitir `message_thread_id` (ver seção 1 — Chats e IDs)
- Para outros tópicos, incluir `message_thread_id` com o ID correspondente
- Conteúdo acima de 32.768 caracteres deve ser dividido em múltiplas mensagens

#### Referência interna

- Implementação no Hermes: `/root/.hermes/plugins/platforms/telegram/adapter.py`, método `_rich_message_payload` (linha 1218) e `_send_rich_message` (linha 1322)
- Habilitação no Hermes: `gateway.platforms.telegram.extra.rich_messages: true` em `config.yaml`

---

## 4. Reações (setMessageReaction)

### Endpoint

```
POST https://api.telegram.org/bot{TOKEN}/setMessageReaction
```

### Procedimento

#### 1. Obter o ID da mensagem

```bash
source ~/.hermes/.env
echo "HERMES_SESSION_MESSAGE_ID: $HERMES_SESSION_MESSAGE_ID"
echo "HERMES_SESSION_CHAT_ID: $HERMES_SESSION_CHAT_ID"
```

> ⚠️ **HERMES_SESSION_MESSAGE_ID não está validado de forma confiável** — todos os testes até 2026-06-27 não confirmaram funcionamento consistente. Se a variável retornar vazio ou ID incorreto, o passo 2 detectará o problema. Ver seção 2 (HERMES_SESSION_MESSAGE_ID) acima.

#### 2. Validar com reação benigna (CRÍTICO)

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

#### 3. Aplicar a reação desejada

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

### Parâmetros do payload

| Campo | Tipo | Descrição |
|---|---|---|
| `chat_id` | integer | ID do chat |
| `message_id` | integer | ID da mensagem a reagir |
| `reaction` | array | Lista de objetos `{"type": "emoji", "emoji": "🎯"}` |
| `is_big` | boolean | (opcional) Reação grande — efeito visual diferente |

### Emojis válidos

O Telegram aceita uma lista fechada de ~70 emojis para reações via bot. Usar emoji fora da lista retorna `REACTION_INVALID`.

❤️ 👍 👎 🔥 🥰 👏 😁 🤔 🤯 😱 🤬 😢 🎉 🤩 🤮 💩 🙏 👌 🕊 🤡 🥱 🥴 😍 🐳 ❤‍🔥 🌚 🌭 💯 🤣 ⚡ 🍌 🏆 💔 🤨 😐 🍓 🍾 💋 🖕 😈 😴 😭 🤓 👻 👨‍💻 👀 🎃 🙈 😇 😨 🤝 ✍ 🤗 🫡 🎅 🎄 ☃ 💅 🤪 🗿 🆒 💘 🙉 🦄 😘 💊 🙊 😎 👾 🤷‍♂ 🤷 🤷‍♀ 😡

> ⚠️ **🐙 polvo NÃO está na lista.** Testado em 2026-06-21 — API retorna `REACTION_INVALID`.

### Status de validação

Este procedimento foi identificado e documentado mas **não foi validado de ponta a ponta** até 2026-06-27. O gargalo atual é a confiabilidade do `HERMES_SESSION_MESSAGE_ID` — sem um ID correto, o `setMessageReaction` falha. Os testes realizados não confirmaram que a variável retorna o ID certo de forma consistente.

O padrão test-then-act (validar com 👍 antes da reação real) é o protocolo de segurança para qualquer operação de reação.

---

## 5. Integrações com o Hermes

| Página | Como usa o Telegram |
|---|---|
| [[infraestrutura/hermes.md]] | Gateway principal — Telegram é o canal de entrada do Hermes |
| [[infraestrutura/hermes-api.md]] | Endpoints REST de messaging (`/api/messaging/telegram/`) |
| [[automacao/curador-wiki.md]] | Entrega curadorias e dailies ao Geral via `sendRichMessage` |
| [[automacao/wiki-review.md]] | Notifica o tópico `wiki_review` após cada revisão |

---

## Conexões

- [[infraestrutura/hermes.md]]
- [[infraestrutura/hermes-api.md]]
- [[automacao/curador-wiki.md]]
- [[automacao/wiki-review.md]]
- [[pendencias/proximos-passos.md]]
