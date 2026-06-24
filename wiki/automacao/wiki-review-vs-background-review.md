---
type: reference
tags: [hermes, wiki, automacao, background-review, wiki-review, comparacao]
title: wiki_review vs background_review — Comparação Completa
description: Tabela completa de todas as diferenças entre o clone wiki_review e o background_review nativo do Hermes (41 itens).
timestamp: 2026-06-24T00:00:00+00:00
status: stable
---

# wiki_review vs background_review — Comparação Completa

Documentação de todas as diferenças identificadas entre `wiki_review.py` (clone customizado) e `background_review.py` (nativo do Hermes). Gerada em 2026-06-24 após revisão linha a linha dos dois arquivos.

## Legenda

- ✅ — correto/igual/corrigido
- ❌ pendente — falta, pode ser problema ou melhoria futura
- ⚠️ OK — diferença intencional ou sem impacto real
- 🆕 único — existe só no wiki_review, correto assim

---

## Tabela completa (41 itens)

| # | Categoria | Item | background_review | wiki_review | Status |
|---|---|---|---|---|---|
| 1 | Callback | `_bg_review_auto_deny` (terminal auto-deny) | antes do try | adicionado (2026-06-24) | ✅ corrigido |
| 2 | Callback | `_set_approval_callback(None)` no finally | sim | adicionado (2026-06-24) | ✅ corrigido |
| 3 | Agente | `review_agent = None` antes do try (init defensivo) | sim | não | ❌ pendente |
| 4 | Agente | `enabled_toolsets` no constructor | herda do pai | `["file"]` | ⚠️ intencional |
| 5 | Agente | `disabled_toolsets` no constructor | herda do pai | ausente | ⚠️ OK |
| 6 | Agente | `_memory_write_origin` | "background_review" | ausente | ⚠️ OK (skip_memory) |
| 7 | Agente | `_memory_write_context` | "background_review" | ausente | ⚠️ OK (skip_memory) |
| 8 | Agente | `_skip_mcp_refresh = True` | sim | adicionado (2026-06-24) | ✅ corrigido |
| 9 | Agente | `_memory_store = agent._memory_store` | sim | ausente | ⚠️ OK (skip_memory) |
| 10 | Agente | `_memory_enabled = agent._memory_enabled` | sim | ausente | ⚠️ OK (skip_memory) |
| 11 | Agente | `_user_profile_enabled = agent._user_profile_enabled` | sim | ausente | ⚠️ OK (skip_memory) |
| 12 | Agente | `_memory_nudge_interval = 0` | sim | sim | ✅ igual |
| 13 | Agente | `_skill_nudge_interval = 0` | sim | sim | ✅ igual |
| 14 | Agente | `suppress_status_output = True` | sim | sim | ✅ igual |
| 15 | Cache | `_cached_system_prompt` herdado do pai (mesmo modelo) | sim | não | ❌ pendente |
| 16 | Cache | `session_start` herdado do pai (junto com #15) | sim | não | ❌ pendente |
| 17 | Sessão | `session_id = agent.session_id` | sim | sim | ✅ igual |
| 18 | Sessão | `_end_session_on_close = False` | sim | sim | ✅ igual |
| 19 | Sessão | `compression_enabled = False` | sim | sim | ✅ igual |
| 20 | Whitelist | ferramentas permitidas | memory + skills | file | ⚠️ intencional |
| 21 | Whitelist | `clear_thread_tool_whitelist` no finally | sim | sim | ✅ igual |
| 22 | Histórico | digest compacto quando routed (modelo diferente) | `_digest_history()` | sempre full | ❌ pendente |
| 23 | Prompt | aviso inline "outras tools serão negadas" no run_conversation | sim | não | ❌ menor |
| 24 | Teardown | `shutdown_memory_provider()` dentro do redirect | sim | não | ⚠️ OK (skip_memory) |
| 25 | Teardown | `review_agent.close()` | sim | sim | ✅ igual |
| 26 | Teardown | `review_agent = None` após close | sim | não | ❌ menor |
| 27 | Mensagens | snapshot `_session_messages` após run | sim | não | ⚠️ intencional |
| 28 | Notificação | `_safe_print` com mensagem de conclusão | `💾 Self-improvement review: ...` | `📓 Wiki review: 📝 Wiki daily note atualizada` | ✅ restaurado (2026-06-24) |
| 29 | Notificação | `background_review_callback` | sim | adicionado (2026-06-24) | ✅ restaurado |
| 30 | Erro | `logger.warning` no except da thread | sim | adicionado (2026-06-24) | ✅ corrigido |
| 31 | Erro | `agent._emit_auxiliary_failure` no except | sim | não | ⚠️ OK (silencioso por design) |
| 32 | Erro | cleanup de `review_agent` no finally (exception path) | sim | não | ❌ menor |
| 33 | Único wiki | git commit após escrever o diário | — | sim | 🆕 único |
| 34 | Único wiki | frontmatter YAML no arquivo de diário | — | sim | 🆕 único |
| 35 | Único wiki | contador persistido em disco (`wiki_review_counter`) | — | removido (2026-06-24) | 🆕 substituído |
| 36 | Ativação | Tipo de contador | RAM (zera no restart) | RAM (2026-06-24) | ✅ corrigido |
| 37 | Ativação | O que incrementa o contador | turno com `memory` disponível + `_memory_store` truthy | idem (2026-06-24) | ✅ corrigido |
| 38 | Ativação | Número de gatilhos | 2 independentes (memória + skills) | 1 (só memória) | ⚠️ intencional |
| 39 | Ativação | Condição extra para disparar | memory tool disponível + `_memory_store` truthy | idem (2026-06-24) | ✅ corrigido |
| 40 | Ativação | Intervalo padrão | `_memory_nudge_interval` (padrão 10 turnos) | idem (2026-06-24) | ✅ corrigido |
| 41 | Ativação | Entry points no código | `turn_finalizer.py` + `codex_runtime.py` | só `turn_finalizer.py` | ⚠️ OK |

---

## Itens ❌ pendentes por ordem de impacto

| # | Item | Impacto |
|---|---|---|
| 15+16 | cache parity (`_cached_system_prompt`, `session_start`) | tokens extras a cada disparo (custo) |
| 22 | digest logic para modelo roteado | necessário ao configurar modelo auxiliar mais barato |
| 3+26+32 | `review_agent = None` defensivo + cleanup no except/finally | leak se o constructor do AIAgent falhar no meio |
| 23 | aviso inline sobre tools no prompt | reduz tentativas do modelo de usar tools negadas |

---

## Histórico de correções (2026-06-24)

| Item | O que foi corrigido |
|---|---|
| Gatilho (#36–40) | Trocado contador em disco por `_should_review_memory` (mesmo gatilho do background_review) |
| `_skip_mcp_refresh` (#8) | Adicionado — evita que refresh de MCP corrompa o fork |
| `_bg_review_auto_deny` (#1) | Adicionado — evita deadlock em input() se terminal tool for acionada |
| `_set_approval_callback(None)` (#2) | Adicionado no finally — limpa callback ao fim da thread |
| Logger na thread (#30) | Trocado `logger.debug` por `logger.warning` — erros agora visíveis nos logs |
| `_safe_print` + callback (#28+29) | Restaurado — `📓 Wiki review: 📝 Wiki daily note atualizada` (existia na versão original, perdido na recriação do arquivo) |

## Conexões

- [[wiki/automacao/wiki-review.md|wiki-review]] — documentação de funcionamento
- [[wiki/infraestrutura/hermes.md|Hermes Config]] — onde os arquivos vivem
