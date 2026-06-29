---
type: procedure
tags: [auditoria, eval, validacao, multi-agente, diagnostico, design]
title: Auditor da Wiki — Evals e Validação
description: Diagnóstico da primeira execução do auditor, lições dos raws agents-cli e Karpathy, e processo de eval estruturado com gates obrigatórios antes de qualquer run em produção.
timestamp: 2026-06-29T00:00:00-03:00
status: draft
---

# Auditor da Wiki — Evals e Validação

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

A implementação atual é um script bash (`/root/auditor-wiki-v1.sh`) com subagentes invocados via `claude` CLI. Nada nessa implementação é obrigatório — a ideia central (análise → aprovação granular → commit por finding) pode ser reimplementada de forma diferente.

Documentação completa da implementação atual: [[wiki/procedures/auditor-wiki.md|Auditor da Wiki]].

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

## Princípios de redesign

Derivados do diagnóstico + raws. Não são mudanças de implementação ainda — são os critérios que qualquer nova versão precisa satisfazer.

**1. Fail-fast, não silencioso**
Se um agente produz prosa em vez de JSON, o script deve abortar imediatamente com log do output (primeiros 300 chars) e mensagem Telegram. Nunca aceitar silenciosamente e seguir com dados vazios.

**2. Testar o contrato de output antes de gastar tokens**
O primeiro run de qualquer versão nova do auditor deve ser com um único agente, na pasta mais pequena, e verificar que o JSON retornado tem todos os campos obrigatórios. Zero tokens adicionais gastos antes disso passar.

**3. Serial antes de paralelo**
Provar que um agente funciona → provar que dois funcionam → paralelizar. Nunca lançar 8 simultâneos com código não validado.

**4. Prompt-as-contract completo**
Cada system prompt deve incluir: (a) instrução de formato, (b) exemplo concreto do JSON esperado, (c) instrução explícita para o caso vazio (`findings: []`), (d) proibição de prosa. O flag `--output-format json` é complemento, não substituto.

**5. Struct_str filtrado por agente**
Cada agente recebe apenas os dados da sua pasta. O agente `todo/` não precisa saber dos 20 arquivos de outras pastas. Isso reduz o contexto de entrada e o custo por request.

**6. Gate obrigatório — nenhum run completo sem todos os gates passados**
Os gates abaixo substituem as validações opcionais V1–V17. A diferença: são sequenciais e bloqueantes. O Gate N não pode ser pulado se o Gate N-1 não passou.

---

## Gates de validação

### Gate 1 — Contrato de output (zero tokens)

Verificação estática antes de qualquer chamada à API.

- [ ] System prompt de cada agente inclui exemplo concreto de JSON com todos os campos
- [ ] System prompt instrui o caso vazio: `{"findings": []}`
- [ ] System prompt proíbe prosa explicitamente
- [ ] Script tem fail-fast: se `json.loads()` falhar, aborta e loga os primeiros 300 chars do output
- [ ] `--output-format json` presente em todos os subprocessos

**Critério de aprovação:** revisão manual dos prompts + leitura do código do fallback.

---

### Gate 2 — Agente de pasta isolado

Um único agente, uma pasta, dados reais mínimos. Equivale à validação V2 original.

```bash
# Rodar só o agente de uma pasta pequena (ex: todo/)
# Inspecionar output diretamente
```

- [ ] Output é JSON válido (`json.loads()` não lança exceção)
- [ ] Campos obrigatórios presentes: `folder`, `findings`, `agent`
- [ ] Cada finding tem: `id`, `severity`, `file`, `problem`, `correctable`, `correction` (se correctable)
- [ ] Nenhuma prosa fora do JSON

**Critério de aprovação:** um agente, uma pasta, output válido. Parar aqui se falhar — não continuar.

**Custo esperado:** 1–3 requests, contexto < 30KB.

---

### Gate 3 — Agentes de pasta em série (não paralelo)

Rodar cada agente de pasta individualmente, em sequência, antes de paralelizar. Coleta relatórios reais.

- [ ] Todos os agentes retornam JSON válido
- [ ] Nenhum agente retorna findings vazios de forma suspeita (pasta com arquivos que claramente têm problemas)
- [ ] Log mostra tempo e tokens por agente

**Critério de aprovação:** todos os agentes passam no Gate 2 individualmente.

**Custo esperado:** N agentes × (1–3 requests cada), contexto filtrado por pasta.

---

### Gate 4 — Agentes Overlap e Links isolados

Equivale a V3 e V4 originais.

- [ ] Agente Overlap recebe inventário real, retorna JSON válido
- [ ] Agente Overlap não flaga falsos positivos óbvios
- [ ] Agente Links detecta pelo menos um link quebrado se houver
- [ ] Ambos retornam JSON válido com campos corretos

**Critério de aprovação:** os dois agentes retornam JSON válido.

---

### Gate 5 — Coordenador isolado

Alimentar o coordenador com outputs reais dos Gates 2 e 3. Equivale a V6 original.

- [ ] Output é JSON válido com `executive_summary` e `findings`
- [ ] Campo `correctable` está classificado coerentemente
- [ ] Nenhum finding duplicado
- [ ] Funciona com input misto (pasta com findings + pasta com `findings: []`)

**Critério de aprovação:** JSON válido, sem duplicatas.

---

### Gate 6 — Agente Corretor isolado

Passar um finding real do Gate 5 e verificar a edição proposta. Equivale a V7 original — o maior risco técnico do script.

- [ ] `old_string` é substring exata do arquivo alvo (verificar com `grep -F`)
- [ ] `new_string` está correto
- [ ] Campo `file` usa caminho relativo ao `WIKI_DIR` (ex: `systems/hermes.md`, não absoluto)
- [ ] Testar com dois findings no mesmo arquivo (risco D2 — ordem de aplicação)

**Critério de aprovação:** `apply_edit` aplica a edição sem erro de "old_string não encontrado".

---

### Gate 7 — Telegram (já validado)

V9a, V9b e V9c passaram em 2026-06-28 (3/3, 3/3, 2/2). Revalidar apenas se houver mudança no script de interação Telegram.

- [x] V9a — Resumo executivo (3 botões) ✅
- [x] V9b — Finding corrigível (3 botões + Ajustar) ✅
- [x] V9c — Finding não-corrigível (2 botões + Instruir) ✅

---

### Gate 8 — Run completo em dry-run

Executar Fase 1 + Fase 2 + Fase 3 completas, mas **sem Fase 5** (sem aplicar nenhuma edição).

- [ ] Todos os agentes retornam JSON válido (Gate 2–3 em paralelo agora)
- [ ] Coordenador consolida sem erro (Gate 5 em condições reais de paralelismo)
- [ ] Log mostra tempo total e estimativa de tokens consumidos
- [ ] Mensagem Telegram de resumo executivo chega com conteúdo real

**Critério de aprovação:** resumo executivo no Telegram com findings reais, nenhum erro de JSON.

**Custo esperado:** equivalente ao run completo de Fase 1–3, mas sem os tokens da Fase 5 (corretor).

---

### Gate 9 — Run completo real

Apenas após todos os gates anteriores passarem. Este é o run que aplica edições na wiki.

- [ ] Gates 0–7 todos aprovados e documentados aqui
- [ ] Giovani deu autorização explícita para este run
- [ ] `WIKI_DIR` apontando para `/root/wiki`
- [ ] Log de execução em `/var/log/auditor-wiki.log`

---

## Mapeamento: validações originais → gates

| Validação original | Gate correspondente | Status |
|---|---|---|
| V1 — Fase 1: descoberta dinâmica | Gate 8 (dry-run) | pendente |
| V2 — Agente de pasta isolado | **Gate 2** | pendente |
| V3 — Agente Overlap isolado | **Gate 4** | pendente |
| V4 — Agente Links isolado | **Gate 4** | pendente |
| V5 — Extração JSON (fallback) | **Gate 1** (estático) + Gate 2 | pendente |
| V6 — Coordenador isolado | **Gate 5** | pendente |
| V7 — Agente Corretor isolado | **Gate 6** | pendente |
| V8 — Telegram: token e chat_id | Gate 7 (pré-requisito) | ✅ |
| V9 — Telegram: interação completa | **Gate 7** | ✅ |
| V10 — apply_edit: old_string exato | **Gate 6** | pendente |
| V11 — Commits por finding | pós-Gate 9 (1º run real) | pendente |
| V12 — Push final + hook conflict | pós-Gate 9 (1º run real) | pendente |
| V13 — Execução paralela: recursos | **Gate 8** (dry-run paralelo) | pendente |
| V14 — Pasta diary/ vazia | Gate 3 (cobertura por série) | pendente |
| V15 — Timeout Telegram | Gate 7 ou Gate 8 | pendente |
| V16 — claude CLI: autenticação standalone | **Gate 1** (verificar antes de tudo) | pendente |
| V17 — Dois findings no mesmo arquivo | **Gate 6** | pendente |

---

## Consideração de arquitetura alternativa

O diagnóstico e os raws apontam para uma alternativa ao modelo de subprocessos paralelos: **uma única sessão Claude Code que analisa a wiki sequencialmente**, usando os arquivos de prompt como skills lidos on-demand.

Vantagens:
- Contexto único sem duplicação
- Sem crescimento geométrico entre turnos independentes
- Fail-fast nativo (uma falha interrompe a sessão, não fica oculta)
- Modelo de skills do agents-cli: instrução focada injetada no agente em execução

Desvantagens:
- Perde o paralelismo (tempo total maior)
- O contexto cresce ao longo da sessão (mas de forma linear, não geométrica por 8 subprocessos)

Essa alternativa não é uma decisão tomada — é um design candidato a avaliar se o Gate 2 revelar que o problema de JSON persiste mesmo com prompts corrigidos.

---

## Conexões

- [[wiki/procedures/auditor-wiki.md|Auditor da Wiki]] — o script que este documento valida; contém a análise de tokens da primeira execução
- [[raw/agents-cli-README.md|Google Agents CLI — README oficial]] — fonte dos princípios de skills e eval gates
- [[raw/akshay-pachaar-karpathy-agentic-engineering-tooling.md|Karpathy's Agentic Engineering Finally Has Proper Tooling]] — fonte do framework spec design → eval loops → deploy
- [[wiki/todo/proximos-passos.md|Próximos passos]] — onde os gates pendentes podem virar itens acionáveis
