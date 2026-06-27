---
type: concept
tags: [agentes, loops, arquitetura, loop-engineering, planejamento]
title: Plano — Loop Engineering
description: Registro técnico de pesquisa sobre loops agênticos. O que foi lido, o que foi testado, o que está validado e o que ainda é hipótese.
timestamp: 2026-06-27T02:10:00-03:00
status: draft
---

# Plano — Loop Engineering

Registro técnico de pesquisa sobre sistemas de loop agêntico. Agnóstico de modelo e ambiente — qualquer agente deve conseguir executar. Distingue o que foi **validado** do que ainda é **hipótese**.

---

## ✅ Validado

### Loop externo com Claude Code CLI na VPS

> Testado em 2026-06-27 (Claude Code v2.1.183, autenticação OAuth).

O binário `claude` pode ser acionado pelo cron do sistema sem sessão interativa aberta. Credenciais OAuth ficam em `~/.claude/.credentials.json` e são lidas automaticamente.

| Comando | Ambiente mínimo (env -i) | Resultado |
|---|---|---|
| `claude --bare -p "..."` | HOME=/root PATH=... | ❌ "Not logged in" |
| `claude -p "..."` | HOME=/root PATH=... | ✅ funciona |

`--bare` é incompatível com OAuth — assume `ANTHROPIC_API_KEY`. Com OAuth, usar `claude -p` sem `--bare`.

**Modalidades de loop externo:**

| Objetivo | Abordagem |
|---|---|
| Tarefa recorrente, horário fixo | cron + `claude -p` |
| Loop com contexto preservado | cron + `claude -p --resume <session_id>` |
| Loop condicional ("até X acontecer") | shell script `while` + `claude -p` |
| Loop orientado a evento | `inotifywait` / systemd `.path` + `claude -p` |
| Tarefa sem bloquear sessão atual | subagente `background: true` em `.claude/agents/` |
| Pipeline com hooks programáticos | Agent SDK Python |

**Boas práticas observadas (não testadas exaustivamente):**
- Circuit breaker explícito: `MAX` iterações no script ou `maxTurns: N` no frontmatter do subagente
- Estado em arquivo — prompt autocontido, sem histórico de conversa
- `--allowedTools` — restringir ao mínimo necessário
- Logging: `>> /var/log/claude-cron.log 2>&1`
- Idempotência: operação deve poder rodar N vezes sem efeito cumulativo

---

## 🚧 A validar

### Curador da Wiki — Teste 1

> Implementado em 2026-06-27. Aguardando avaliação do output pelo Giovani.

Primeiro loop concreto construído a partir deste plano. Valida a hipótese central: um único `claude -p` com contexto distilado consegue curar uma daily note corretamente?

**Script:** `/root/curator-teste1.sh`

**Arquitetura:**
```
bash curator-teste1.sh
  ├── sorteia daily aleatória de wiki/diario/
  ├── embute index.md + conteúdo da daily no prompt (sem tool calls)
  ├── claude --allowedTools "" -p "..."
  └── envia 2 mensagens ao Telegram Geral (thread_id=1)
        msg 1: conteúdo completo da daily
        msg 2: bloco de curadoria estruturado
```

**Decisões de design adotadas:**

| Decisão | Motivo |
|---|---|
| Contexto embutido no prompt (não via Read tools) | Mais simples e previsível para MVP |
| `index.md` dinâmico no prompt | Reflete estado real da wiki sem reconfiguração |
| Daily aleatória | Evita viés de validação com amostras fáceis |
| `--allowedTools ""` | Agente read-only por construção, não por confiança |
| Um agente único | Valida o critério antes de separar responsabilidades |

**O que este teste valida:**
- Prompt com critério Karpathy + OKF distilado é suficiente para raciocinar sobre valor
- `index.md` como único mapa de navegação da wiki funciona para o agente
- Pipeline shell → `claude -p` → Telegram funciona de ponta a ponta

**O que ainda não está validado (próximos passos após aprovação):**
- Qualidade real da curadoria (o Giovani avalia o output)
- Comportamento com dailies longas (561 linhas)
- Se um agente único escala ou se precisará ser dividido em 4

Ver documentação completa: [[automacao/curador-wiki.md|Curador da Wiki]]

---

## 🔬 Hipóteses (não testadas)

Padrões descritos em fontes públicas, ainda não verificados na prática.

**Memória em disco, não em contexto**
O agente começa cada execução sem contexto. Estado persistente fica em arquivo (ex: `.tsv`, `.json`, `.md`), não no histórico de conversa.

**Supressão de output do contexto**
Redirecionar output de processos longos para arquivo e extrair só o sinal necessário evita poluir o contexto e degradar a qualidade das próximas iterações.

**Condição de parada externa**
O agente não decide quando parar — a condição de parada é definida pelo humano no design do loop (número máximo de iterações, threshold de métrica, interrupção manual).

**Maker/checker**
O agente que produz algo não deve revisar o próprio trabalho. Um segundo agente com instruções diferentes revisa.

**Git como máquina de estados**
Usar commit/reset como lógica de decisão do loop (keep = commit, discard = reset) em vez de lógica aplicacional separada.

---

## 🧪 Sistemas pesquisados — a testar

### Teste 1: karpathy/autoresearch

**Repositório:** https://github.com/karpathy/autoresearch  
**Autor:** Andrej Karpathy  
**Stars:** 88.8k | **Stack:** Python + uv | **Último commit:** mar/2026

**O que é:**
Loop de pesquisa autônoma em ML. O agente modifica `train.py`, treina por 5 minutos, verifica se o resultado melhorou, mantém ou descarta, e repete indefinidamente. O humano edita `program.md` (instruções do agente) e não toca no código.

**Arquitetura:**

```
program.md    → instruções do agente (editado pelo humano)
train.py      → único arquivo que o agente modifica
prepare.py    → imutável (avaliação, dados, tokenizer)
results.tsv   → memória em disco, não commitada
```

**Lógica do loop:**
```bash
# cada iteração:
uv run train.py > run.log 2>&1
grep "^val_bpb:" run.log

# decisão:
val_bpb melhorou? → git commit   (keep)
val_bpb piorou?   → git reset    (discard)
```

**Critério de aceitação de mudança:**
Não é só "o score melhorou?" — é "o ganho justifica a complexidade adicionada?":
- Ganho pequeno + código novo → discard
- Remoção de código com ganho igual → keep
- Ganho zero + código mais simples → keep

**Pré-requisito bloqueante:** GPU NVIDIA — ⚠️ VPS atual não tem.

**Opções para executar:**
- Máquina local com GPU NVIDIA
- Instância cloud com GPU (Vast.ai, RunPod)
- Fork para outro hardware (MacOS MLX: https://github.com/trevin-creator/autoresearch-mlx)
- Adaptar o loop para tarefa sem GPU (substituir `train.py` por outra tarefa mensurável)

---

### Teste 2: OpenClaw

**Repositório:** https://github.com/openclaw/openclaw  
**Autor:** Peter Steinberger + comunidade  
**Stars:** 381k | **Stack:** TypeScript/Node.js | **Último commit:** 27 jun 2026

**O que é:**
Gateway agêntico com control plane central em WebSocket (`127.0.0.1:18789`). Todos os canais (WhatsApp, Telegram, Discord, CLI, Web UI) conectam no mesmo hub.

**Arquitetura (hub-and-spoke):**

Agent Runtime (`src/agents/piembeddedrunner.ts`) — a cada turno:
1. Resolve sessão
2. Monta contexto
3. Streama resposta e executa tool calls
4. Persiste estado

**Decisões de design documentadas:**
- System prompt como stack de arquivos — composição de múltiplas configs, não string única
- Idempotency key obrigatório em toda operação com efeito colateral
- Cron jobs escopados por agente, não compartilhados (commit #96883)
- Session isolation — sessão `main` roda tools no host; `dm`/`group` têm sandbox restrito
- Plugin system — 4 tipos (channel, memory, tool, provider), discovery via `package.json`

**Sistema de detecção de loop infinito (documentado em issue #57263):**

| Detector | O que detecta | Warning | Bloqueio |
|---|---|---|---|
| `generic_repeat` | mesma tool + args + resultado idêntico | 10 calls | 20 calls |
| `unknown_tool_repeat` | tool inexistente chamada repetidamente | — | 10 calls |
| `known_poll_no_progress` | poll/log sem progresso real | 10 calls | 20 calls |
| `global_circuit_breaker` | qualquer tool 30+ vezes sem progresso | — | 30 calls |
| `ping_pong` | alternância A→B→A→B sem resultado novo | 10 calls | 20 calls |

**Tool Outcome Hashing:** `sha256(args_sem_volatile + resultado)` — campos voláteis (`timestamp`, `messageId`, `runId`) removidos antes do hash. Só é loop se o resultado também não mudou.

**Limitação de acesso:** código-fonte de `src/agents/` não acessível publicamente sem autenticação GitHub. Informações acima vêm de documentação pública e issues.

---

## ❓ Gaps — o que ainda não sabemos

- [ ] Como o OpenClaw implementa maker/checker na prática
- [ ] Como funciona o checker de condição de parada do `/goal` do Claude Code
- [ ] Como implementar idempotency key sem redesenhar APIs existentes

---

## 📚 Leituras pendentes

- **Agent Harness Engineering:** https://addyosmani.com/blog/agent-harness-engineering/
- **Long-Running Agents:** https://addyosmani.com/blog/long-running-agents/
- **The Orchestration Tax:** https://addyosmani.com/blog/orchestration-tax/
- **Adversarial Code Review (maker/checker):** https://addyosmani.com/blog/adversarial-code-review/
- **The Intent Debt:** https://addyosmani.com/blog/intent-debt/

---

## Conexões

- [[infraestrutura/hermes.md|Hermes]] — identidade, regras e stack do Hermes Agent
- [[pendencias/proximos-passos.md|Próximos Passos]] — to-do list ativa
- [[automacao/curador-wiki.md|Curador da Wiki — Teste 1]] — primeiro loop implementado a partir deste plano; MVP de curadoria de daily notes
