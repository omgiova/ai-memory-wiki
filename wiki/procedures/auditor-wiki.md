---
type: procedure
tags: [auditoria, lint, wiki, multi-agente, telegram]
title: Auditor da Wiki
description: Automação multi-agente que executa health-check completo da wiki, apresenta cada correção via Telegram para aprovação e aplica somente o que for autorizado.
timestamp: 2026-06-28T00:00:00-03:00
status: draft
---

# Auditor da Wiki

Automação bash que roda uma auditoria completa da wiki em 5 fases, valida cada correção individualmente via Telegram com botões interativos e commita somente o que o usuário aprovar.

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
| `/root/auditor-wiki-agent-prompt-v1.md` | System prompt dos agentes de pasta |
| `/root/auditor-wiki-coord-prompt-v1.md` | System prompt do coordenador |
| `/root/auditor-wiki-corrector-prompt-v1.md` | System prompt do agente corretor |
| `/root/auditor-wiki-overlap-prompt-v1.md` | System prompt do agente de sobreposição |
| `/root/auditor-wiki-links-prompt-v1.md` | System prompt do agente de links |

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

## Conexões

- [[AGENTS.md]] — taxonomia, templates e checklist de Lint que este script implementa
- [[wiki/procedures/curador-wiki.md|Curador da Wiki]] — automação de referência com mesma arquitetura
- [[wiki/todo/proximos-passos.md|Próximos passos]] — onde os findings desta auditoria podem gerar itens
