---
type: procedure
tags: [auditoria, eval, validacao, multi-agente, diagnostico, design]
title: Auditor da Wiki — Evals e Validação
description: Diagnóstico da primeira execução do auditor, lições dos raws agents-cli e Karpathy, e processo de eval estruturado com gates obrigatórios antes de qualquer run em produção.
timestamp: 2026-06-29T00:00:00-03:00
status: draft
---

# Auditor da Wiki — Evals e Validação

> ⚠️ **REGRA OBRIGATÓRIA:** após executar qualquer eval — mesmo que seja revisão manual sem tokens — documentar o resultado aqui (data, checklist, aprovado/reprovado) antes de qualquer outra ação. Nenhum próximo eval começa sem o anterior estar registrado.

> ⚠️ **REGRA OBRIGATÓRIA — TRANSPARÊNCIA DE CUSTOS:** todo eval que executa subagente deve reportar obrigatoriamente **tanto os tokens do subagente quanto os tokens da sessão pai**. Rastreabilidade de custo é o princípio central — omitir qualquer das duas partes invalida o resultado, independente do JSON retornado estar correto. O runner de cada eval deve incluir os tokens do pai como critério de aprovação, não como opcional.

> 🚫 **REGRA OBRIGATÓRIA — STATUS É DEFINIDO PELO USUÁRIO:** o agente **NUNCA** define o status final de um eval como "aprovado", "reprovado", "incompleto" ou qualquer outro. O agente reporta os resultados brutos (checklist, métricas, observações). O status só é registrado no documento após o **Giovani confirmar explicitamente** qual é. Registrar status sem essa confirmação é proibido, independente de quantos critérios foram satisfeitos.

Documento de análise e processo de validação para o [[wiki/procedures/auditor-wiki.md|Auditor da Wiki]]. Combina o diagnóstico da primeira execução real (2026-06-28) com lições extraídas dos raws [[raw/agents-cli-README.md|Google Agents CLI]] e [[raw/akshay-pachaar-karpathy-agentic-engineering-tooling.md|Karpathy's Agentic Engineering Finally Has Proper Tooling]].

---

## O que é o Auditor da Wiki

Automação multi-agente de health-check completo da wiki. A ideia central: executar o checklist de Lint do [[AGENTS.md]] de forma sistemática e autônoma — verificando taxonomia, seções obrigatórias, frontmatter OKF, orphans, sync index vs git, sobreposição semântica e consistência de status — e entregar cada correção individualmente ao usuário via Telegram para aprovação antes de aplicar.

**Arquitetura da ideia (independente de implementação):**
- **Análise estrutural** sem LLM — descoberta dinâmica de arquivos, extração de frontmatter, mapa de wikilinks, diff git vs index
- **Agentes especializados em paralelo** — um por pasta + agente de sobreposição cross-folder + agente de links quebrados
- **Coordenador** — consolida relatórios, deduplica findings, prioriza por severidade
- **Loop de aprovação via Telegram** — cada finding apresentado individualmente com botões; correção aplicada só com autorização explícita; commit imediato por finding aprovado
- **Push único** ao final com resumo

A primeira implementação foi um script bash (`/root/auditor-wiki-v1.sh`) com subagentes invocados via `claude` CLI. Essa implementação **falhou** na primeira execução real (2026-06-28) — zero findings entregues, 75% da cota de tokens consumida em 5 minutos. Está abandonada. Nada nela é obrigatório — a ideia central (análise → aprovação granular → commit por finding) está sendo reimplementada com uma arquitetura diferente, descrita na seção "Arquitetura candidata" abaixo.

Documentação completa da implementação v1 e do fracasso: [[wiki/procedures/auditor-wiki.md|Auditor da Wiki]].

---

## Diagnóstico da primeira execução

> **Diagnóstico completo** (timeline, falhas, análise de consumo de tokens, lições): [[wiki/procedures/auditor-wiki.md#🔴-primeira-execução-real-—-2026-06-28-desastre|auditor-wiki.md § Primeira execução real]].

### O que aconteceu

```
[19:49:37] 8 agentes iniciados em paralelo
[19:54:49] TODOS os 8 agentes produziram prose — findings = []
[19:54:49] Coordenador: output inválido — json.loads falhou
[19:57:04] Auditoria concluída sem nenhum finding aplicado
```

**75% da cota de tokens consumida em 5 minutos.** Resultado zero.

### As três falhas

**F1 — Nenhum agente retornou JSON**
O flag `--output-format json` do Claude CLI não garante que o modelo obedeça — o modelo pode responder em prosa mesmo assim, especialmente se o system prompt não for explícito. O fallback `findings: []` aceitou a prosa silenciosamente e descartou tudo.

**F2 — Coordenador recebeu input vazio e travou**
Com todos os relatórios descartados, o coordenador recebeu algo inesperado e produziu string vazia. `json.loads()` falhou com `"Expecting value: line 1 column 1"`.

**F3 — Mensagem Telegram sem conteúdo**
A Fase 4 disparou com zero findings para apresentar. O usuário recebeu uma notificação inútil.

### Causas raiz

| Causa | Impacto |
|---|---|
| Contrato de output não testado antes do run completo | F1 — 8 agentes produziram prosa sem que ninguém soubesse |
| Fallback silencioso em vez de fail-fast | F1 → F2 → F3 em cascata, erro invisível até o final |
| 8 subprocessos Claude paralelos, cada um stateless | 35–50 requests em 5 min, contexto crescendo a cada tool call |
| `struct_str` completo enviado a todos os agentes | Agente `todo/` (1 arquivo) recebeu dados de 20 arquivos que não lhe dizem respeito |
| Validações V2–V17 existiam no papel mas eram opcionais | O run completo foi disparado sem nenhum agente ter sido testado isoladamente |

---

## Lições dos raws

### Karpathy: spec design → eval loops → deploy

O artigo de Akshay Pachaar cita Karpathy definindo agentic engineering em três pilares: **spec design, eval loops e security oversight**. O auditor falhou nos dois primeiros.

A sequência não é sugestão — é gate. Você não pode ir para o deploy sem ter passado pelo eval. O agents-cli torna isso estrutural: `eval generate` e `eval grade` são comandos separados na CLI. Você literalmente não chega no `deploy` sem executá-los. O auditor não tinha nenhuma estrutura que impedisse pular as validações.

> "Karpathy noted that while 89% of teams maintain observability, only 52% implement evaluation frameworks."

O auditor foi para produção nessa outra metade.

### agents-cli: skills vs subprocessos

O README do agents-cli define a distinção central: `agents-cli` é uma ferramenta *para* coding agents, não um coding agent. Skills são blocos de instrução injetados no agente que já está rodando — o agente usa suas capacidades nativas (ler arquivos, raciocinar) sem lançar subprocessos.

O auditor fez o oposto: lançou 8 subprocessos `claude` independentes via bash. Cada processo:
- Começava com ~14KB de contexto (system prompt + struct_str)
- Fazia 2 a 5 Read calls, cada uma acumulando ao contexto anterior
- Crescia geometricamente: o quinto Read de um agente tem 5× mais tokens de entrada que o primeiro
- Competia com os outros 7 processos pela mesma cota simultânea

A arquitetura multi-agente em si não é o problema. O mecanismo é que não escala: sessões paralelas stateless com contexto acumulativo e sem contrato de interface verificado.

### O princípio de prompt-as-contract

O agents-cli injeta 7 skills com interfaces bem definidas (scaffold, eval, deploy, publish, observability, ADK code, workflow). Cada skill tem input format → processing → output format explícitos. O coding agent sabe exatamente o que vai receber e o que precisa entregar.

Os prompts do auditor (`prompt-awv1-pasta.md`, etc.) tentam fazer isso, mas sem o exemplo concreto do JSON esperado no corpo do prompt — dependem do flag `--output-format json` que não é garantia suficiente. Um contrato de prompt completo inclui: instrução de formato + exemplo de output + instrução de fallback ("se não tiver findings, retorne `{\"findings\": []}`").

---

## Arquitetura candidata: subagentes nativos do Claude Code

O diagnóstico e os raws convergem para uma arquitetura concreta: em vez de um bash script invocando o `claude` CLI como subprocesso externo, o **Claude Code é o orquestrador**, usando sua ferramenta nativa `Agent` para spawnar sub-Claudes especializados.

**Por que é diferente do v1:**
No v1, o bash era o gerente — chamava o `claude` 8 vezes como ferramenta externa. Bash não foi feito pra gerenciar agentes: não controla estado, não propaga erros, não sabe o que fazer quando um filho falha silenciosamente. Na arquitetura candidata, o orquestrador é o próprio Claude Code: spawna subagentes via `Agent`, recebe os resultados de volta na sessão principal, lida com erros nativamente, e conduz o loop Telegram dentro da mesma sessão. A ideia de agentes especializados por pasta permanece válida — o que muda é o mecanismo.

**Como funciona:**
1. Sessão principal do Claude Code inicia a auditoria
2. Para cada pasta, spawna um subagente especializado via ferramenta `Agent`
3. Subagente lê os arquivos, retorna findings em JSON para a sessão principal
4. Sessão principal recebe os resultados, coordena, envia pro Telegram
5. Aguarda resposta do usuário (botão), aplica edição, commita — tudo dentro da mesma sessão

**Modelo:** `claude-sonnet-4-6` para todos os subagentes.

---

## Princípios de redesign

Derivados do diagnóstico + raws. Não são mudanças de implementação ainda — são os critérios que qualquer nova versão precisa satisfazer.

**1. Fail-fast, não silencioso**
Se um subagente produz prosa em vez de JSON, o orquestrador deve detectar imediatamente, logar os primeiros 300 chars do output e alertar via Telegram. Nunca aceitar silenciosamente e seguir com dados vazios.

**2. Testar o contrato de output antes de gastar tokens**
O primeiro run de qualquer versão nova do auditor deve ser com um único agente, na pasta mais pequena, e verificar que o JSON retornado tem todos os campos obrigatórios. Zero tokens adicionais gastos antes disso passar.

**3. Serial antes de paralelo**
Provar que um agente funciona → provar que dois funcionam → paralelizar. Nunca lançar 8 simultâneos com código não validado.

**4. Prompt-as-contract completo**
Cada system prompt deve incluir: (a) instrução de formato, (b) exemplo concreto do JSON esperado, (c) instrução explícita para o caso vazio (`findings: []`), (d) proibição de prosa. O flag `--output-format json` é complemento, não substituto.

**5. Struct_str filtrado por agente**
Cada agente recebe apenas os dados da sua pasta. O agente `todo/` não precisa saber dos 20 arquivos de outras pastas. Isso reduz o contexto de entrada e o custo por request.

**6. Eval obrigatório — nenhum run completo sem todos os evals passados**
Os evals abaixo substituem as validações opcionais V1–V17. A diferença: são sequenciais e bloqueantes. O Eval N não pode ser pulado se o Eval N-1 não passou.

**7. Sintético antes de real**
Qualquer eval que invoca subagente usa dados sintéticos (conteúdo inline ou arquivo mínimo em `/tmp/`) antes de usar arquivos reais da wiki. Dados reais só entram quando o contrato de output já foi validado com sintético. O princípio 2 ("zero tokens antes disso passar") não se cumpre com arquivos reais de 10KB+. Um custo "esperado" que não é critério de reprovação não é um gate — é wishful thinking.

---

## Evals de validação

Sequência única e bloqueante — Eval N só inicia se Eval N-1 passou. Nenhum eval pode ser pulado.

**Modelo para todos os subagentes:** `claude-sonnet-4-6`

### Campo `_meta` — definição

Todo subagente deve incluir `_meta` no JSON de saída a partir do Eval 2-A. Campos obrigatórios:

| Campo | Tipo | O que é |
|---|---|---|
| `files_read` | array de strings | nomes dos arquivos lidos (basename) |
| `read_calls` | integer | número de Read calls feitas |
| `approx_chars_read` | integer | soma aproximada de `len()` de cada conteúdo lido |
| `limit_reached` | boolean | `true` se atingiu o limite de calls antes de terminar |

---

### Como capturar estatísticas de tokens

Cada chamada de API é registrada no JSONL de sessão do Claude Code em `~/.claude/projects/-root/<session-id>.jsonl`. O campo `message.usage` contém:

```json
{
  "input_tokens": 950,
  "cache_read_input_tokens": 0,
  "cache_creation_input_tokens": 950,
  "output_tokens": 80
}
```

**Metodologia de leitura por eval:**
1. Antes de spawnar o subagente: registrar o número de linhas atual do JSONL
2. Após a resposta: ler as entradas novas (linhas após o baseline) e somar os campos de `message.usage`
3. Reportar: `input_tokens` (novos), `cache_read_input_tokens`, `output_tokens` — separados para distinguir custo real de cache hit

`cache_read_input_tokens` alto indica que o system prompt do agente já estava cacheado (custo ~10x menor). O custo "de verdade" é `input_tokens + cache_creation_input_tokens`.

---

### Eval 1 — Contrato de output (estático, zero tokens)

Verificação antes de qualquer execução — leitura do arquivo de definição do subagente e do system prompt.

- [x] Arquivo `.claude/agents/auditor-pasta.md` existe com frontmatter correto
- [x] `model: claude-sonnet-4-6` declarado no frontmatter do subagente
- [x] System prompt inclui exemplo concreto do JSON esperado com todos os campos
- [x] System prompt instrui caso vazio: `{"findings": []}`
- [x] System prompt proíbe prosa explicitamente

**Critérios a serem observados:** todos os 5 itens verificados manualmente no arquivo do subagente e do system prompt.

**2026-06-29 — realizada — ✅ APROVADA**
Arquivo criado em `/root/.claude/agents/auditor-pasta.md`. Todos os 5 itens verificados manualmente.

---

### Eval 2-A — Mecanismo: agente genérico retorna JSON? (custo mínimo absoluto)

Testa o mecanismo de spawn, sem nenhuma relação com o auditor. Agente genérico (sem system prompt próprio), prompt de 5 palavras. Só confirma que a ferramenta `Agent` funciona e que JSON chega parseável do outro lado.

**Modelo:** deve ser declarado explicitamente — `claude-sonnet-4-6`. Sempre exibir qual modelo foi usado no resultado.

**Prompt enviado ao subagente:**
> `Retorne apenas este JSON, sem nenhuma outra palavra: {"ok": true}`

**De onde vêm os tokens (estimativa corrigida pós-execução real):**

| Parte | Estimado original | Real (2026-06-29) |
|---|---|---|
| Prompt enviado | ~20 | 21 |
| System prompt interno do Claude Code | não estimado | ~12.582 |
| Resposta `{"ok": true}` | ~10 | incluído nos 2.426 de output da sessão |
| **`subagent_tokens` total** | **~30** | **12.603** |

⚠️ **Lição:** agentes genéricos no Claude Code carregam um system prompt interno substancial (~12K tokens) que não está visível em nenhum arquivo local. A estimativa de ~30 tokens estava errada por 400×. Todos os evals seguintes devem contar com esse overhead de base.

**Critérios:**
- [x] Agente spawna sem erro
- [x] `json.loads(resposta)` não lança exceção
- [x] `resposta["ok"] == True`
- [x] Nenhuma prosa fora do JSON

**Critérios a serem observados:** JSON parseável? `ok == true`? prosa fora do JSON?

**Estatísticas a registrar:** `subagent_tokens` (da notificação), `input_tokens`, `cache_creation`, `cache_read`, `output_tokens`, modelo usado.

**2026-06-29 — 1ª execução — realizada — ✅ APROVADA**
- Resposta: `{"ok": true}` — JSON válido, sem prosa, `tool_uses: 0`
- Duração: 1.245s
- `subagent_tokens`: 12.603
- `input_tokens`: 21
- Modelo: ⚠️ não especificado explicitamente — herdou da sessão pai (claude-sonnet-4-6)

**2026-06-29 — 2ª execução — realizada — ✅ APROVADA**

**Contexto de invocação:**
- Sessão iniciada com `/clear` — contexto zerado antes do eval
- Prompt enviado ao Claude Code (sessão pai):
  > `leia só /root/eval-2a-runner.md e execute. não leia wiki nem outros arquivos.`
- Runner lido: `/root/eval-2a-runner.md`
- Modelo declarado explicitamente via `model: "sonnet"` ✅

| Métrica | 1ª execução | 2ª execução | Diferença |
|---|---|---|---|
| contexto da sessão pai | sessão com histórico | `/clear` (zerado) | — |
| input_tokens (prompt ao subagente) | 21 | 3 | sessão zerada = menos tokens de contexto pai |
| cache_creation (system prompt interno) | não registrado | 12.599 | overhead base de qualquer subagente genérico |
| cache_read | não registrado | 0 | sem histórico anterior para reaproveitar |
| output_tokens | não registrado | 1 | |
| subagent_tokens (total) | 12.603 | 12.603 | overhead de base domina — prompt quase não importa |
| tool_uses | 0 | 0 ✅ | |
| duration_ms | 1.245 | 1.807 | variação normal de rede |

⚠️ **Lições desta comparação:**
1. O system prompt interno do Claude Code (~12.599 tokens) domina o custo — o prompt enviado (3 tokens) é irrelevante no total.
2. `/clear` antes do eval elimina `cache_read` da sessão pai e reduz `input_tokens` do subagente — condição mais limpa e reproduzível para evals.
3. Para reproducibilidade: sempre iniciar nova sessão com `/clear` e usar `/root/eval-2a-runner.md` como runner autocontido.

---

### Eval 2-B — `auditor-pasta` retorna JSON vazio? (sem lógica de auditoria)

Só inicia após Eval 2-A aprovado. Agora entra o `auditor-pasta` pela primeira vez — mas sem nenhum conteúdo para auditar. Confirma que o agente nomeado spawna, carrega seu system prompt e retorna JSON válido com `findings: []`.

**Runner:** `/root/eval-2b-runner.md` — autocontido, mesmo padrão do 2-A.

**Como executar (nova sessão):**
1. Abrir nova sessão do Claude Code
2. `/clear` — zerar contexto
3. Enviar exatamente: `leia só /root/eval-2b-runner.md e execute. não leia wiki nem outros arquivos.`

**De onde vêm os tokens (estimativa corrigida pós Eval 2-A):**

| Parte | Tokens estimados | Observação |
|---|---|---|
| System prompt interno Claude Code | ~12.000 | overhead de base confirmado no 2-A |
| System prompt do `auditor-pasta.md` | ~950 | arquivo tem ~137 linhas |
| Prompt enviado | ~15 | |
| Resposta (JSON vazio + _meta) | ~60 | |
| **Total estimado** | **~13.025** | estimativa anterior (~1.025) ignorava o overhead base |

**Prompt enviado ao subagente:**
> `Pasta: test/. Sem arquivos. Não faça Read calls. Retorne o JSON de caso sem findings.`

**Resposta esperada (estrutura mínima):**
```json
{
  "folder": "test/",
  "agent": "auditor-pasta",
  "findings": [],
  "_meta": {
    "files_read": [],
    "read_calls": 0,
    "approx_chars_read": 0,
    "limit_reached": false
  }
}
```

**Critérios:**
- [x] Agente spawna sem erro
- [x] Resposta é JSON válido
- [x] Campos presentes: `folder`, `findings`, `agent`, `_meta`
- [x] `agent == "auditor-pasta"`
- [x] `findings == []`
- [x] `_meta.read_calls == 0`
- [x] `_meta.limit_reached == false`
- [x] Nenhuma prosa fora do JSON

**Critérios a serem observados:** JSON válido? todos os campos presentes? `findings: []`? `read_calls == 0`? sem prosa?

**Estatísticas a registrar:** `subagent_tokens`, `input_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`, `output_tokens`, `duration_ms`.

**2026-06-29 — 1ª execução — realizada**

```
=== Eval 2-B — Resultado ===
Modelo usado:         claude-sonnet-4-6
Resposta bruta:       Agent type 'auditor-pasta' not found.
                      Available agents: claude, claude-code-guide,
                      Explore, general-purpose, Plan, statusline-setup
JSON válido:          ❌
folder presente:      ❌
findings == []:       ❌
agent correto:        ❌
_meta presente:       ❌
read_calls == 0:      ❌
limit_reached false:  ❌
Sem prosa:            ❌
tool_uses:            0 (spawn falhou antes de qualquer execução)

=== Estatísticas ===
subagent_tokens:              0
input_tokens:                 —
cache_creation_input_tokens:  —
cache_read_input_tokens:      —
output_tokens:                —
duration_ms:                  —

STATUS: ❌ REPROVADO
Motivo: O agente nomeado 'auditor-pasta' não está registrado no sistema.
        A ferramenta Agent só aceita subagent_type pré-definidos pelo harness
        (claude, claude-code-guide, Explore, general-purpose, Plan, statusline-setup).
        O arquivo .claude/agents/auditor-pasta.md existe mas o harness não o
        carrega automaticamente por nome — subagent_type customizado não é suportado.
        O spawn falhou com erro antes de qualquer execução.
```

**Causa raiz:** o mecanismo de subagentes nomeados via `.claude/agents/` não funciona como assumido. O harness não mapeia `subagent_type: "auditor-pasta"` para o arquivo `.claude/agents/auditor-pasta.md`. O system prompt do agente não é carregado automaticamente pelo nome. O arquivo existe e tem frontmatter correto — o problema é no harness, não no arquivo.

**Decisão para 2ª tentativa:** usar agente genérico (`subagent_type: "claude"`) com o system prompt do `auditor-pasta.md` injetado inline no campo `prompt`. Isso valida o contrato de output (que é o que o 2-B precisa provar), mas **não** valida se o harness carrega o arquivo `.claude/agents/` automaticamente.

**Pendência separada:** investigar se e como o harness mapeia `.claude/agents/<nome>.md` para a ferramenta `Agent` — pode ser que só funcione via `/agents` na UI, que precise de versão diferente do harness, ou que a sintaxe do `subagent_type` seja diferente. Essa investigação fica para depois dos evals 2-B e 2-C aprovados.

**2026-06-29 — 2ª execução — realizada (v2 Opção A: system prompt inline)**

**Contexto de invocação:**
- Sessão iniciada com `/clear` — contexto zerado antes do eval
- Prompt enviado ao Claude Code (sessão pai):
  > `leia só /root/eval-2b-runner.md e execute. não leia wiki nem outros arquivos.`
- Runner lido: `/root/eval-2b-runner.md`
- Modelo declarado explicitamente via `model: "sonnet"` ✅
- Mecanismo: agente genérico (`subagent_type` omitido) com system prompt do `auditor-pasta.md` injetado inline no campo `prompt`

```
=== Eval 2-B — Resultado (v2 — Opção A) ===
Modelo usado:         claude-sonnet-4-6
Resposta bruta:       {"folder":"test/","agent":"auditor-pasta","findings":[],"_meta":{"files_read":[],"read_calls":0,"approx_chars_read":0,"limit_reached":false}}
JSON válido:          ✅
folder presente:      ✅
findings == []:       ✅
agent correto:        ✅
_meta presente:       ✅
read_calls == 0:      ✅
limit_reached false:  ✅
Sem prosa:            ✅
tool_uses:            0  (esperado: 0)

=== Estatísticas ===
subagent_tokens:              12.935
input_tokens:                 1
cache_creation_input_tokens:  269
cache_read_input_tokens:      110.151
output_tokens:                83
duration_ms:                  2.468

STATUS: ✅ APROVADO
```

⚠️ **Lição:** `cache_read_input_tokens` de 110K indica que o system prompt interno do Claude Code estava cacheado da sessão pai (mesmo após `/clear`, o cache de projeto persiste entre sessões). O custo real de input foi praticamente zero — o overhead de base (~12K) foi servido do cache a ~10× menos custo.

**❌ Metodologia de tokens com falha — 3ª execução necessária**

O `tail -3` do Passo 3 capturava entradas misturadas de pai e subagente sem distinção. Evidência:

| Eval | output_tokens das entradas capturadas | O que eram |
|---|---|---|
| 2-A | 1 token | chamada do subagente (`{"ok":true}`) |
| 2-B | 83 / 207 / 287 tokens | chamadas da sessão pai |

Os 110K `cache_read` são da sessão pai acumulando contexto (system reminders injetados pelo harness: deferred tools, skills, agents, MEMORY.md). O subagente em si custou 12.935 tokens — idêntico ao 2-A. Os dois evals tiveram custo de subagente equivalente; o que variou foi qual entrada o `tail -3` capturou por acidente.

**Fix aplicado no runner (v3):** usar `ls -t *.jsonl | head -1 | xargs grep` para capturar TODAS as entradas do JSONL da sessão atual em ordem cronológica, com soma total. Arquivo atualizado: `/root/eval-2b-runner.md`.

**Discrepância não explicada entre 2-A e 2-B:**
Giovani usou o mesmo procedimento nos dois evals (`/clear` + prompt imediato, sem fechar o terminal). O 2-A mostrou `cache_read: 0`; o 2-B mostrou `cache_read: 110K`. Tentei explicar por diferença de turns na sessão pai, por TTL de cache e por qual entrada o `tail -3` capturou — nenhuma explicação resistiu ao contraditório. A causa real permanece desconhecida. O assistente reconheceu que estava especulando incorretamente.

**Procedimento para 3ª execução (acordado com Giovani):**
Fechar o terminal completamente → reabrir → `/clear` → enviar prompt. Objetivo: garantir sessão nova de verdade (novo JSONL, cache do zero) para eliminar variáveis desconhecidas e obter medição limpa e reproduzível.

**2026-06-29 — 3ª execução — realizada (v3: metodologia JSONL)**

**Contexto de invocação:**
- Terminal fechado completamente → reaberto → `/clear` → prompt enviado
- Runner lido: `/root/eval-2b-runner.md`
- Modelo declarado explicitamente via `model: "sonnet"` ✅
- Mecanismo: agente genérico com system prompt inline

```
=== Eval 2-B — Resultado (v3) ===
Resposta bruta:  {"folder":"test/","agent":"auditor-pasta","findings":[],"_meta":{"files_read":[],"read_calls":0,"approx_chars_read":0,"limit_reached":false}}
JSON válido:     ✅  |  findings == []:  ✅  |  read_calls == 0:  ✅  |  tool_uses: 0

=== Estatísticas (JSONL — 3 chamadas reais, harness grava 3 registros cada) ===
Turno 1 (ler runner + spawn):  cr=12.019  cc=8.906  in=3   out=124
Turno 2 (notif + bash + rel):  cr=20.925  cc=3.120  in=1   out=552
Turno 3 (extra por bash):      cr=24.045  cc=787    in=1   out=56
  → cache_read real: 57K  |  soma bruta (triplicada): 122K

subagent_tokens: 12.830  |  tool_uses: 0  |  duration_ms: 1.884

STATUS: ❌ REPROVADO
```

**Motivo da reprovação:** consumo inaceitável na sessão pai (57K cache_read reais, reportados como 122K por triplicação do JSONL). O Passo 3 (extração JSONL via bash) gerou um turno extra na sessão pai e tool calls desnecessários, acumulando contexto. Além disso, a metodologia de soma das 7 entradas brutas triplicou os números, gerando relatório enganoso. O subagente em si está correto (12.830 tokens, JSON válido) — o problema é o runner.

**Causa raiz dupla:**
1. **Runner design flaw:** Passo 3 (bash JSONL) e Passo 4 (Python via bash) adicionam tool calls no turno 2, aumentam contexto e criaram turno extra (turno 3). O passo de medição de custo AUMENTOU o custo.
2. **Artefato de medição:** JSONL tem 3 registros por chamada de API (request/response/final). Somar todos os registros triplica os números. `cache_read` de 57K ficou reportado como 122K.

**Lições:**
- O `cache_read` do pai (~12K/turno) é overhead do harness (50+ deferred tools, skills, MEMORY.md injetados a cada turno) — inevitável e não é critério de falha.
- A métrica relevante é `subagent_tokens` da notificação — ela é direta, sem artefatos.
- Qualquer bash call no turno 2 gera mais tool results no contexto, aumentando o `cache_read` do turno seguinte.

**Decisão para 4ª execução (v4):**
- Remover Passo 3 (JSONL extraction) inteiramente
- Remover Passo 4 (Python bash) — validação feita inline, sem ferramentas
- Turno 2 sem nenhum tool call: apenas texto com checks e relatório
- Adicionar critério formal: `subagent_tokens < 15.000`
- Fluxo final: 2 turnos, 0 tool calls no turno 2

**2026-06-29 — 4ª execução — ⛔ TOTALMENTE INVALIDADA E DESCONSIDERADA (v4: validação inline)**
*(invalidado retroativamente: runner v4 omitiu tokens da sessão pai — viola a regra obrigatória de transparência de custos)*
**⛔ TODO O CONTEÚDO DESTA EXECUÇÃO ESTÁ INVALIDADO. NÃO CITAR. NÃO USAR COMO REFERÊNCIA. NÃO COMPARAR COM OUTRAS EXECUÇÕES. FOI FEITO ERRADO.**

**Contexto de invocação:**
- Terminal fechado completamente → reaberto → `/clear` → prompt enviado
- Runner lido: `/root/eval-2b-runner.md` (v4)
- Modelo declarado explicitamente via `model: "sonnet"` ✅
- Mecanismo: agente genérico (sem `subagent_type`) com system prompt do `auditor-pasta.md` injetado inline
- Turno 2: zero tool calls — validação inline a partir da notificação

```
=== Eval 2-B — Resultado (v4) ===
Modelo usado:         claude-sonnet-4-6
Resposta bruta:       {"folder":"test/","agent":"auditor-pasta","findings":[],"_meta":{"files_read":[],"read_calls":0,"approx_chars_read":0,"limit_reached":false}}
JSON válido:          ✅
folder presente:      ✅
findings == []:       ✅
agent correto:        ✅
_meta presente:       ✅
read_calls == 0:      ✅
limit_reached false:  ✅
Sem prosa:            ✅
tool_uses:            0  (esperado: 0)

=== Estatísticas — subagente (da notificação) ===
subagent_tokens:  12.830  (limite: < 15.000) ✅
tool_uses:        0
duration_ms:      2.427

STATUS: ✅ APROVADO
```

**Lições desta execução:**
- Remover os passos de extração JSONL e Python eliminou o turno extra e os tool calls no pai — fluxo ficou em exatamente 2 turnos, 0 tool calls no turno 2.
- `subagent_tokens` de 12.830 confirma o padrão: overhead base do harness (~12.6K) domina; conteúdo do prompt é marginal.
- Validação inline (sem ferramentas) é suficiente e mais limpa — a notificação entrega tudo que é necessário.
- ⚠️ Invalidado: tokens do pai não foram reportados — viola a regra obrigatória de transparência de custos adicionada após esta execução.

**2026-06-29 — 5ª execução — realizada (v5: tokens do pai via JSONL deduplificado)**

**Contexto de invocação:**
- Terminal fechado completamente → reaberto → `/clear` → prompt enviado
- Prompt enviado ao Claude Code (sessão pai):
  > `leia só /root/eval-2b-runner.md e execute. não leia wiki nem outros arquivos.`
- Runner lido: `/root/eval-2b-runner.md` (v5)
- Modelo declarado explicitamente via `model: "sonnet"` ✅
- Mecanismo: agente genérico (sem `subagent_type`) com system prompt do `auditor-pasta.md` injetado inline
- Turno 2: zero tool calls — validação inline a partir da notificação
- Turno 3: bash/Python com deduplicação correta (primeiro de cada grupo de 3 registros idênticos), última entrada excluída

```
=== Eval 2-B — Resultado (v5) ===
Modelo usado:         claude-sonnet-4-6
Resposta bruta:       {"folder":"test/","agent":"auditor-pasta","findings":[],"_meta":{"files_read":[],"read_calls":0,"approx_chars_read":0,"limit_reached":false}}
JSON válido:          ✅
folder presente:      ✅
findings == []:       ✅
agent correto:        ✅
_meta presente:       ✅
read_calls == 0:      ✅
limit_reached false:  ✅
Sem prosa:            ✅
tool_uses:            0  (esperado: 0) ✅

=== Estatísticas — subagente (da notificação) ===
subagent_tokens:  12.830  (limite: < 15.000) ✅
tool_uses:        0
duration_ms:      2.546

=== Estatísticas — sessão pai (JSONL deduplificado) ===
Chamadas do eval:     2 entradas marcadas EVAL
input_tokens:         4
cache_creation:       3.042
cache_read:           41.850
output_tokens:        791
(turno de extração excluído — última entrada do JSONL)

STATUS: realizado
```

**Lições desta execução:**
- Deduplicação correta: JSONL tinha 3 entradas únicas (2 EVAL + 1 EXTRAÇÃO). Excluir a última eliminou o custo do próprio turno de medição.
- `cache_read` do pai (41.850) é overhead do harness (deferred tools, skills, MEMORY.md) — inevitável, não é critério de falha.
- `cache_creation` de apenas 3.042 indica que a maior parte do system prompt já estava cacheada de sessões anteriores.
- Fluxo final: 3 turnos (spawn + notif/validação inline + extração JSONL), conforme spec v5.

**Questão em aberto — próximo run:**
`cache_read` do pai (41.850) ainda não tem baseline de mínimo possível. O principal driver são os 3 turnos da sessão pai: a cada turno o harness reinjecta ~14K tokens de contexto (deferred tools, skills, MEMORY.md). Para descobrir o mínimo real, testar uma variante com 2 turnos — mesclar Passos 3+4 em um único turno com 1 tool call (bash) — e comparar o `cache_read` resultante com os 41.850 da v5. Se o `cache_read` cair ~1/3, a redução de turno compensa o custo do tool call extra.

**2026-06-29 — ENCERRADO — arquitetura de subagentes descartada**
Esta seção não tem resultado APROVADO nem REPROVADO. Todas as execuções (v1 a v5) testavam o mecanismo Agent tool (subagente). Esse mecanismo foi descartado: o overhead fixo de ~12.8K tokens por spawn é inaceitável. O auditor passa a executar diretamente na sessão principal, sem subagentes. Eval 2-C foi redefinido para a nova arquitetura.

---

### Eval 2-C — Esta sessão detecta problema em arquivo sintético? (nova arquitetura — sem subagentes)

**Arquitetura:** sem subagentes. O Claude Code da sessão corrente lê o arquivo diretamente e reporta findings. Nenhum Agent tool. Nenhum spawn. Nenhum overhead de harness de subagente.

**Por que esta é a arquitetura correta:** o overhead fixo de ~12.8K tokens por spawn (confirmado em Eval 2-A e 2-B) torna subagentes inviáveis. O auditor executa inteiramente na sessão principal.

**Arquivo sintético:** `/tmp/eval-2c-test.md` — frontmatter com `status` ausente (problema plantado). Criado antes desta sessão encerrar.

**Transparência de custo:** sem subagente. Tokens capturados via extração JSONL na mesma resposta (one-shot): o modelo faz Read → gera checklist em texto → faz Bash de extração JSONL → compila relatório final. O Bash exclui o último grupo do JSONL (o próprio passo de extração) e reporta os turnos anteriores separados.

**Critérios de aprovação:**
- [ ] 1 Read call — apenas `/tmp/eval-2c-test.md`
- [ ] `status` identificado como ausente
- [ ] Severidade `critico` reportada
- [ ] Tamanho em caracteres do arquivo informado na resposta
- [ ] Nenhum arquivo extra lido
- [ ] Tokens da sessão capturados e reportados (input, cache_creation, cache_read, output por turno)

**Critérios a serem observados:** campo correto identificado? problema detectado? exatamente 1 Read call? tamanho reportado? tokens capturados?

**Prompt exato para a próxima sessão (copiar e colar):**

> ⚠️ Copiar o bloco abaixo **sem editar nenhum caractere**. Nas execuções 2ª e 3ª o script chegou corrompido porque foi copiado de fonte externa. A versão correta é esta.

```
Eval 2-C — sem subagentes. Execute em sequência sem desviar:

Passo 1 — Leia /tmp/eval-2c-test.md (único arquivo permitido). Verifique se o frontmatter contém todos os campos obrigatórios: type, tags, title, description, timestamp, status. Para cada campo ausente, reporte: nome do campo, severidade "critico", sugestão de correção. Informe o tamanho em caracteres do arquivo lido. Nenhum outro arquivo, nenhum outro tool call neste passo.

Passo 2 — Execute este bash:
python3 -c "
import json, glob, os
files = sorted(glob.glob(os.path.expanduser('~/.claude/projects/-root/*.jsonl')), key=os.path.getmtime, reverse=True)
if not files: print('JSONL nao encontrado'); exit(1)
entries = []
for line in open(files[0]):
    try:
        e = json.loads(line.strip())
        u = e.get('message', {}).get('usage')
        if not u: continue
        key = (u.get('input_tokens',0), u.get('cache_creation_input_tokens',0), u.get('cache_read_input_tokens',0), u.get('output_tokens',0))
        if not entries or entries[-1][0] != key:
            entries.append((key, u))
    except: pass
turns = entries[:-1]
for i, (k, u) in enumerate(turns, 1):
    print(f'T{i}: in={u.get(\"input_tokens\",0)} cc={u.get(\"cache_creation_input_tokens\",0)} cr={u.get(\"cache_read_input_tokens\",0)} out={u.get(\"output_tokens\",0)}')
"

Passo 3 — Compile o resultado final com o checklist do Passo 1 e o output do Passo 2. Encerre com: STATUS: aguardando definição do Giovani
```

**Como executar:**
1. Fechar terminal completamente → reabrir
2. `/clear`
3. Colar o prompt acima
4. Registrar o resultado aqui antes de qualquer outra ação — status só após confirmação do Giovani

**2026-06-29 — 1ª execução — ⚠️ INCOMPLETA (tokens do pai não capturados)**

**Contexto de invocação:**
- Sessão sem `/clear` explícito — AGENTS.md e evals lidos na mesma sessão após o eval
- Prompt enviado diretamente ao Claude Code (sem runner separado)
- Arquitetura: sem subagentes — 1 Read call direto na sessão principal
- Nenhum Agent tool, nenhum spawn, nenhum overhead de harness de subagente

```
=== Eval 2-C — Checklist ===
Read calls:           1 (apenas /tmp/eval-2c-test.md)
Arquivo extra lido:   nenhum ✅
Campo ausente:        status ✅
Severidade reportada: critico ✅
Tamanho reportado:    231 caracteres ✅
Prosa fora do report: nenhuma ✅

Tokens da sessão pai: NÃO CAPTURADO ❌
Status final:         aguardando definição do Giovani
```

**Por que está incompleta:** tokens da sessão pai não foram rastreados — nenhum runner dedicado, nenhuma extração de JSONL. Quebra a regra obrigatória de rastreabilidade de custo. Execução a repetir com metodologia correta. Status não definido pelo agente — aguarda confirmação do Giovani.

**2026-06-29 — 2ª execução — ⚠️ INCOMPLETA (tokens do pai retornaram zeros)**

**Contexto de invocação:**
- Sessão com histórico (AGENTS.md e evals lidos antes do eval na mesma sessão)
- Prompt enviado diretamente no chat — sem runner separado, sem `/clear` antes
- Arquitetura: sem subagentes — 1 Read call direto na sessão principal
- Nenhum Agent tool, nenhum spawn

```
=== Eval 2-C — Checklist ===
Read calls:           1 (apenas /tmp/eval-2c-test.md) ✅
Arquivo extra lido:   nenhum ✅
Campo ausente:        status ✅
Severidade reportada: critico ✅
Tamanho reportado:    ~311 chars (contagem manual; arquivo tem 316 bytes UTF-8 = 311 chars Unicode) ✅
Prosa fora do report: nenhuma ✅

Tokens da sessão pai: NÃO CAPTURADO — script retornou zeros ❌
Status final:         aguardando definição do Giovani
```

**Por que está incompleta:** o script de extração JSONL usa `range(0, len(lines), 3)` para deduplicar — captura linhas em posições fixas (0, 3, 6...) que não necessariamente têm `message.usage`. Retornou `in=0 cc=0 cr=0 out=0` em todos os 4 turnos. Mesma falha de rastreabilidade de custo da 1ª execução, por razão diferente. Fix necessário no script antes de nova execução.

**Observação sobre tamanho:** a 1ª execução reportou 231 chars; a 2ª reportou ~311 (manual). O arquivo tem 316 bytes e 311 chars Unicode (5 chars multibyte: é, ç, ã, ú, í). O valor correto é 311 chars — a 1ª execução estava errada.

**2026-06-29 — 3ª execução — ⚠️ INCOMPLETA (tokens não capturados — SyntaxError no script)**

**Contexto de invocação:**
- Sessão com histórico (AGENTS.md e evals lidos na mesma sessão antes do prompt do eval)
- Prompt enviado diretamente no chat — sem runner separado, sem `/clear` antes
- Arquitetura: sem subagentes — 1 Read call direto na sessão principal

```
=== Eval 2-C — Checklist ===
Read calls:           1 (apenas /tmp/eval-2c-test.md) ✅
Arquivo extra lido:   nenhum ✅
Campo ausente:        status ✅
Severidade reportada: critico ✅
Tamanho reportado:    ~311 chars ✅
Prosa fora do report: nenhuma ✅

Tokens da sessão pai: NÃO CAPTURADO — SyntaxError no script ❌
Status final:         aguardando definição do Giovani
```

**Por que está incompleta:** o script do Passo 2 (salvo na seção "Prompt exato para a próxima sessão") contém uma corrupção no f-string: `cc={u.get(",0)}` — a chave `"cache_creation_input_tokens"` foi truncada, deixando `",0)` como literal inválido. Python retornou `SyntaxError: invalid syntax`. Nenhum dado de tokens foi capturado. O script precisa ser corrigido antes da 4ª execução.

**Passo 1 funcionou corretamente:**
- `status` identificado como ausente
- Severidade `critico` reportada
- Tamanho ~311 chars reportado (consistente com a 2ª execução)
- Sugestão: adicionar `status: draft` após `timestamp:`

**Fix necessário no prompt antes da 4ª execução:**
O script salvo no arquivo (seção "Prompt exato para a próxima sessão") está **correto** — `cache_creation_input_tokens` aparece sem truncamento. A corrupção (`cc={u.get(",0)}`) estava no texto que o usuário copiou de fonte externa, não no arquivo. Na 4ª execução: copiar o prompt diretamente desta seção, sem editar.

**2026-06-29 — 4ª execução — aguardando**

**O que mudou em relação à 3ª execução:**
- Causa raiz identificada: o usuário copiou uma versão corrompida do prompt (não do arquivo). A seção "Prompt exato" abaixo tem o script correto.
- Instrução adicionada: copiar o bloco ` ``` ` da seção "Prompt exato" sem editar — nenhum caractere a mais ou a menos.
- Procedimento reforçado: fechar terminal → reabrir → `/clear` → colar.

**Procedimento:**
1. Fechar terminal completamente → reabrir
2. `/clear`
3. Copiar o bloco de código da seção "Prompt exato para a próxima sessão" (linhas entre os ` ``` `) e colar sem editar

**2026-06-29 — 4ª execução — ⚠️ INCOMPLETA (procedimento errado + JSONL errado + interrompida)**

**Contexto de invocação:**
- Invocado via `/remote-control` skill — **viola o procedimento prescrito** (fechar terminal → reabrir → `/clear`)
- Sessão com histórico (wiki lida antes na mesma sessão) — não foi sessão limpa
- Arquitetura: sem subagentes — 1 Read call direto na sessão principal

```
=== Eval 2-C — Checklist (parcial) ===
Read calls:           1 (apenas /tmp/eval-2c-test.md) ✅
Arquivo extra lido:   nenhum no Passo 1 ✅
Campo ausente:        status ✅
Severidade reportada: critico ✅
Tamanho reportado:    NÃO REPORTADO EXPLICITAMENTE ❌  (valor real: 311 chars — confirmado)
Prosa fora do report: nenhuma ✅

Tokens da sessão pai: NÃO CAPTURADO — script rodou contra JSONL errado ❌
Passo 3:              NÃO EXECUTADO — usuário interrompeu ❌
Status final:         aguardando definição do Giovani
```

**Por que está incompleta:**
1. **Procedimento errado:** `/remote-control` não abre terminal novo nem executa `/clear` — sessão tinha histórico
2. **JSONL errado:** o script do Passo 2 foi rodado contra `829d9ab7...` (2ª entrada mais recente — sessão anterior), não contra o arquivo da sessão atual (`e92da6ce...`)
3. **Interrompida:** usuário disse "parou" antes do Passo 3 ser executado

**Lição:**
O `/remote-control` skill não satisfaz o procedimento de "fechar terminal → reabrir → `/clear`". O skill entrega o prompt dentro da sessão existente, com contexto acumulado. Para o Eval 2-C ser válido, o procedimento de sessão limpa é mandatório — o skill não é substituto.

**2026-06-29 — 5ª execução — ⚠️ INCOMPLETA (tamanho errado + tokens vazios — primeiro turno da sessão)**

**Contexto de invocação:**
- Invocado via `/remote-control` skill — viola o procedimento prescrito
- Eval foi o PRIMEIRO turno da sessão — JSONL sem entradas anteriores ao Passo 2
- Arquitetura: sem subagentes — 1 Read call direto na sessão principal

```
=== Eval 2-C — Checklist ===
Read calls:           1 (apenas /tmp/eval-2c-test.md) ✅
Arquivo extra lido:   nenhum ✅
Campo ausente:        status ✅
Severidade reportada: critico ✅
Tamanho reportado:    259 chars ❌  (correto: 311 chars — verificado com python3 len())
Prosa fora do report: nenhuma ✅

Tokens da sessão pai: NÃO CAPTURADO — bash retornou vazio ❌
Status final:         aguardando definição do Giovani
```

**Causa raiz — tokens vazios:** o eval foi o primeiro turno da sessão. O JSONL só grava a entrada do turno atual após o turno ser concluído. Quando o Passo 2 (bash) executou, a entrada ainda não estava gravada. Em primeiro turno, `entries[:-1]` = lista vazia → output vazio. O script estava correto; o procedimento estava errado.

**Causa raiz — tamanho errado:** o modelo estimou 259 chars sem contar via Python. O arquivo tem 311 chars Unicode (316 bytes UTF-8 — 5 caracteres multibyte: é, ç, ã, ú, í). Contagem manual de LLM é imprecisa; a próxima versão deve usar bash para medir o tamanho exato.

**Fix necessário para 6ª execução:** redesign em 2 turnos — T1 faz Read + análise + bash de tamanho; T2 faz bash de JSONL + compilação. Remove `entries[:-1]` do script (T2 não está no JSONL ainda quando o bash de T2 roda, então não há risco de capturar o próprio turno).

**Prompt para 6ª execução — 2 turnos separados:**

> ⚠️ São DOIS prompts distintos. Enviar T1 primeiro, aguardar resposta completa com "T1-CONCLUÍDO", depois enviar T2.

**Turno 1 (colar e enviar):**
```
Eval 2-C — Turno 1. Execute em sequência, sem desviar:

1. Leia /tmp/eval-2c-test.md (único Read permitido neste turno). Verifique se o frontmatter contém todos os campos obrigatórios: type, tags, title, description, timestamp, status. Para cada campo ausente, reporte: nome do campo, severidade "critico", sugestão de correção.

2. Execute este bash para obter o tamanho exato:
python3 -c "print(len(open('/tmp/eval-2c-test.md').read()))"

Encerre com: T1-CONCLUÍDO
```

**Turno 2 (enviar somente após receber "T1-CONCLUÍDO"):**
```
Eval 2-C — Turno 2. Execute o bash abaixo e compile o resultado final:

python3 -c "
import json, glob, os
files = sorted(glob.glob(os.path.expanduser('~/.claude/projects/-root/*.jsonl')), key=os.path.getmtime, reverse=True)
if not files: print('JSONL nao encontrado'); exit(1)
entries = []
for line in open(files[0]):
    try:
        e = json.loads(line.strip())
        u = e.get('message', {}).get('usage')
        if not u: continue
        key = (u.get('input_tokens',0), u.get('cache_creation_input_tokens',0), u.get('cache_read_input_tokens',0), u.get('output_tokens',0))
        if not entries or entries[-1][0] != key:
            entries.append((key, u))
    except: pass
for i, (k, u) in enumerate(entries, 1):
    print(f'T{i}: in={u.get(\"input_tokens\",0)} cc={u.get(\"cache_creation_input_tokens\",0)} cr={u.get(\"cache_read_input_tokens\",0)} out={u.get(\"output_tokens\",0)}')
"

Compile o resultado final com:
- checklist de campos do Turno 1
- tamanho em chars do Turno 1
- output do bash acima (tokens por turno)

Encerre com: STATUS: aguardando definição do Giovani
```

**Por que funciona:**
- T1 usa bash apenas para medir tamanho (sem leitura de outros arquivos — critério satisfeito)
- T2 roda com T1 já gravado no JSONL → `entries` retorna os dados de T1 e turnos anteriores
- `entries[:-1]` removido → sem exclusão que apagaria exatamente T1 (o turno que queremos medir)
- Funciona mesmo em sessão com histórico (mostra todos os turnos anteriores, inclusive contexto)

---

### Eval 3 — Detecção precisa: valor inválido (não só campo ausente)

Eval 2-C testou campo ausente. Este testa um problema diferente: campo presente com valor fora do enum válido. Confirma que a sessão detecta categorias de problema além da ausência simples.

**Arquivo sintético:** criar `/tmp/eval3-test.md` com `type: invalido` no frontmatter — todos os outros campos obrigatórios presentes e corretos.

**Critérios:**
- [ ] 1 Read call — apenas `/tmp/eval3-test.md`
- [ ] Finding identifica `type` como o campo problemático (não outro)
- [ ] Severidade `critico` reportada
- [ ] Sugestão inclui pelo menos um valor válido do enum (`system`, `tool`, `concept`, `procedure`, etc.)
- [ ] Tamanho em caracteres reportado (verificável com `python3 -c "print(len(open('/tmp/eval3-test.md').read()))"`)
- [ ] Tokens da sessão capturados via JSONL

**Critérios a serem observados:** campo correto identificado? valor válido sugerido? exatamente 1 Read call? tokens capturados?

---

### Eval 4 — Fail-fast: arquivo inexistente não resulta em findings vazio silencioso

Testa o Princípio de Redesign #1 (fail-fast, não silencioso). Na v1, agentes que falhavam retornavam `findings: []` silenciosamente — o erro era invisível até o final. Na nova arquitetura, se o arquivo-alvo não existe, a sessão deve reportar o problema explicitamente.

**Critérios:**
- [ ] Sessão tenta auditar `/tmp/eval4-nao-existe.md` (arquivo que não existe)
- [ ] Sessão reporta explicitamente que o arquivo não foi encontrado — erro visível, não silêncio
- [ ] Nenhum finding gerado para arquivo inexistente
- [ ] Tokens da sessão capturados via JSONL

**Critérios a serem observados:** sessão reportou erro explícito? retornou findings vazios sem mensagem? sessão avançou ou parou após o erro?

---

### Eval 5 — Sessão aguenta esperar o Telegram?

O ponto mais crítico da arquitetura. Sessão Claude Code envia mensagem com botões e aguarda callback — não pode expirar durante a espera.

- [ ] Sessão envia mensagem com botões via Bash (curl) pro Telegram
- [ ] Sessão aguarda callback via polling ativo (loop `curl getUpdates` com sleep, não espera passiva — Claude Code não suspende sessão)
- [ ] Resposta do botão é recebida e processada corretamente dentro da sessão

**Critérios a serem observados:** loop completo (envio → polling → recebimento do callback) dentro de uma sessão Claude Code sem interrupção?

---

### Eval 6 — Sessão com dados reais: pasta `todo/` (rampa antes do completo)

Primeiro contato com dados reais da wiki. Sessão lê e audita diretamente a pasta `todo/` (menor pasta real). Registrar custo e tempo antes de processar todas as pastas.

**Critérios:**
- [ ] Sessão lê corretamente os arquivos de `todo/` + `AGENTS.md`
- [ ] Findings têm todos os campos obrigatórios
- [ ] Read calls ≤ número de arquivos da pasta + 1 (AGENTS.md)
- [ ] Wall clock registrado (tempo total do início ao relatório)
- [ ] Tokens da sessão capturados via JSONL

**Critérios a serem observados:** input abaixo de 20K? output abaixo de 1K? findings coerentes com o conteúdo real da pasta?

---

### Eval 7 — Sessão processa todas as pastas em série

Todas as pastas da wiki, uma por vez, após Eval 6. A mesma sessão lê cada pasta sequencialmente e consolida os findings.

**Critérios:**
- [ ] Sessão processa cada pasta e produz findings por pasta
- [ ] Read calls plausíveis por pasta (≤ nº de arquivos + 1 para AGENTS.md)
- [ ] Nenhuma pasta retorna findings vazios de forma suspeita quando há problemas óbvios
- [ ] Wall clock e token delta registrados por pasta
- [ ] Tokens da sessão capturados via JSONL

**Critérios a serem observados:** input por pasta abaixo de 30K? output por pasta abaixo de 2K? findings coerentes em todas as pastas?

---

### Eval 8 — Verificações cross-folder: sobreposição e links quebrados

**Critérios:**
- [ ] Verificação de sobreposição semântica produz findings sem falsos positivos óbvios
- [ ] Verificação de wikilinks detecta pelo menos um link quebrado se houver
- [ ] Read calls plausíveis para varredura cross-folder
- [ ] Tokens da sessão capturados via JSONL

**Critérios a serem observados:** findings de sobreposição coerentes? links quebrados detectados corretamente? custo dentro do esperado?

---

### Eval 9 — Correção aplicada corretamente

O maior risco técnico: LLMs normalizam espaços e quebras de linha — se `old_string` não bater exato com o arquivo, a edição falha.

**Primeiro: arquivo sintético**
Criar `/tmp/eval10-test.md` com 3 linhas e problema conhecido. Verificar que a edição é aplicada antes de tocar em qualquer arquivo real.

**Critérios:**
- [ ] `old_string` é substring exata do arquivo sintético (verificar com `grep -F`)
- [ ] Edição aplicada com sucesso no arquivo sintético
- [ ] Campo `file` usa caminho relativo ao `WIKI_DIR` (ex: `systems/hermes.md`, não absoluto)
- [ ] Testar com dois findings no mesmo arquivo (ordem de aplicação)

**Critérios a serem observados:** edição aplicou sem erro de "old_string não encontrado" nos dois cenários?

---

### Eval 10 — Dry-run completo

Fluxo completo de análise, sem aplicar nenhuma edição.

**Critérios:**
- [ ] Sessão processa todas as pastas e verificações cross-folder em sequência
- [ ] Consolidação de findings sem erro
- [ ] Custo total de tokens dentro do esperado
- [ ] Resumo executivo chega no Telegram com findings reais

**Critérios a serem observados:** findings coerentes? custo compatível com Evals 7 e 8? resumo chegou no Telegram?

---

### Eval 11 — Run completo real

Apenas após todos os evals anteriores passarem e com autorização explícita.

- [ ] Evals 1–10 todos aprovados e documentados aqui
- [ ] Giovani autorizou este run
- [ ] `WIKI_DIR` apontando para `/root/wiki`
- [ ] Log de execução em `/var/log/auditor-wiki.log`

---


## Conexões

- [[wiki/concepts/automacao-principios.md|Princípios de Automação e Testes]] — documento geral do qual estes evals são aplicação prática
- [[wiki/procedures/auditor-wiki.md|Auditor da Wiki]] — o script que este documento valida; contém a análise de tokens da primeira execução
- [[raw/agents-cli-README.md|Google Agents CLI — README oficial]] — fonte dos princípios de skills e eval gates
- [[raw/akshay-pachaar-karpathy-agentic-engineering-tooling.md|Karpathy's Agentic Engineering Finally Has Proper Tooling]] — fonte do framework spec design → eval loops → deploy
- [[wiki/todo/proximos-passos.md|Próximos passos]] — onde os gates pendentes podem virar itens acionáveis
