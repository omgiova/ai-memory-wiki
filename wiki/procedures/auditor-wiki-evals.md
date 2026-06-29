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

**Critério de aprovação:** revisão manual do arquivo do subagente e do system prompt.

**✅ APROVADO — 2026-06-29**
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

**Critério de reprovação:** JSON inválido | prosa fora do JSON.

**Critério de aprovação:** `{"ok": true}` parseável, sem prosa.

**Estatísticas a registrar:** `subagent_tokens` (da notificação), `input_tokens`, `cache_creation`, `cache_read`, `output_tokens`, modelo usado.

**✅ APROVADO — 2026-06-29 (1ª execução)**
- Resposta: `{"ok": true}` — JSON válido, sem prosa, `tool_uses: 0`
- Duração: 1.245s
- `subagent_tokens`: 12.603
- `input_tokens`: 21
- Modelo: ⚠️ não especificado explicitamente — herdou da sessão pai (claude-sonnet-4-6)

**✅ APROVADO — 2026-06-29 (2ª execução)**

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

**Critério de reprovação:** JSON inválido | prosa fora do JSON | campo obrigatório ausente | `findings` não vazio | `read_calls != 0` | `agent != "auditor-pasta"`.

**Critério de aprovação:** JSON válido, todos os campos presentes, `findings: []`, `read_calls == 0`, sem prosa.

**Estatísticas a registrar:** `subagent_tokens`, `input_tokens`, `cache_creation_input_tokens`, `cache_read_input_tokens`, `output_tokens`, `duration_ms`.

**❌ REPROVADO — 2026-06-29**

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

**✅ APROVADO — 2026-06-29 (2ª execução — v2 Opção A: system prompt inline)**

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

**⏳ 3ª execução pendente** com runner v3.

---

### Eval 2-C — `auditor-pasta` detecta problema? (conteúdo inline com problema plantado)

Só inicia após Eval 2-B aprovado. Primeiro teste de lógica de auditoria — conteúdo sintético passado inline, problema conhecido e plantado.

**Prompt enviado ao subagente:**
> `Pasta: test/. Arquivo: test/test.md. Conteúdo inline abaixo — não faça Read calls.`
>
> ```
> ---
> type: system
> title: Arquivo de Teste
> description: Arquivo mínimo para eval.
> ---
> Conteúdo mínimo.
> ```

Problema plantado: campo `status` ausente. O agente deve detectar e retornar finding `critico`.

**Critérios:**
- [ ] Resposta é JSON válido
- [ ] Campos presentes: `folder`, `findings`, `agent`, `_meta`
- [ ] `_meta.read_calls == 0`
- [ ] `_meta.limit_reached == false`
- [ ] Finding F1: `severity: "critico"`, `problem` menciona ausência de `status`
- [ ] Nenhuma prosa fora do JSON

**Critério de reprovação:** `output_tokens > 500` | `read_calls > 0` | JSON inválido | prosa fora do JSON | finding ausente ou severidade errada.

**Critério de aprovação:** JSON válido com `read_calls == 0` e finding F1 correto.

**Estatísticas a registrar:** `input_tokens`, `cache_read_input_tokens`, `output_tokens`.

---

### Eval 3 — `_meta` reporta corretamente? (1 read sintético)

Criar `/tmp/eval3-test.md` com ~5 linhas e problema conhecido. Invocar subagente pedindo que leia o arquivo. Verificar que `_meta` reflete exatamente o que foi feito — não o que o subagente acha que fez.

**Critérios:**
- [ ] `_meta.files_read` lista `eval3-test.md`
- [ ] `_meta.read_calls == 1`
- [ ] `_meta.approx_chars_read` condizente com tamanho real do arquivo (± 20%)
- [ ] `_meta.limit_reached == false`
- [ ] Finding esperado presente (problema plantado no arquivo)

**Critério de reprovação:** `read_calls != 1` | `approx_chars_read` implausível | `_meta` ausente ou incompleto.

---

### Eval 4 — Erro propaga ou engole silencioso? (era Eval 3)

Input sintético inline. Invocar subagente **sem** instrução de formato JSON (remover exemplo do JSON e instrução de prosa do prompt). Verificar que a sessão principal reage.

**Mecanismo de detecção — o que a sessão principal executa:**
```python
result = agent_response  # string retornada pelo Agent tool
try:
    json.loads(result)
    # se chegou aqui: subagente obedeceu mesmo sem instrução — registrar como observação
except json.JSONDecodeError:
    print("PASSOU: erro detectado")
    print("Output (primeiros 300 chars):", result[:300])
    # não avança para nenhum próximo passo
```

**Critérios:**
- [ ] `json.loads(result)` lança exceção (subagente produziu prosa, como esperado)
- [ ] Sessão principal captura a exceção e para
- [ ] Os primeiros 300 chars do output são logados e visíveis
- [ ] Nenhuma ação downstream é disparada após a falha

**Critério de aprovação:** falha detectada, logada, sessão não avança.

---

### Eval 5 — Sessão aguenta esperar o Telegram? (era Eval 4)

O ponto mais crítico da arquitetura. Sessão Claude Code envia mensagem com botões e aguarda callback — não pode expirar durante a espera.

- [ ] Sessão envia mensagem com botões via Bash (curl) pro Telegram
- [ ] Sessão aguarda callback via polling ativo (loop `curl getUpdates` com sleep, não espera passiva — Claude Code não suspende sessão)
- [ ] Resposta do botão é recebida e processada corretamente dentro da sessão

**Critério de aprovação:** loop Telegram completo (envio → polling → recebimento do callback) dentro de uma sessão Claude Code sem interrupção.

---

### Eval 6 — Um agente, dados reais (rampa antes do serial)

Primeiro contato com dados reais da wiki. Um agente, pasta `todo/` (menor pasta real). Registrar custo e tempo antes de escalar para todos os agentes.

**Critérios:**
- [ ] Agente retorna JSON válido
- [ ] `_meta.read_calls` ≤ 3 (AGENTS.md + arquivos da pasta)
- [ ] `_meta.limit_reached == false`
- [ ] Wall clock registrado (tempo total do spawn à resposta)
- [ ] Token delta registrado (input + output via JSONL delta da sessão)

**Critério de reprovação:** `input_tokens > 20K` | `output_tokens > 1K` | `read_calls > 3` | `limit_reached == true`.

---

### Eval 7 — Agentes de pasta em série (era Eval 5)

Todos os agentes de pasta, um por vez, após Eval 6 aprovado com dados reais.

**Critérios:**
- [ ] Cada agente retorna JSON válido
- [ ] `_meta.read_calls` plausível por agente (≤ nº de arquivos da pasta + 1 para AGENTS.md)
- [ ] `_meta.limit_reached == false` em todos
- [ ] Nenhum agente retorna `findings: []` de forma suspeita para pasta com problemas óbvios
- [ ] Wall clock e token delta registrados por agente

**Critério de reprovação por agente:** `input_tokens > 30K` | `output_tokens > 2K` | `limit_reached == true`.

**Critério de aprovação:** todos os agentes passam individualmente dentro dos limites.

---

### Eval 8 — Agentes Overlap e Links (era Eval 6)

- [ ] Agente Overlap retorna JSON válido sem falsos positivos óbvios
- [ ] Agente Links detecta pelo menos um link quebrado se houver
- [ ] Ambos retornam `_meta` com `read_calls` plausível e `limit_reached == false`

**Critério de aprovação:** os dois agentes retornam JSON válido dentro dos limites de custo.

---

### Eval 9 — Coordenador isolado (era Eval 7)

Alimentado com outputs reais dos Evals 7 e 8.

- [ ] Output é JSON válido com `executive_summary` e `findings`
- [ ] Campo `correctable` classificado coerentemente
- [ ] Nenhum finding duplicado
- [ ] Funciona com input misto (pasta com findings + pasta com `findings: []`)

**Critério de aprovação:** JSON válido, sem duplicatas.

---

### Eval 10 — Agente Corretor (era Eval 8)

O maior risco técnico: LLMs normalizam espaços e quebras de linha — se `old_string` não bater exato com o arquivo, a edição falha.

**Primeiro: arquivo sintético**
Criar `/tmp/eval10-test.md` com 3 linhas e problema conhecido. Verificar que a edição é aplicada antes de tocar em qualquer arquivo real.

- [ ] `old_string` é substring exata do arquivo sintético (verificar com `grep -F`)
- [ ] Edição aplicada com sucesso no arquivo sintético

**Depois: arquivo real**
- [ ] Campo `file` usa caminho relativo ao `WIKI_DIR` (ex: `systems/hermes.md`, não absoluto)
- [ ] Testar com dois findings no mesmo arquivo (ordem de aplicação)

**Critério de aprovação:** edição aplicada sem erro de "old_string não encontrado" nos dois cenários.

---

### Eval 11 — Dry-run completo (era Eval 9)

Fluxo completo de análise e coordenação, sem aplicar nenhuma edição.

- [ ] Todos os agentes retornam JSON válido em condições reais de paralelismo
- [ ] Coordenador consolida sem erro
- [ ] Custo total de tokens dentro do esperado
- [ ] Resumo executivo chega no Telegram com findings reais

**Critério de aprovação:** resumo executivo no Telegram com findings reais, nenhum erro de JSON.

---

### Eval 12 — Run completo real (era Eval 10)

Apenas após todos os evals anteriores passarem e com autorização explícita.

- [ ] Evals 1–11 todos aprovados e documentados aqui
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
