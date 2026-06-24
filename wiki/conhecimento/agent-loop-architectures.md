---
type: concept
tags: [agentes, loops, arquitetura, harness]
title: Arquiteturas de Loop em Agentes
description: Comparação de como diferentes frameworks de agente (Hermes, OpenClaw, Claude Code, Codex, Cline) implementam detecção de loop, aprovação, circuit breaker e verificação cruzada.
timestamp: 2026-06-23T21:00:00-03:00
status: draft
---

# Arquiteturas de Loop em Agentes

> Insights da sessão com Giovani — comparação entre Hermes, OpenClaw (Peter Steinberger), Claude Code, Codex CLI e Cline.

## O problema

Agentes de IA frequentemente entram em **reply loop**: planejam, falam "vou fazer", mas não executam. Ou repetem a mesma tool call com os mesmos args sem progresso. Cada framework trata isso de forma diferente.

## OpenClaw — o benchmark atual

Peter Steinberger codificou soluções reais no OpenClaw (issue #57263). O sistema mais completo e **aberto** de loop detection.

### 5 Detectores de Loop

| Detector | O que detecta | Warning | Critical |
|---|---|---|---|
| `generic_repeat` | Mesma tool + args + resultado idêntico | 10 calls | 20 calls |
| `unknown_tool_repeat` | Tentativa de tool que não existe | — | 10 calls |
| `known_poll_no_progress` | `process(poll)` / `process(log)` sem progresso | 10 calls | 20 calls |
| `global_circuit_breaker` | Qualquer tool 30+ repetições sem progresso | — | **30 calls** (hard block) |
| `ping_pong` | Alternância A→B→A→B sem mudar resultado | 10 calls | 20 calls |

### Tool Outcome Hashing

Diferencial crítico: não compara args pelo nome apenas. Usa **sha256 determinístico** do args + resultado. Só considera "loop" se o resultado também não mudou — isso separa tentativa genuína de repetição.

Casos especiais tratados:
- **Volatile IDs** (messageId, runId, timestamp) são stripped do hash antes de comparar
- **`exec` results** — hash do exit code + output agregado
- **Veto do próprio loop** — não reseta a streak (impede que o bloqueio se auto-reinicie)

### Mecanismos extras do OpenClaw

1. **`execution-contract.ts`** — "strict agentic execution contract". Se ativado, o modelo é *obrigado* a chamar tool depois de planejar. Se só planeja, o guard bloqueia (`STRICT_AGENTIC_BLOCKED`).

2. **`bash-tools.exec-approval-followup.ts`** — aprovação assíncrona com retomada. Usuário aprova comando → quando termina, agente é retomado com resultado. Tem idempotency key + session rebind detection (se deu `/new` no meio, o followup é dropado).

3. **`post-compaction-loop-guard.ts`** — guard pós-compressão. Depois que o contexto é compactado, detector extra evita que a compressão em si cause loop.

4. **Ralph loop** (`subagent-spawn.ts`) — cada subagente começa com sessão **limpa**, sem arrastar histórico.

### Arquivos de referência

- `src/agents/tool-loop-detection.ts` — implementação completa dos 5 detectores
- `src/agents/embedded-agent-runner/run.ts` — run loop principal com guard integration
- `src/agents/bash-tools.exec-approval-followup.ts` — aprovação assíncrona
- `src/agents/bash-tools.exec-approval-followup-state.ts` — estado persistente

## Hermes — estado atual

### O que já tem
- `delegate_task` — spawn de subagentes (paralelo, isolado)
- `cronjob` — loops agendados
- `execute_code` — pipelines programáticos multi-tool
- `max_turns` (default 90) — hard stop global

### O que não tem
- ❌ Loop detector com hash de args + resultado
- ❌ Fast-path de aprovação curta ("ok do it" → executa direto)
- ❌ Strict-agentic guard (travar se só planejar)
- ❌ Retry com escalation (falhou → tenta X → avisa)
- ❌ Verificação cruzada (agente A faz, agente B verifica)
- ❌ Post-compaction loop guard
- ❌ Async approval followup

## Claude Code (Anthropic)

- Usa **thinking blocks** — modelo planeja antes de chamar tools. Se mostra "vou fazer X" e não chama, próximo turno corrige.
- **`tool_choice: {type: "any"}`** na API — força modelo a chamar tool no próximo turno.
- **Sem loop detector próprio** — confia em rate limiting + max_turns.
- **Sem fast-path de aprovação.**
- Código fechado — o que se sabe é por observação.

## Codex CLI (OpenAI)

- **Responses API** — cada turno é resposta completa. Se modelo só planeja, retorna texto e acabou.
- **`auto-execute` mode** — executa tools sem aprovação.
- **`tool_choice: "required"`** — equivalente ao "any" do Claude.
- **Loop detection via max_turns** (default 25) + rate limiting.
- **Fast-path parcial** — `allow-auto-approve` pra comandos seguros.

## Cline / Roo Code (VSCode)

- **`alwaysAllow`** — lista de tools que rodam sem aprovação.
- **`maxRequestsPerTask`** — default 25.
- **`autoApprovalMode`** — "fast" (leitura) ou "full".
- **Detector simples** — se mesma tool é chamada 3+ vezes com mesmo conteúdo, avisa.
- **Não tem hashing de resultado** — só compara string de args.

## Comparativo

| Funcionalidade | Hermes | OpenClaw | Claude Code | Codex | Cline |
|---|---|---|---|---|---|
| Loop detector (arg hash) | ❌ | ✅ 5 detectores | ❌ | ❌ | Parcial |
| Result hash (mudou?) | ❌ | ✅ sha256 | ❌ | ❌ | ❌ |
| Fast-path aprovação | ❌ | ✅ | ❌ | Parcial | ✅ alwaysAllow |
| Strict-agentic guard | ❌ | ✅ execution-contract | ❌ | ❌ | ❌ |
| Ping-pong detection | ❌ | ✅ | ❌ | ❌ | ❌ |
| Circuit breaker | ✅ max_turns | ✅ 30 calls | ✅ max_turns | ✅ max_turns | ✅ maxRequests |
| Post-compaction guard | ❌ | ✅ | ❌ | ❌ | ❌ |
| Subagente fresh context | ✅ delegate_task | ✅ Ralph loop | ✅ fork | ❌ | ❌ |
| Async approval followup | ❌ | ✅ idempotent | ❌ | ❌ | ❌ |

## Conexões

- [[infraestrutura/hermes.md|Hermes]] — identidade, regras e stack do Hermes Agent
- [[automacao/firecrawl.md|Firecrawl]] — busca web com Firecrawl
- [[conhecimento/wiki.md|Wiki]] — sobre esta wiki
