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

---

## Evals de validação

Sequência única e bloqueante — Eval N só inicia se Eval N-1 passou. Nenhum eval pode ser pulado.

**Modelo para todos os subagentes:** `claude-sonnet-4-6`

---

### Eval 1 — Contrato de output (estático, zero tokens)

Verificação antes de qualquer execução — leitura do arquivo de definição do subagente e do system prompt.

- [ ] Arquivo `.claude/agents/auditor-pasta.md` existe com frontmatter correto
- [ ] `model: claude-sonnet-4-6` declarado no frontmatter do subagente
- [ ] System prompt inclui exemplo concreto do JSON esperado com todos os campos
- [ ] System prompt instrui caso vazio: `{"findings": []}`
- [ ] System prompt proíbe prosa explicitamente

**Critério de aprovação:** revisão manual do arquivo do subagente e do system prompt.

---

### Eval 2 — Subagente retorna JSON?

Primeiro teste com tokens. Um subagente, pasta mais pequena (`todo/`), invocado via ferramenta `Agent` pela sessão principal.

- [ ] Sessão principal spawna o subagente via `Agent` sem erro
- [ ] Resposta que chega de volta é JSON válido (`json.loads()` não lança exceção)
- [ ] Campos obrigatórios presentes: `folder`, `findings`, `agent`
- [ ] Cada finding tem: `id`, `severity`, `file`, `problem`, `correctable`, `correction` (se correctable)
- [ ] Nenhuma prosa fora do JSON

**Critério de aprovação:** JSON válido recebido na sessão principal. Parar aqui se falhar — não continuar.

**Custo esperado:** 1–3 requests, contexto < 30KB.

---

### Eval 3 — Erro propaga ou engole silencioso?

Provocar falha intencional: invocar subagente sem instrução de formato JSON e verificar se a sessão principal recebe e reage ao erro — o teste que v1 nunca fez.

- [ ] Sessão principal detecta que o subagente não retornou JSON válido
- [ ] Erro não é silenciosamente descartado (equivalente ao fallback `findings: []` do v1)
- [ ] Sessão principal consegue abortar, logar ou alertar

**Critério de aprovação:** falha no subagente resulta em erro visível e tratável na sessão principal.

---

### Eval 4 — Sessão aguenta esperar o Telegram?

O ponto mais crítico da arquitetura. Sessão Claude Code envia mensagem com botões e aguarda callback — não pode expirar durante a espera.

- [ ] Sessão envia mensagem com botões via Bash (curl) pro Telegram
- [ ] Sessão aguarda callback via polling ativo (loop `curl getUpdates` com sleep, não espera passiva — Claude Code não suspende sessão)
- [ ] Resposta do botão é recebida e processada corretamente dentro da sessão

**Critério de aprovação:** loop Telegram completo (envio → polling → recebimento do callback) dentro de uma sessão Claude Code sem interrupção.

---

### Eval 5 — Agentes de pasta em série

Todos os agentes de pasta, um por vez, antes de qualquer paralelismo.

- [ ] Cada agente retorna JSON válido
- [ ] Nenhum agente retorna `findings: []` de forma suspeita para pasta com problemas óbvios
- [ ] Tempo e custo por agente registrados

**Critério de aprovação:** todos os agentes passam individualmente.

---

### Eval 6 — Agentes Overlap e Links isolados

- [ ] Agente Overlap retorna JSON válido sem falsos positivos óbvios
- [ ] Agente Links detecta pelo menos um link quebrado se houver
- [ ] Ambos retornam JSON válido com campos corretos

**Critério de aprovação:** os dois agentes retornam JSON válido.

---

### Eval 7 — Coordenador isolado

Alimentado com outputs reais dos Evals 5 e 6.

- [ ] Output é JSON válido com `executive_summary` e `findings`
- [ ] Campo `correctable` classificado coerentemente
- [ ] Nenhum finding duplicado
- [ ] Funciona com input misto (pasta com findings + pasta com `findings: []`)

**Critério de aprovação:** JSON válido, sem duplicatas.

---

### Eval 8 — Agente Corretor isolado

O maior risco técnico: LLMs normalizam espaços e quebras de linha — se `old_string` não bater exato com o arquivo, a edição falha.

- [ ] `old_string` é substring exata do arquivo alvo (verificar com `grep -F`)
- [ ] `new_string` está correto
- [ ] Campo `file` usa caminho relativo ao `WIKI_DIR` (ex: `systems/hermes.md`, não absoluto)
- [ ] Testar com dois findings no mesmo arquivo (ordem de aplicação)

**Critério de aprovação:** edição aplicada sem erro de "old_string não encontrado".

---

### Eval 9 — Dry-run completo

Fluxo completo de análise e coordenação, sem aplicar nenhuma edição.

- [ ] Todos os agentes retornam JSON válido em condições reais de paralelismo
- [ ] Coordenador consolida sem erro
- [ ] Custo total de tokens dentro do esperado
- [ ] Resumo executivo chega no Telegram com findings reais

**Critério de aprovação:** resumo executivo no Telegram com findings reais, nenhum erro de JSON.

---

### Eval 10 — Run completo real

Apenas após todos os evals anteriores passarem e com autorização explícita.

- [ ] Evals 1–9 todos aprovados e documentados aqui
- [ ] Giovani autorizou este run
- [ ] `WIKI_DIR` apontando para `/root/wiki`
- [ ] Log de execução em `/var/log/auditor-wiki.log`

---


## Conexões

- [[wiki/procedures/auditor-wiki.md|Auditor da Wiki]] — o script que este documento valida; contém a análise de tokens da primeira execução
- [[raw/agents-cli-README.md|Google Agents CLI — README oficial]] — fonte dos princípios de skills e eval gates
- [[raw/akshay-pachaar-karpathy-agentic-engineering-tooling.md|Karpathy's Agentic Engineering Finally Has Proper Tooling]] — fonte do framework spec design → eval loops → deploy
- [[wiki/todo/proximos-passos.md|Próximos passos]] — onde os gates pendentes podem virar itens acionáveis
