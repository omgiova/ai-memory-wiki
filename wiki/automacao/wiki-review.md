---
type: procedure
tags: [hermes, wiki, automacao, background, diario, plugin]
title: Wiki Review — Auto-escrita de diário em background
description: Plugin que roda a cada N turnos e salva insights da conversa na wiki (diario/).
timestamp: 2026-06-24T09:00:00-03:00
status: stable
---

# Wiki Review

Plugin do Hermes que analisa a conversa automaticamente e escreve insights no diário da wiki (`wiki/diario/YYYY-MM-DD-{session}.md`). Roda em background sem intervenção do usuário.

## Implementação atual (v6 — sessão por session_id, contador por sessão)

**O que mudou em v6 (2026-06-24):**
- Estado unificado em `~/.hermes/wiki_review_session.json` como dicionário `{session_id: {diary_path, last_activity, counter}}`
- Contador deixou de ser global (`wiki_review_counter`) — agora é por session_id
- Arquivo de diário nomeado `YYYY-MM-DD-HHMM-{session_short}.md` — cada tópico/chat tem o seu
- Timestamps em BRT (`-03:00`) em vez de UTC
- Notificação Telegram no tópico wiki_review (thread 749) após cada commit

**Rollback para v5:** restaurar o `__init__.py` com `_get_diary_path()` sem parâmetro, SESSION_STATE_FILE como `{"diary_path":..., "last_activity":...}` e contador global em `~/.hermes/wiki_review_counter`. Ver seção v5 no histórico abaixo.

---

## Implementação anterior (v4/v5 — plugin, sobrevive a hermes update)

O wiki_review vive em `~/.hermes/plugins/wiki-review/`. Arquivo não rastreado pelo git do hermes-agent — sobrevive a qualquer `hermes update`.

```
~/.hermes/plugins/wiki-review/
  __init__.py    → lógica: contador, spawn de thread, AIAgent, git commit
  plugin.yaml    → manifest: name, version, hooks: [post_llm_call]
```

Ativação em `~/.hermes/config.yaml`:
```yaml
plugins:
  enabled:
    - wiki-review
```

### Fluxo

```
turno N → post_llm_call hook disparado
               ↓ verifica contador (/root/.hermes/wiki_review_counter)
               ↓ se count >= nudge_interval → reseta para 0, dispara
          spawn thread daemon
               ↓
          Python cria diary_path com frontmatter (se não existir)
               ↓
          AIAgent(enabled_toolsets=["file"], skip_memory=True)
          → run_conversation(prompt, histórico)
          → write_file(/tmp/wiki_review_{session}.md)   ← APENAS novas seções
               ↓ após agente terminar
          Python lê temp, faz append no diary_path, deleta temp
               ↓
          git -C /root/wiki add -A && git commit
               ↓ hook post-commit
          git pull --rebase + push
```

### Nome do arquivo de saída

```
wiki/diario/YYYY-MM-DD-HHMM.md
```

O horário é o do **primeiro disparo da sessão**. Disparos subsequentes na mesma sessão appendam no mesmo arquivo.

**Por que não usa session_id?** No gateway Telegram o `session_id` é o `chat_id` — fixo para sempre, igual em todas as conversas do usuário. Usar o session_id como nome faria TODAS as conversas ir para o mesmo arquivo. A detecção de sessão por inatividade resolve isso sem precisar de acesso ao objeto `agent`.

### Detecção de sessão por inatividade

Estado persistido em `/root/.hermes/wiki_review_session.json`:
```json
{"diary_path": "/root/wiki/wiki/diario/2026-06-24-1430.md", "last_activity": "2026-06-24T14:32:00+00:00"}
```

Lógica:
- Se `now - last_activity < 60 min` → mesma sessão, usa o mesmo arquivo, atualiza `last_activity`
- Se `now - last_activity >= 60 min` → nova sessão, cria arquivo com novo `HHMM`, reseta o estado

### Por que arquivo temporário?

O bug crítico das versões anteriores: o agente tentava ler o diário existente e reescrever o arquivo inteiro com as novas seções adicionadas. O argumento `content` do `write_file` ficava enorme (frontmatter + conteúdo antigo + novas seções). O modelo truncava o JSON na geração e o `message_sanitization` não conseguia reparar:

```
WARNING agent.message_sanitization: Unrepairable tool_call arguments for write_file
```

A solução: Python cria o arquivo e faz o merge. O agente só gera as novas seções (~10-20 linhas) e salva num temp. JSON pequeno, sem truncamento.

### Parâmetros

| Parâmetro | Onde | Padrão |
|---|---|---|
| Intervalo de disparo | `memory.nudge_interval` em config.yaml | 10 |
| Diretório da wiki | hardcoded no plugin | `/root/wiki` |
| Modelo usado | passado pelo hook `post_llm_call` | mesmo do turno |

### Contador de turnos

Persiste em `/root/.hermes/wiki_review_counter` (inteiro simples). Sobrevive a restarts do gateway. Incrementa a cada turno; zera quando dispara.

### Limitações conhecidas em relação ao background_review

O plugin recebe `session_id`, `model`, `platform` e `conversation_history` via hook, mas não tem acesso ao objeto `agent`. Por isso:

- **Credenciais**: usa as do config.yaml, não herda o pool de credenciais do pai
- **Cache de prompt**: não herda `_cached_system_prompt` — não aproveita o cache de prefixo
- **Notificação visual**: apenas `logger.info`, não aparece no terminal do usuário (futuro: explorar hook `on_session_start` ou similar)
- **Condição de disparo**: contador por N turnos (mesmo `nudge_interval`), não exatamente `_should_review_memory`

## O prompt — critério de durabilidade

O agente filho recebe o histórico e filtra por **durabilidade**: só escreve o que for útil na PRÓXIMA sessão. Não é um resumo do que aconteceu.

Sinais que justificam uma entrada:
- Usuário corrigiu comportamento/tom/abordagem (frustração explícita é sinal de primeira classe)
- Usuário revelou algo DURÁVEL sobre si — preferência estável, estilo de trabalho, objetivo de longo prazo
- Técnica, fix, config ou caminho de debugging reutilizável surgiu
- Decisão explicitamente adiada com follow-up necessário

Não captura (ruído que polui o diário):
- Narrativas de sessão ("falamos sobre X", "testamos Y") — o que aconteceu hoje não é durável
- Erros transitórios que se resolveram
- Pendências vagas sem dono ou contexto acionável
- Afirmações negativas sobre ferramentas ("X não funciona")
- Resumos do que o agente pesquisou ou produziu

**"Nada a registrar." é um outcome legítimo e esperado** — sessões tranquilas sem correções devem retornar vazio.

Cada finding vira uma seção `## {título} — {HH:MM}` com 2-4 linhas: o que é, por que importa para o futuro, como aplicar.

## Arquivo de saída

Ver seção "Nome do arquivo de saída" acima — gerado por detecção de sessão por inatividade.

## Restrições do agente filho

- Toolset: apenas `file` (read_file, write_file) + whitelist de thread enforced em runtime
- `skip_memory=True` — não acessa nem escreve na `memory()` do Hermes
- `max_iterations=16`
- `compression_enabled=False`
- Auto-deny em comandos perigosos (mesma proteção do background_review)

## Rollback

```bash
# Desativar plugin
# Editar ~/.hermes/config.yaml: remover "- wiki-review" de plugins.enabled
hermes gateway restart

# Remover completamente
rm -rf /root/.hermes/plugins/wiki-review/
```

---

## Histórico de implementações (para não repetir erros)

### v1 — source tree (2026-06-22, funcionou até o próximo update)

Código adicionado diretamente em `/usr/local/lib/hermes-agent/agent/turn_finalizer.py` e `agent/wiki_review.py` no source tree do Hermes. Funcionou inicialmente.

**Por que falhou:** o source tree em `/usr/local/lib/hermes-agent/` é o git repo do hermes-agent. `hermes update` faz `git pull` — qualquer mudança no `turn_finalizer.py` upstream gerava conflito e o git fazia stash das modificações locais. As mudanças ficaram presas em `git stash@{0}` e foram perdidas.

---

### v2 — .hermes/agent/ override com turn_finalizer (2026-06-23, quebrava em todo update)

Gatilho adicionado ao override `/root/.hermes/agent/turn_finalizer.py` (que é o arquivo realmente carregado — ver editable install). `wiki_review.py` em `/root/.hermes/agent/wiki_review.py` (não rastreado, sobrevive).

**Por que falhou:** `turn_finalizer.py` É rastreado pelo git (`git status` mostra `M agent/turn_finalizer.py`). Em todo `hermes update` onde o upstream muda esse arquivo, o git cria conflito e faz stash das mudanças locais. Como `turn_finalizer.py` é um arquivo central que muda frequentemente no upstream, na prática quebrava em todo update.

**Lição:** arquivos rastreados pelo git do hermes-agent nunca são lugar seguro para customizações que precisam sobreviver a updates.

---

### v3 — plugin com estratégia errada de escrita (2026-06-24, não funcionou)

Plugin correto em `~/.hermes/plugins/wiki-review/` (não rastreado, sobrevive a updates). Mas o prompt pedia para o agente **ler o arquivo existente e reescrever tudo de volta** com as novas seções.

**Por que falhou:** o argumento `content` do `write_file` ficava enorme (frontmatter + conteúdo antigo + novas seções). O modelo truncava o JSON no meio da geração. O `message_sanitization` registrava:
```
WARNING: Unrepairable tool_call arguments for write_file — replaced with empty object
```
Resultado: o arquivo nunca era escrito. Aconteceu 10+ vezes nos logs.

**Causa raiz:** contextos pesados (sessões longas com DeepSeek em ~97k tokens) agravavam o problema — stream stale de 311s foi registrado.

---

### v4 — plugin com arquivo temporário (2026-06-24, superado por v5)

Plugin não rastreado + temp file. Corrigiu o bug do write_file. Mas usava `session_id[:8]` para nomear o arquivo — e o session_id no gateway Telegram é o chat_id (fixo), então todos os disparos do dia iam para o mesmo arquivo. Um arquivo acumulou +10 sessões misturadas.

---

### v5 — detecção de sessão por inatividade (2026-06-24, implementação atual)

Plugin não rastreado + estratégia de escrita correta: Python cria o arquivo com frontmatter, agente gera só as novas seções em `/tmp/`, Python faz o append. Sem leitura nem reescrita do arquivo inteiro pelo agente.

Um arquivo por sessão de conversa. Sessão detectada por inatividade (60 min sem turno = nova sessão). Nome do arquivo: `YYYY-MM-DD-HHMM.md`. Estado persistido em `~/.hermes/wiki_review_session.json`.

## Conexões

- [[infraestrutura/hermes.md|Hermes Config]] — stack e arquitetura
- [[conhecimento/wiki.md|Wiki]] — estrutura da wiki e regras de escrita
- [[pendencias/proximos-passos.md|Próximos passos]]
