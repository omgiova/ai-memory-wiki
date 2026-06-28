---
type: procedure
tags: [auditoria, lint, wiki, multi-agente]
title: Auditor da Wiki
description: Automação multi-agente que executa health-check completo da wiki em 3 fases — estrutural, semântica e coordenação — e entrega página de auditoria priorizada.
timestamp: 2026-06-28T00:00:00-03:00
status: draft
---

# Auditor da Wiki

Automação bash que roda uma auditoria completa da wiki em 3 fases paralelas, produz uma página de findings priorizada em `wiki/todo/` e notifica o Telegram.

Segue o checklist de Lint definido em [[AGENTS.md]] — verifica taxonomia, seções obrigatórias, type OKF, frontmatter, orphans, sync index vs git, sobreposição semântica e consistência de status.

## O que faz

- **Fase 0:** análise estrutural em bash puro — sem LLM. Coleta arquivos por pasta, extrai type/status do frontmatter de cada arquivo, calcula diff entre `git ls-files` e `index.md`.
- **Fase 1:** 3 agentes Claude paralelos, cada um com escopo distinto, lendo os arquivos atribuídos com `--allowedTools "Read"` e produzindo relatório JSON estruturado.
- **Fase 2:** agente coordenador recebe os 3 relatórios + dados estruturais, deduplica findings, prioriza por severidade e produz o conteúdo da página de auditoria.
- **Fase 3 (bash):** escreve a página em `wiki/todo/`, atualiza `index.md`, appenda `log.md`, commita e notifica o Telegram.

## Gatilho

Manual — on demand. Executar quando solicitado pelo usuário para health-check da wiki.
Candidato futuro a cron mensal após validação completa.

## Como executar

```bash
bash /root/auditor-wiki-v1.sh
```

Log de execução: `/var/log/auditor-wiki.log`

## Entradas e saídas

**Entradas:**
- Todos os arquivos `.md` em `wiki/` (lidos pelos agentes via Read)
- `/root/wiki/AGENTS.md` (taxonomia e regras — lido por todos os agentes)
- `/root/wiki/index.md` (lido pelo agente C)

**Saídas:**
- `wiki/todo/auditoria-YYYY-MM-DD.md` — página com findings priorizados
- `/var/log/auditor-wiki.log` — log de execução com timestamps BRT
- `/tmp/auditor-wiki-*/` — arquivos temporários removidos ao final
- Notificação Telegram com resumo executivo

## Arquivos

| Arquivo | Papel |
|---|---|
| `/root/auditor-wiki-v1.sh` | Script principal |
| `/root/auditor-wiki-agent-prompt-v1.md` | System prompt dos agentes A, B, C |
| `/root/auditor-wiki-coord-prompt-v1.md` | System prompt do coordenador |

## Escopo dos agentes

| Agente | Arquivos auditados | Foco adicional |
|---|---|---|
| A | `systems/` + `tools/` | Sobreposição semântica entre ferramentas; seções obrigatórias por tipo |
| B | `procedures/` + `concepts/` | Procedures que deveriam ser concepts e vice-versa; conteúdo mencionado sem página própria |
| C | `index.md` + `history/` + `todo/` + diff estrutural | Sync git vs index; items maduros em todo/ prontos para procedures/; orphans |

## Resultado esperado

Página `wiki/todo/auditoria-YYYY-MM-DD.md` com:

```
## Crítico
## Alto impacto
## Médio impacto
## Baixo impacto
## Estrutural (index vs git)
## Sem ação necessária
```

Cada finding com: arquivo afetado, descrição objetiva, sugestão acionável.

Notificação Telegram:
```
🔍 Auditoria wiki — YYYY-MM-DD
X findings: N críticos, N alto, N médio, N baixo
→ wiki/todo/auditoria-YYYY-MM-DD.md
```

## Pontos de validação obrigatórios antes de usar em produção

### V1 — Fase 0 (bash/Python estrutural)
Rodar isolado e inspecionar `/tmp/auditor_struct_test.json`:
```bash
python3 /root/auditor-wiki-v1.sh --dry-run-phase0
```
Verificar: todos os arquivos listados, types corretos, diff git vs index preciso.

### V2 — Agente A isolado
Rodar só o agente A e inspecionar output antes de rodar os 3 em paralelo:
```bash
# descomentar bloco de teste no script
```
Verificar: output é JSON válido, findings têm todos os campos obrigatórios, não há prose no lugar de JSON.

### V3 — Execução paralela
3 processos `claude` simultâneos consomem recursos significativos. Verificar:
- Memória disponível na VPS
- Rate limits da API Claude
- Se o gateway Hermes interfere com chamadas externas ao `claude` CLI

### V4 — Extração JSON
O script tenta parsear o `result` de cada agente como JSON. Se o agente produzir prose, cai no fallback (findings = []). Verificar se o fallback é acionado e como o coordenador lida com relatório vazio.

### V5 — Coordenador
Verificar que o coordenador recebe todos os dados e produz markdown bem estruturado, sem truncamento (o prompt pode ser grande com 3 JSONs concatenados).

### V6 — Escrita da página e OKF
Verificar que a página gerada tem frontmatter OKF completo e válido antes do commit.

### V7 — Atualização do index.md
O script faz append automático na seção `### todo/` do index.md. Verificar que não duplica entradas se rodado mais de uma vez no mesmo dia.

### V8 — Token Telegram
Script lê `TELEGRAM_BOT_TOKEN` de `~/.hermes/.env`. Verificar disponibilidade antes da primeira execução.

### V9 — Hardcoded file lists
A fase 1 tem listas fixas de arquivos por agente. Se arquivos forem adicionados ou removidos das pastas, o script precisa ser atualizado manualmente. Candidato a descoberta dinâmica em v2.

## Conexões

- [[AGENTS.md]] — taxonomia, templates e checklist de Lint que este script implementa
- [[wiki/procedures/curador-wiki.md|Curador da Wiki]] — automação de referência com mesma arquitetura
- [[wiki/todo/proximos-passos.md|Próximos passos]] — onde os findings desta auditoria podem gerar itens
