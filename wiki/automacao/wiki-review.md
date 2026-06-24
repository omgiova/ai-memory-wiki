---
type: procedure
tags: [hermes, wiki, automacao, background, diario]
title: Wiki Review — Auto-escrita de diário em background
description: Agente clone do background_review nativo do Hermes que roda a cada 10 turnos e salva insights da conversa na wiki (diario/).
timestamp: 2026-06-23T00:00:00+00:00
status: stable
---

# Wiki Review

Processo em background que analisa a conversa automaticamente e escreve insights no diário da wiki (`diario/YYYY-MM-DD.md`). Roda sem intervenção do usuário, a cada 10 turnos de conversa.

## Como funciona

1. `turn_finalizer.py` é chamado ao fim de cada turno do Hermes
2. Verifica `_should_review_memory` — o mesmo gatilho do `background_review` nativo (a cada `_memory_nudge_interval` turnos, padrão 10)
3. Quando o gatilho dispara, wiki_review roda **no mesmo turno** que o background_review de memória
4. A thread instancia um `AIAgent` filho (fork do agente principal) com toolset restrito a `file`
5. O agente filho lê o histórico da conversa e escreve seções no diário
6. Após o agente terminar, o código Python faz `git add -A && git commit` diretamente
7. O hook `post-commit` de `/root/wiki/` faz `git pull --rebase + push` automaticamente

```
turno N → finalize_turn() → _should_review_memory == True
                                    ↓ (mesmo gatilho do background_review)
                          spawn_wiki_review_thread()
                                    ↓ thread daemon
                          AIAgent(toolset=["file"])
                          → run_conversation(prompt, histórico)
                          → write_file(diario/YYYY-MM-DD-*.md)
                                    ↓ após agente terminar
                          git add -A && git commit
                                    ↓ hook post-commit
                          git pull --rebase + git push
```

## Arquivos envolvidos

| Arquivo | Papel |
|---|---|
| `/root/.hermes/agent/turn_finalizer.py` | Gatilho — verifica `_should_review_memory` e dispara a thread |
| `/root/.hermes/agent/wiki_review.py` | Toda a lógica: agente filho, git commit |
| `/root/wiki/diario/YYYY-MM-DD-*.md` | Saída — daily notes escritas pelo agente |
| `/root/wiki/.git/hooks/post-commit` | Hook que faz push automático após commit |

## Configuração

O intervalo de disparo é o mesmo do background_review nativo: `agent._memory_nudge_interval` (padrão 10 turnos, configurável em `config.yaml` via `memory.nudge_interval`).

```python
# em wiki_review.py
WIKI_DIR = Path("/root/wiki")
DIARIO_DIR = WIKI_DIR / "diario"
```

## O prompt (3 perguntas)

O agente filho recebe o histórico da conversa e responde a 3 perguntas:

1. **O usuário revelou algo sobre si?** — persona, desejos, preferências, estilo de trabalho
2. **Teve correção, técnica nova, ferramenta instalada, skill desatualizada?** — mudanças de abordagem, novos workflows, configurações
3. **Tem pendência ou próximo passo?** — decisões adiadas, issues abertas, caminhos não explorados

Cada finding vira uma seção `## {título} — {HH:MM}` no diário do dia.

## Nome do arquivo de saída

```
diario/YYYY-MM-DD-{8 chars do session_id}.md
```

Exemplo: `diario/2026-06-23-20260623.md`

Se o arquivo já existir (outro disparo no mesmo dia), o agente lê o conteúdo atual e reescreve com as novas seções anexadas (read → append → write, já que `write_file` não tem modo append nativo confiável).

## Restrições do agente filho

- Toolset: apenas `file` (read_file, write_file)
- `skip_memory=True` — não acessa nem escreve na `memory()` do Hermes
- `max_iterations=16`
- `compression_enabled=False`
- Qualquer tool fora do whitelist é negada com log de warning

## Histórico e bugs corrigidos (2026-06-23)

**Bug 1 — contador resetava a cada restart do gateway**
O contador era uma variável global Python (`_wiki_review_turn_counter = 0`). Cada restart do gateway (update, crash) zerava o valor. Corrigido: contador agora persiste em `/root/.hermes/wiki_review_counter`.

**Bug 2 — prompt contraditório sobre git**
O `_WIKI_REVIEW_PROMPT` dizia "só use write_file e read_file", mas o código concatenava depois "você pode usar terminal para git commit/push". O modelo seguia uma ou outra instrução aleatoriamente, resultando em arquivos escritos mas não commitados. Corrigido: git removido do escopo do LLM — agente filho só escreve o arquivo, commit é feito em Python após o agente terminar.

## Conexões

- [[wiki/infraestrutura/hermes.md|Hermes Config]] — onde os arquivos modificados vivem (`agent/`)
- [[wiki/conhecimento/wiki.md|Wiki]] — estrutura do vault e regras de escrita
- [[wiki/pendencias/proximos-passos.md|Próximos passos]]
