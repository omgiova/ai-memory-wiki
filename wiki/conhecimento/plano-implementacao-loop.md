---
type: concept
tags: [agentes, loops, arquitetura, harness]
title: Arquiteturas de Loop em Agentes
description: Comparação de como diferentes frameworks de agente (Hermes, OpenClaw, Claude Code, Codex, Cline) implementam detecção de loop, aprovação, circuit breaker e verificação cruzada.
timestamp: 2026-06-27T01:55:00-03:00
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
- ❌ Loop detector com hash de args + resultado ← implementação mapeada, ver seção abaixo
- ❌ Fast-path de aprovação curta ("ok do it" → executa direto)
- ❌ Strict-agentic guard (travar se só planejar) ← contornável via `tool_choice: any` + tool sentinela
- ❌ Retry com escalation (falhou → tenta X → avisa)
- ❌ Verificação cruzada (agente A faz, agente B verifica)
- ❌ Post-compaction loop guard
- ❌ Async approval followup
- ❌ Prompt caching (`cache_control`) ← não relacionado a loop, mas reduz 85–90% dos input tokens
- ❌ Circuit breaker com sliding window ← substituto mais granular para `max_turns`

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

## Implementação no Hermes — caminhos mapeados

> Pesquisa realizada em 2026-06-27. Ordenado por ROI vs esforço.

### 1. Prompt Caching (`cache_control`) — 1–2h, 85–90% de economia em input tokens

System prompt e definição de tools não mudam entre turns — candidatos ideais. Requer que o system seja array de blocos, não string simples. TTL padrão 5min; TTL de 1h disponível com custo de write 2× e read 0,1×.

```python
system=[{
    "type": "text",
    "text": SYSTEM_PROMPT,
    "cache_control": {"type": "ephemeral"}
}]
```

**Gotcha crítico:** qualquer campo dinâmico (timestamp, UUID) injetado no prompt invalida o cache. Verificar se o Hermes injeta isso no system. Mínimo 1.024 tokens para Sonnet cachear.

### 2. `tool_choice: any` + tool sentinela — 30min

Replica 80% do `execution-contract.ts` do OpenClaw. Força o modelo a chamar pelo menos uma tool a cada turno, eliminando turns de "só texto, sem ação".

Complementar: adicionar tool `task_complete(result: str)` como saída explícita — o modelo tem saída clara em vez de texto livre.

### 3. Exact-match request dedup — 30min

Hash do último user message + model antes de encaminhar ao Claude. TTL de 30s é suficiente para duplos de reconexão do Telegram/WA sem colidir com conversa normal.

### 4. Token counting pre-flight — 1h

`client.messages.count_tokens(...)` antes de cada request real. Permite truncagem inteligente do histórico antes de estourar o contexto (e pagar o erro).

### 5. Hash loop detector — 4–8h

Replicação do OpenClaw para o Hermes. Pontos-chave:
- `sha256(tool_name + args_sem_volatile + resultado)[:16]` como fingerprint
- Strip de campos voláteis (`timestamp`, `messageId`, `runId`, `id`) antes de hashear
- Sliding window de 20 calls; warn em 5 repetições, block em 10
- Ao bloquear: injetar `tool_result` de erro artificial no loop — modelo recebe feedback sem consumir turno de LLM
- 3 padrões em paralelo: `generic_repeat`, `no_progress` (resultado idêntico), `ping_pong` (A→B→A→B)

### 6. Circuit breaker com sliding window — 2h

Substituto mais granular para `max_turns`. Janela de 60s, abre após N erros. Distinção por tipo:
- 429 rate-limit → exponential backoff com jitter, **não** conta para o breaker
- 5xx / timeout → conta
- 400 validation error → não conta

### 7. Semantic cache CPU — 1–2 dias (baixa prioridade)

**Atenção:** paper de 2026 mostra que caching de resposta em agentes multi-turn falha — o contexto muda entre turns mesmo com query idêntica. Só é seguro cachear **tool results determinísticos**. Útil apenas para queries informacionais isoladas (usuário pergunta algo ao Hermes fora de um loop de agente).

Stack viável sem GPU: `all-MiniLM-L6-v2` (22M params) + FAISS + Redis.

---

## Loop externo com Claude Code CLI na VPS

> Testado e validado em 2026-06-27 na VPS (Claude Code v2.1.183, autenticação OAuth).

Além do Hermes, o Claude Code instalado na VPS pode ser acionado como processo headless — via cron do sistema, shell scripts ou systemd — sem nenhuma sessão interativa aberta.

### Como funciona

O binário `claude` é um processo comum. Quando chamado, lê o prompt, faz requisição HTTP à API da Anthropic, executa tool calls na VPS e termina. O cron (daemon do sistema, sempre ativo desde o boot) aciona esse processo no horário configurado.

### Autenticação no cron — o que foi testado

A autenticação OAuth é salva em `~/.claude/.credentials.json`. O cron rodando como root acessa `/root/.claude/.credentials.json` automaticamente — **não é necessário configurar variáveis de ambiente**.

| Comando | Ambiente mínimo (cron) | Resultado |
|---|---|---|
| `claude --bare -p "..."` | env -i HOME=/root PATH=... | ❌ "Not logged in" |
| `claude -p "..."` | env -i HOME=/root PATH=... | ✅ funciona |

**`--bare` é incompatível com OAuth** — quebra a leitura do credentials file. A documentação recomenda `--bare` para scripts, mas assume autenticação via `ANTHROPIC_API_KEY` como env var. Com OAuth, usar `claude -p` sem `--bare`.

### Exemplo de crontab

```bash
# crontab -e (root)
0 * * * * claude -p "verifica se todos os containers do swarm estão healthy. Se algum down, registra em /tmp/swarm-health.log com timestamp BRT" --allowedTools "Bash,Write" --output-format text >> /var/log/claude-cron.log 2>&1
```

### Quando usar cada abordagem de loop externo

| Objetivo | Melhor opção |
|---|---|
| Tarefa recorrente, horário fixo, sem contexto anterior | cron + `claude -p` |
| Tarefa recorrente que preserva contexto entre execuções | cron + `claude -p --resume <session_id>` |
| Loop condicional ("roda até X acontecer") | shell script com `while` + `claude -p` |
| Loop orientado a evento (arquivo novo, webhook) | `inotifywait` / systemd `.path` + `claude -p` |
| Tarefa dentro da sessão atual sem bloquear | subagente com `background: true` em `.claude/agents/` |
| Pipeline complexo com hooks programáticos | Agent SDK Python |

### Outros flags relevantes (documentação oficial)

- `--resume <session_id>` — continua sessão específica com contexto preservado entre invocações
- `--output-format json` — output estruturado (capturável por `jq`)
- `--json-schema '{...}'` — força output a seguir um JSON Schema
- `--allowedTools "Bash,Read,Write"` — limita tools disponíveis (reduz blast radius em automações)
- `--append-system-prompt "..."` — adiciona instrução ao system prompt sem substituir

### Boas práticas validadas

- **Circuit breaker explícito** — sempre definir `MAX` de iterações no shell script; `maxTurns: N` no frontmatter de subagentes
- **Estado em arquivo** — entre invocações o Claude começa sem contexto; o que precisa persistir vai em arquivo referenciado no prompt
- **Prompt autocontido** — em modo `-p`, sem histórico de conversa; o prompt precisa incluir todo o contexto necessário
- **`--allowedTools`** — em loops automatizados, restringir ao mínimo necessário
- **Logging** — redirecionar stdout/stderr para arquivo (`>> /var/log/claude-cron.log 2>&1`)
- **Idempotência** — verificar se a operação pode rodar N vezes sem efeito colateral acumulado

---

## Conexões

- [[infraestrutura/hermes.md|Hermes]] — identidade, regras e stack do Hermes Agent
- [[automacao/firecrawl.md|Firecrawl]] — busca web com Firecrawl
- [[conhecimento/wiki.md|Wiki]] — sobre esta wiki
