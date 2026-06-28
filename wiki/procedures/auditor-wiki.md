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

**✅ V9 — Telegram: interação completa (todos os tipos de mensagem)**
Verificar todas as interações possíveis entre o usuário e o auditor via Telegram. São 3 tipos de mensagem, cada um com script próprio.

*Prefixo `/` para texto livre:* quando o auditor pede texto (Ajustar / Instruir), o usuário envia com `/` na frente. O auditor filtra mensagens sem `/` e faz strip antes de usar. O Hermes trata como comando desconhecido sem acionar o LLM.

**✅ V9a — Resumo executivo** (`/root/valid-9-resumo.sh`) — 3/3 passaram (2026-06-28)

| # | Botão | callback_data esperado |
|---|---|---|
| 1 | ✅ Prosseguir | `proceed` |
| 2 | 🔄 Recomeçar | `restart` |
| 3 | ❌ Encerrar | `abort` |

**✅ V9b — Finding corrigível** (`/root/valid-9-corrigivel.sh`) — 3/3 passaram (2026-06-28)

| # | Botão | callback_data esperado |
|---|---|---|
| 1 | ✅ Aplicar | `apply` |
| 2 | ❌ Pular | `skip` |
| 3 | ✏️ Ajustar → texto com `/` | `adjust` + texto capturado |

**✅ V9c — Finding não-corrigível** (`/root/valid-9-nao-corrigivel.sh`) — 2/2 passaram (2026-06-28)

| # | Botão | callback_data esperado |
|---|---|---|
| 1 | ✅ Entendido | `skip` |
| 2 | ✏️ Instruir correção → texto com `/` | `instruct` + texto capturado |

*Resultado esperado:* V9a 3/3, V9b 3/3, V9c 2/2.

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

## Problemas de design identificados nas validações

**D1 — Fluxo "Ajustar" exige texto exato que o usuário não tem de cabeça**
O fluxo atual de "Ajustar" pede que o usuário envie o `new_string` literal que substituirá o trecho no arquivo. Na prática isso é inviável: o usuário não sabe de cabeça os valores válidos para campos como `type` ou `status`.

Solução proposta: ao invés de pedir texto livre, o bot deve enviar um segundo teclado inline com as opções válidas para aquele campo (ex: todos os `type` válidos definidos em AGENTS.md) + botão "✏️ Outro" para casos realmente livres. Só ao pressionar "Outro" o bot pede texto livre com prefixo `/`.

Impacto: mudança no auditor script (`auditor-wiki-v1.sh`) e nos prompts dos agentes de pasta (que precisam incluir o campo `field` e `valid_values` nos findings corrigíveis).

**D2 — resolvido pelo V9 (3 scripts)**
Coberto por V9a/V9b/V9c que testam os 3 tipos de mensagem com os botões reais, incluindo o novo botão 🔄 Recomeçar.

---

## 🔴 Primeira execução real — 2026-06-28 (desastre)

### Timeline

```
[19:49:37] === Auditoria wiki 2026-06-28 ===
[19:49:37] Fase 1: análise estrutural
[19:49:37] Fase 1 concluída: 20 arquivos analisados
[19:49:37] Fase 2: iniciando agentes em paralelo
           8 agentes iniciados (concepts, history, procedures, systems, todo, tools, overlap, links)
[19:54:49] Fase 2 concluída (5 min 12 s)
           TODOS os 8 agentes produziram prose — findings = []
[19:54:49] Fase 3: coordenador consolidando relatórios
           Output inválido: "Expecting value: line 1 column 1 (char 0)"
           Findings consolidados: 0
[19:56:47] Fase 3 concluída (1 min 58 s)
[19:56:47] Fase 4: validação via Telegram
[19:57:04] === Auditoria concluída ===
```

**Tempo total:** ~7,5 minutos

### Falhas

**Falha #1 — 8/8 agentes produziram prosa em vez de JSON**
Nenhum agente (pasta, overlap ou links) retornou JSON válido. O fallback `findings: []` foi acionado para todos. Causa mais provável: o Claude CLI não recebeu (ou ignorou) a instrução de `--output-format json` OU o system prompt não foi carregado corretamente, fazendo o modelo responder em linguagem natural.

**Falha #2 — Coordenador recebeu input vazio e travou**
Com todos os relatórios vazios, o coordenador recebeu algo inesperado e produziu string vazia — `json.loads()` falhou com "Expecting value: line 1 column 1 (char 0)".

**Falha #3 — Mensagem Telegram sem conteúdo**
Fase 4 disparou, mas sem findings para apresentar, a mensagem no Telegram veio vazia/inútil.

### Consumo de tokens

- Gastou **~75% da cota disponível em ~5 minutos** (~4 horas de uso normal comprimidas em 5 min)
- Esse volume normalmente é consumido ao longo de horas em uso normal do Claude Code via Hermes
- Proporção sugere que **cada agente consumiu uma quantidade massiva de tokens** — possivelmente por ler arquivos inteiros da wiki sem necessidade ou por gerar respostas longas em prosa (cada prosa = dezenas de milhares de tokens de saída)

### Análise de causa raiz

| Fator | Impacto |
|---|---|
| **8 agentes paralelos, cada um lendo múltiplos arquivos** | Cada agente recebeu o conteúdo de uma pasta inteira como contexto. Com 20 arquivos na wiki, agentes como `tools/` (4 arquivos) e `systems/` (4 arquivos) podem ter consumido 10K–20K+ tokens de entrada cada. Multiplicado por 8 = 80K–160K+ tokens de entrada em paralelo. |
| **Nenhum agente retornou JSON** | Todo o processamento foi desperdiçado — os tokens de saída (prosa longa) foram descartados pelo fallback. Se os agentes tivessem retornado JSON compacto como esperado, o custo de saída seria drasticamente menor. |
| **Prompt `--output-format json` pode não estar funcionando** | O Claude CLI não tem garantia de que `--output-format json` force o modelo a obedecer — o modelo pode ainda responder em prosa, principalmente se o system prompt não deixar explícito. |
| **Fallback `findings: []` esconde o erro** | Em vez de travar e alertar que o formato está errado, o script silenciosamente aceita prosa, joga fora e segue com findings vazio. O usuário só descobre o problema no final, depois de pagar o custo. |
| **Sem dry-run ou modo verboso** | O script não tem um modo de teste que mostre o prompt exato enviado a cada agente antes de disparar a API paga. |

### Lições

1. **Teste de formato primeiro** — V2 (agente isolado) foi planejado mas nunca executado. Rodar um único agente com `--output-format json` antes do run completo teria revelado o problema com 1/8 do custo.
2. **Prompt precisa ser explícito sobre formato** — A instrução de output em JSON puro precisa estar no corpo do prompt (não só no `--output-format` flag), com exemplo concreto do JSON esperado.
3. **Fallback deve ALERTAR, não silenciar** — Se um agente produz prosa, o script deveria abortar ou pelo menos logar um WARNING com os primeiros 200 chars do output, não só registrar findings vazio e seguir.
4. **8 agentes paralelos drena recursos rapidamente** — O volume de 35-50 requests com contexto crescente em 5 minutos consome o orçamento de forma desproporcional. Reduzir para 2-3 agentes por run, ou serializar.

---

### Análise do consumo de tokens — por que 75% em 5 minutos?

O gasto massivo não foi acidental — é consequência direta da arquitetura do script. Abaixo, a decomposição do que cada agente consumiu.

#### Mecânica: como Claude Code conta o consumo

Cada chamada ao `claude` CLI inicia uma **sessão de múltiplos turns** quando `--allowedTools Read` está ativo. Cada turno (tool call) conta como um request — e a entrada de cada turno inclui **todo o contexto acumulado dos turnos anteriores** (Claude Code não descarta histórico da conversa).

Ou seja: se um agente faz 5 Read calls, são 5 requests, e cada request custa mais que o anterior porque o contexto cresceu.

#### O que cada agente consumiu

| Agente | Pasta | Read calls | Contexto final estimado |
|---|---|---|---|
| **Folder: concepts** | `concepts/` | AGENTS.md + 4 arquivos = 5 | ~80KB (orquestrador.md = 30KB) |
| **Folder: history** | `history/` | AGENTS.md + 3 arquivos = 4 | ~90KB (2026-06-24.md = 63KB) |
| **Folder: procedures** | `procedures/` | AGENTS.md + 4 arquivos = 5 | ~114KB (curador-historico = 50KB) |
| **Folder: systems** | `systems/` | AGENTS.md + 4 arquivos = 5 | ~68KB (endpoints = 23KB) |
| **Folder: tools** | `tools/` | AGENTS.md + 4 arquivos = 5 | ~61KB (obsidian-git = 16KB) |
| **Folder: todo** | `todo/` | AGENTS.md + 1 arquivo = 2 | ~35KB |
| **Folder: diario** | `diario/` | AGENTS.md + 1 arquivo = 2 | ~27KB |
| **Overlap** | todos | **pode ler QUALQUER arquivo** para confirmar suspeitas | potencialmente 300KB+ |
| **Links** | todos | lê cada destino de wikilink com label | depende do número de wikilinks |

**Além das Reads, CADA agente recebeu:**

| Item | Tamanho |
|---|---|
| System prompt (`prompt-awv1-pasta.md`) | ~3,5KB |
| `struct_str` — JSON com dados de todos os 21 arquivos da wiki | **~10KB** |
| Instrução no user prompt | ~0,5KB |

Ou seja, cada agente já começava com ~14KB de contexto **antes de qualquer Read call**.

#### O efeito multiplicador

O problema não é um agente isolado — é o **paralelismo combinado com o crescimento geométrico do contexto**:

1. **8 processos Claude simultâneos**, cada um numa sessão independente
2. Cada processo faz 2 a 5 Read calls (dependendo do número de arquivos na pasta)
3. Cada Read call adiciona o **arquivo inteiro** ao contexto acumulado
4. O contexto **cresce a cada turno** — o quinto Read call de um agente tem 5× mais tokens de entrada que o primeiro
5. **Todos competem pela mesma cota** do Claude Code CLI

#### Estimativa total

| Componente | Cálculo |
|---|---|
| Input tokens (todos os agentes, todas as turns) | ~700KB–1,5MB |
| Output tokens (cada agente gera resposta final) | ~5–20KB × 8 = 40–160KB |
| Read calls totais | ~35–50 chamadas |
| Requests totais (cada turno = 1 request) | ~35–50 |

Para comparação: uma sessão normal de Claude Code onde você faz perguntas sequenciais gera ~5–15 requests por hora com contexto estável. O auditor fez **35–50 requests em 5 minutos** com contexto **crescendo em cada request**.

#### O fator "struct_str em todo lugar"

Cada agente recebe o `struct_str` completo (dados de TODOS os 21 arquivos) mesmo que só precise auditar sua própria pasta. O agente `todo/` (1 arquivo de 7KB) recebeu ~10KB de struct_str com dados de 20 arquivos que não lhe dizem respeito — **mais dados de entrada que o próprio conteúdo que ele precisava auditar**.

#### Por que no uso normal não gasta isso

No uso normal do Claude Code (via CLI interativa ou via Hermes), os requests são feitos de forma **sequencial e espaçada** — uma conversa de cada vez, com pausas entre requests. O auditor lança **8 conversas simultâneas**, cada uma fazendo múltiplos tool calls em RÁPIDA SUCESSÃO. Esse consumo em rajada drena a cota muito mais rápido que o uso sequencial.

## Conexões

- [[AGENTS.md]] — taxonomia, templates e checklist de Lint que este script implementa
- [[wiki/procedures/curador-wiki.md|Curador da Wiki]] — automação de referência com mesma arquitetura
- [[wiki/todo/proximos-passos.md|Próximos passos]] — onde os findings desta auditoria podem gerar itens
