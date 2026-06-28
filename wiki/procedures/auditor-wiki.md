---
type: procedure
tags: [auditoria, lint, wiki, multi-agente, telegram]
title: Auditor da Wiki
description: Automação multi-agente que executa health-check completo da wiki, apresenta cada correção via Telegram para aprovação e aplica somente o que for autorizado.
timestamp: 2026-06-28T00:00:00-03:00
status: draft
---

# Auditor da Wiki

Automação bash que roda uma auditoria completa da wiki em 6 fases, valida cada correção individualmente via Telegram com botões interativos e commita somente o que o usuário aprovar.

Segue o checklist de Lint definido em [[AGENTS.md]] — verifica taxonomia, seções obrigatórias, type OKF, frontmatter, orphans, sync index vs git, qualidade de nomes de arquivo, labels de wikilinks, sobreposição semântica e consistência de status.

## O que faz

- **Fase 1:** análise estrutural em Python puro — sem LLM. Descobre todos os arquivos dinamicamente via `os.walk`, extrai frontmatter, agrupa por pasta, extrai mapa de wikilinks e calcula diff `git ls-files` vs `index.md`.
- **Fase 2:** agentes Claude em paralelo — um por pasta com arquivos (dinâmico, escala automaticamente) + agente Overlap (sobreposição cross-folder) + agente Links (wikilinks quebrados e labels incorretos). Todos rodam simultaneamente.
- **Fase 3:** agente coordenador recebe todos os relatórios, deduplica findings, prioriza por severidade e classifica quais são corrigíveis automaticamente.
- **Fase 4:** envia resumo executivo no Telegram com botões. Usuário aprova prosseguir ou encerra sem alterar nada.
- **Fase 5:** para cada finding em ordem de prioridade — agente corretor gera o diff proposto → Telegram apresenta com botões → usuário decide. Se aplicado: commit imediato do arquivo + `log.md`. Sequencial (um finding por vez, garante que edições no mesmo arquivo não conflitem).
- **Fase 6:** push de todos os commits + resumo final no Telegram.

## Gatilho

Manual — on demand. Executar quando solicitado pelo usuário para health-check da wiki.

## Como executar

```bash
bash /root/auditor-wiki-v1.sh
```

Log de execução: `/var/log/auditor-wiki.log`

## Entradas e saídas

**Entradas:**
- Todos os arquivos `.md` em `wiki/` (descobertos dinamicamente via `os.walk`)
- `/root/wiki/AGENTS.md` — taxonomia e regras (lido por todos os agentes)
- `/root/wiki/index.md` — para o diff estrutural

**Saídas:**
- Commits individuais por finding aplicado (arquivo corrigido + `log.md`)
- Push de todos os commits ao final
- Notificações Telegram em cada etapa (resumo, findings, confirmações, resultado final)
- `/var/log/auditor-wiki.log` — log de execução com timestamps BRT
- `/tmp/auditor-wiki-*/` — arquivos temporários removidos ao final

## Arquivos

| Arquivo | Papel |
|---|---|
| `/root/auditor-wiki-v1.sh` | Script principal |
| `/root/prompt-awv1-pasta.md` | System prompt dos agentes de pasta |
| `/root/prompt-awv1-coordenador.md` | System prompt do coordenador |
| `/root/prompt-awv1-corretor.md` | System prompt do agente corretor |
| `/root/prompt-awv1-overlap.md` | System prompt do agente de sobreposição |
| `/root/prompt-awv1-links.md` | System prompt do agente de links |

## Escopo dos agentes de pasta

Dinâmico — um agente por pasta que tiver arquivos `.md`. Para a wiki atual:

| Agente | Pasta | Checks |
|---|---|---|
| systems | `systems/` | taxonomia, seções, type, frontmatter, status, nome de arquivo, labels |
| tools | `tools/` | idem |
| procedures | `procedures/` | idem |
| concepts | `concepts/` | idem |
| history | `history/` | idem (corpo imutável — só frontmatter corrigível) |
| todo | `todo/` | idem + items maduros para promover a procedures/ |
| Overlap | todos | sobreposição semântica cross-folder |
| Links | todos | wikilinks quebrados e labels divergentes do título real |

## Interação via Telegram — Fase 4

Para cada finding corrigível automaticamente:
```
📋 F3 — 🔴 CRÍTICO (3/12)
📄 systems/hermes.md
❌ Problema: ...

✂️ Correção proposta:
`- texto antigo`
`+ texto novo`

[✅ Aplicar]  [❌ Pular]  [✏️ Ajustar]
```

Para findings sem correção automática (sobreposição, rename):
```
📋 F7 — 🟡 MÉDIO (7/12)
📄 systems/hermes.md
⚠️ Problema: ...
💡 Sugestão: ...

[✅ Entendido]  [✏️ Instruir correção]
```

**Ajustar:** usuário envia o `new_string` correto via mensagem — aplicado no lugar do proposto.
**Instruir correção:** usuário descreve o que fazer em texto livre — agente corretor executa com essa instrução.

## Commits por finding

Cada finding aplicado gera um commit imediato:
- `git add <arquivo_editado> log.md`
- `git commit -m "edit(<arquivo>): <finding_id> — <descrição>"`

Push único ao final, após todos os findings processados.

## Validações pré-produção

Executar na ordem antes do primeiro run completo. Cada validação é independente e pode ser feita isoladamente.

> ⚠️ **Protocolo obrigatório para executar qualquer validação:**
>
> 1. **Criar** o script de teste (em `/root/test-v<N>-*.sh`)
> 2. **Documentar** no item V\<N\> desta seção: o que o script faz, passo a passo, e o resultado esperado
> 3. **Commitar** wiki + log.md antes de rodar qualquer coisa
> 4. **Rodar** o script — somente após o commit estar no remoto
>
> Nunca rodar sem documentar antes. A autorização do usuário para rodar é dada após ver a documentação commitada.

**V1 — Fase 1: descoberta dinâmica**
Rodar isolado e inspecionar o JSON gerado via bash. Verificar: todos os arquivos listados, `files_by_folder` agrupa corretamente, `wikilinks_map` tem entradas, diff git vs index está preciso.

**V2 — Agente de pasta isolado**
Descomentar e rodar só o agente de um folder (ex: `systems/`) antes de rodar todos em paralelo. Verificar: output é JSON válido, findings têm todos os campos obrigatórios, nenhum prose no lugar de JSON.

**V3 — Agente Overlap isolado**
Rodar só o agente Overlap com o inventário real. Verificar: lê os arquivos suspeitos com Read, não flaga falsos positivos, output é JSON válido.

**V4 — Agente Links isolado**
Rodar só o agente Links com o `wikilinks_map` real. Verificar: detecta links quebrados corretamente, compara labels com `title:` do frontmatter, output é JSON válido.

**V5 — Extração JSON (fallback)**
Verificar que o fallback `findings: []` é acionado corretamente quando um agente produz prose, e que o coordenador lida com relatório vazio sem travar.

**V6 — Coordenador isolado**
Alimentar o coordenador com relatórios reais dos agentes e verificar: output é JSON válido com `executive_summary` e `findings`, campo `correctable` está classificado coerentemente, nenhum finding duplicado.

**V7 — Agente Corretor isolado**
Passar um finding real e verificar: `old_string` é substring exata do arquivo (crítico — se não for, o `apply_edit` falha), `new_string` está correto, `edits` tem o caminho de arquivo no formato certo (`systems/hermes.md`, não absoluto).

**V8 — Telegram: token e chat_id**
Verificar que `TELEGRAM_BOT_TOKEN` está disponível em `~/.hermes/.env` e que `CHAT_ID=-1003870518428` corresponde ao chat correto. Testar com um `sendMessage` simples antes de rodar o script completo.

**⚠️ V9 — Telegram: inline keyboard**
Verificar que os botões aparecem corretamente no Telegram (o bot precisa ter permissão de enviar mensagens com `reply_markup` no grupo). Testar `answerCallbackQuery` — se não for chamado, o botão fica com loading infinito.

*Script de teste:* `/root/test-v9-inline-keyboard.sh`

O script:
1. Drena updates pendentes para zerar o offset
2. Envia `sendMessage` com `reply_markup` (inline keyboard com 2 botões)
3. Aguarda até 120s pelo `callback_query` do usuário via `getUpdates` (long-polling)
4. Chama `answerCallbackQuery` com `text="✅ Recebido!"` — isso remove o loading do botão
5. Remove os botões com `editMessageReplyMarkup` (inline_keyboard vazia)
6. Imprime `✅ V9 PASSOU` com `callback_data` recebido e confirmação do `answerCallbackQuery`, ou `❌ V9 FALHOU` com timeout

*Resultado esperado:* `status: ok`, `answer_ok: True`, botões sumindo da mensagem após clique.

*Resultado obtido (2026-06-28):* ⚠️ **Parcialmente validado.** Testado com 2 botões — mas o script real envia 3 botões [✅ Aplicar] [❌ Pular] [✏️ Ajustar] para findings corrigíveis. Configuração diferente de `inline_keyboard`. Requer novo teste com 3 botões antes de marcar como validado.

**V10 — apply_edit: old_string exato**
O maior risco do script. O agente corretor copia o trecho do arquivo, mas LLMs às vezes normalizam espaços ou quebras de linha. Verificar na primeira aplicação real se o `old_string` está sendo encontrado ou se cai no erro "old_string não encontrado".

**V11 — Commits por finding**
Verificar que o `git add` passa os caminhos relativos corretos ao `WIKI_DIR` (não absolutos, não relativos a `wiki/`). Rodar um finding de teste e inspecionar `git log --oneline -3`.

**V12 — Push final**
A wiki tem hook post-commit que auto-pusha. Verificar se isso cria conflito com o `git push` explícito da Fase 6 (pode resultar em "Everything up-to-date", que é ok, ou em erro se o hook já tiver pushado algo diferente).

**V13 — Execução paralela: recursos**
8 agentes simultâneos consomem memória e rate limit da API. Verificar memória disponível na VPS e se o gateway Hermes interfere com chamadas externas ao `claude` CLI.

**V14 — Pasta diary/ vazia**
Verificar que o script não lança agente para `diary/` quando a pasta não tem arquivos `.md` (o `if not file_list: continue` cobre isso, mas vale confirmar).

**V15 — Timeout Telegram**
O timeout padrão é 10 min por resposta. Verificar se é suficiente ou se precisa ajustar `TG_TIMEOUT` no script.

**V16 — claude CLI: autenticação standalone**
O auditor invoca `claude` como subprocesso bash, fora do contexto do Hermes gateway. Verificar se a autenticação funciona de forma independente (`claude -p "teste" --output-format json`). Se a autenticação depender de variável de ambiente que o gateway injeta em runtime, o auditor não vai tê-la e vai falhar silenciosamente nos agentes da Fase 2.

**V17 — Dois findings consecutivos no mesmo arquivo**
`apply_edit` usa `str.replace(old_string, new_string, 1)`. Se dois findings apontam para o mesmo arquivo e são processados em sequência, o primeiro pode alterar o contexto onde o `old_string` do segundo estava — o segundo finding cai no erro "old_string não encontrado". Verificar se o coordenador pode ser instruído a agrupar edits do mesmo arquivo num único finding, ou confirmar que a Fase 5 sequencial já mitiga o risco.

**V18 — poll_text: conflito com Hermes e filtro por prefixo `!`**

Contexto: o auditor e o Hermes usam o mesmo bot-token com long-polling (`getUpdates`, sem webhook). Quando o auditor aguarda `poll_text` (fluxo "Ajustar" ou "Instruir"), Hermes também está escutando — há corrida para capturar a mensagem do usuário, com resultado não-determinístico. A solução adotada: exigir que respostas ao auditor comecem com `!`; o auditor filtra e faz strip do prefixo; o Hermes deve ignorar silenciosamente.

*Script de teste:* `/root/test-v18-poll-text-prefix.sh`

O script:
- **Fase 1** — solicita uma mensagem SEM `!` e aguarda 30s. Documenta se o auditor captura (comportamento atual sem filtro) ou não vê nada (Hermes capturou primeiro). Não é pass/fail — é observação do estado atual.
- **Fase 2** — solicita uma mensagem COM `!` (ex: `! type: system`). `poll_text` com filtro ignora mensagens sem `!` e captura somente a prefixada. Faz strip do `!` e espaço(s), ecoa o texto limpo de volta no Telegram.
- **Verificação manual obrigatória:** observar se o Hermes responde à mensagem com `!`. Se sim → `!` não é seguro como prefixo, escolher outro (ex: `/adj`). Se não → `!` é seguro.

*Resultado esperado:* Fase 2 captura a mensagem com `!`, strip correto, Hermes não responde.

*Resultado obtido (2026-06-28):* ❌ **Prefixo `!` não é seguro.** O script capturou corretamente (`raw: ! teste...`, `strip: teste...`), mas o Hermes também recebeu e respondeu à mensagem — gastando token. Ambos consumiram o mesmo update simultaneamente. A abordagem de prefixo simples não resolve o conflito; é necessária outra estratégia (ver nota abaixo).

**V18b — poll_text: prefixo `/` como alternativa sem gasto de token**

Hipótese: o Hermes trata mensagens iniciadas com `/` como comandos. Se `/algo` não é um comando reconhecido, ele responde "desconhecido" ou ignora — sem acionar o LLM. Isso evita gasto de token mesmo que ambos recebam o update simultaneamente.

Formato de uso: `/ <texto>` (ex: `/ type: system`) — o auditor captura mensagens iniciando com `/`, faz strip do `/ ` e usa o restante como `new_string`.

*Script de teste:* `/root/test-v18b-slash-prefix.sh`

O script:
1. Drena offset inicial
2. Envia instrução no Telegram pedindo mensagem com `/` na frente
3. `poll_text` com filtro: ignora mensagens sem `/`, captura a primeira com `/`
4. Strip do `/` e espaço(s), ecoa o resultado limpo no Telegram

*Verificação manual obrigatória após rodar:*
- Hermes não respondeu → prefixo `/` seguro ✅
- Hermes respondeu "comando desconhecido" (sem LLM) → sem gasto de token, aceitável ✅
- Hermes respondeu via LLM → gasta token, prefixo `/` não resolve ❌

*Resultado esperado:* captura e strip corretos; Hermes não aciona LLM.

## Problemas de design identificados nas validações

**D1 — Fluxo "Ajustar" exige texto exato que o usuário não tem de cabeça**
O fluxo atual de "Ajustar" pede que o usuário envie o `new_string` literal que substituirá o trecho no arquivo. Na prática isso é inviável: o usuário não sabe de cabeça os valores válidos para campos como `type` ou `status`.

Solução proposta: ao invés de pedir texto livre, o bot deve enviar um segundo teclado inline com as opções válidas para aquele campo (ex: todos os `type` válidos definidos em AGENTS.md) + botão "✏️ Outro" para casos realmente livres. Só ao pressionar "Outro" o bot pede texto livre com prefixo `/`.

Impacto: mudança no auditor script (`auditor-wiki-v1.sh`) e nos prompts dos agentes de pasta (que precisam incluir o campo `field` e `valid_values` nos findings corrigíveis).

**D2 — V9 testou 2 botões, script real envia 3**
O teste V9 enviou `[✅ Botão A] [❌ Botão B]`. O script real envia `[✅ Aplicar] [❌ Pular] [✏️ Ajustar]` para findings corrigíveis e `[✅ Entendido] [✏️ Instruir correção]` para não-corrigíveis. Ambos os casos precisam de teste próprio.

## Conexões

- [[AGENTS.md]] — taxonomia, templates e checklist de Lint que este script implementa
- [[wiki/procedures/curador-wiki.md|Curador da Wiki]] — automação de referência com mesma arquitetura
- [[wiki/todo/proximos-passos.md|Próximos passos]] — onde os findings desta auditoria podem gerar itens
