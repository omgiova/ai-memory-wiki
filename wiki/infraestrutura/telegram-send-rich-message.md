---
type: concept
tags: [telegram, infraestrutura, bot-api, rich-message, formatacao]
title: Telegram — sendRichMessage (Bot API 10.1)
description: Documentação do endpoint sendRichMessage do Telegram Bot API 10.1 — parâmetros, payload, limites e comportamento
timestamp: 2026-06-27T14:10:00-03:00
status: stable
---

# Telegram — sendRichMessage (Bot API 10.1)

## Endpoint

```
POST https://api.telegram.org/bot{TOKEN}/sendRichMessage
```

## Payload

```json
{
  "chat_id": -1003870518428,
  "rich_message": {
    "markdown": "conteúdo em markdown raw aqui"
  }
}
```

### Parâmetros obrigatórios

| Campo | Tipo | Descrição |
|---|---|---|
| `chat_id` | integer | ID do chat de destino |
| `rich_message` | objeto | Objeto `InputRichMessage` |
| `rich_message.markdown` | string | Conteúdo em markdown raw |

### Parâmetros opcionais

| Campo | Tipo | Descrição |
|---|---|---|
| `message_thread_id` | integer | ID do tópico (forum). Omitir para enviar ao Geral |
| `reply_parameters` | objeto | `{"message_id": N}` para responder a uma mensagem. **Não usar** `reply_to_message_id` — é ignorado silenciosamente |
| `rich_message.skip_entity_detection` | boolean | Pula detecção de entidades no conteúdo |

## Limites

| Limite | Valor |
|---|---|
| Tamanho máximo do conteúdo | 32.768 caracteres |
| Encoding | UTF-8 |

## Comportamento do markdown

O campo `rich_message.markdown` aceita **markdown raw** — não MarkdownV2, não HTML. Exemplos de sintaxe renderizada nativamente:

| Sintaxe | Renderização |
|---|---|
| `## Título` | Cabeçalho |
| `**texto**` | Negrito |
| `*texto*` | Itálico |
| `` `código` `` | Código inline |
| ` ```bloco``` ` | Bloco de código |
| `| col1 | col2 |` | Tabela |
| `- [ ] item` | Task list |
| `<details>` | Seção colapsável |

Blocos de matemática também são suportados.

## Erros conhecidos

| Código | Descrição | Causa |
|---|---|---|
| 400 | `rich message must be non-empty` | Campo `rich_message` ausente ou `markdown` vazio |
| 400 | `message thread not found` | `message_thread_id` inválido para o grupo |
| 400 | `chat not found` | `chat_id` inválido |

## Exemplo funcional validado

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

## Regras de uso

- O conteúdo deve ser passado em `rich_message.markdown`, **não** em `text`
- Passar `text` em vez de `rich_message` resulta em erro 400 `rich message must be non-empty`
- Frontmatter YAML (`---`) deve ser removido antes do envio — não é markdown válido para renderização
- Para o tópico Geral do grupo `-1003870518428`, omitir `message_thread_id` (ver [[infraestrutura/telegram-topicos.md]])
- Para outros tópicos, incluir `message_thread_id` com o ID correspondente
- Conteúdo acima de 32.768 caracteres deve ser dividido em múltiplas mensagens

## Referência interna

- Implementação no Hermes: `/root/.hermes/plugins/platforms/telegram/adapter.py`, método `_rich_message_payload` (linha 1218) e `_send_rich_message` (linha 1322)
- Habilitação no Hermes: `gateway.platforms.telegram.extra.rich_messages: true` em `config.yaml`

## Conexões

- [[infraestrutura/telegram-topicos.md|Telegram Tópicos]] — IDs de chats e tópicos
- [[automacao/curador-wiki.md|Curador da Wiki]] — primeiro uso validado deste endpoint fora do Hermes
