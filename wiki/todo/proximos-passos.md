---
type: todo
tags: [github, hermes, obsidian, todo, wiki]
title: Próximos Passos
description: Pendências ativas da wiki — migração de conhecimento, AGENTS.md, revisão do SOUL.md e estrutura de diário/raw
timestamp: 2026-06-18T21:00:00-03:00
status: stable
---

## Próximos Passos

### Pendente

0. **[PRIORIDADE ALTÍSSIMA] Corrigir wiki_review para gravar em `diario/` em vez de `diary/`** — o script atualmente grava os diários na pasta `diary/` (que está vazia), mas a pasta real com conteúdo é `wiki/diario/`. Fix necessário antes de qualquer nova execução do wiki_review. *Registrado 2026-06-29.*

0. **[URGENTE] Gráfico do Obsidian Android mostra nós fantasma que não aparecem no PC** — após toda a limpeza de 2026-06-28, o gráfico do celular ainda mostra `viz.html` (múltiplos), `README` (múltiplos) e possivelmente outros nós que não deveriam aparecer. O PC exibe o gráfico corretamente. A causa identificada é que `raw/google-okf/README.md` contém links markdown para arquivos externos (`bundles/*/viz.html`, `samples/*/README.md`) que o Obsidian interpreta como wikilinks internos. Como `raw/` é imutável, o fix deve ser via configuração do Obsidian (filtro no gráfico ou exclusão da pasta `raw/` da indexação). O usuário quer ver TODOS os arquivos no gráfico — não remover raw/, apenas fazer o gráfico parar de mostrar referências externas como nós. Investigar diferença de configuração entre PC e Android (graph.json é gitignored — pode ser que o PC tenha um filtro configurado que o Android não tem). *Registrado 2026-06-28.*

1. **[PESQUISA] Embasar tecnicamente as regras de colaboração para o AGENTS.md** — Definir como um agente deve se comportar diante de bloqueios: (a) persistir na solução certa solicitando insumos ao humano quando necessário, em vez de desistir ou contornar; (b) não repetir abordagens já provadas ineficazes (ex: não executar o mesmo comando com falha mais de 2–3 vezes). Requer pesquisa de literatura técnica e frameworks de agentes antes de formular o texto definitivo para o AGENTS.md. *Registrado 2026-06-28.*

14. **Validar comportamento de `HERMES_SESSION_MESSAGE_ID`** — confirmar se o valor vem defasado, em que condições isso ocorre e qual a confiabilidade da variável para uso em reações no Telegram. Quando validado, documentar o comportamento real em [[tools/telegram-bot-api.md]]. *Registrado 2026-06-27.*

1. **Executar roadmap do Orquestrador da Memória** — [[concepts/orquestrador.md]] contém auditoria completa da wiki com todos os pontos de atenção, débito técnico e ações recomendadas. Ver o arquivo para detalhes. *Registrado 2026-06-26.*

2. **Validar fix do Obsidian Git (merge.autostash)** — `git config merge.autostash true` configurado no Windows em 2026-06-26. Confirmar que Pull funciona consistentemente sem erros de "would be overwritten by merge". Quando validado, mudar [[tools/obsidian-git.md]] de `status: draft` para `status: stable`. *Registrado 2026-06-26.*

3. **Backup GitHub privado do `.hermes`** — criar repo privado `omgiova/hermes-config` com `agent/`, `skills/`, `plugins/`, `scripts/`, `cron/`, `SOUL.md`, `AGENTS.md`, `config.yaml` (excluindo `.env`, `backups/`, `logs/`, `node_modules/`, `node/`, `venv/`). Configurar auto-push via hook ou cron.

4. **Backup geral da VPS** — ativar snapshot automático no painel da Hostinger (KVM 2) ou configurar restic/borg para storage externo. Cobre tudo que o GitHub não cobre (binários, databases, OS).

5. **Migrar conhecimento acumulado** — revisar sessões passadas do Hermes e capturar decisões, gotchas, procedimentos e regras que estão perdidos na memory() ou só na cabeça do usuário.

6. **[RASCUNHO] Unificar skills na wiki** — pesquisar documentação técnica de sistemas existentes (ex: MCP tools + LLM Wiki, skills como páginas `type: procedure`) para que skills do Hermes também vivam na wiki e sejam acessíveis a qualquer agente (Claude Code, Codex, Manus). Não implementar sem validação externa. *Registrado em 2026-06-24.*

7. **Print visual do wiki_review no terminal** — o plugin atualmente só escreve `logger.info`, que não aparece na tela do usuário. Giovani quer um print visível quando o wiki_review dispara e quando termina. Explorar depois que o sistema estiver rodando estável: opções são `on_session_start` hook, gateway callback, ou acesso ao objeto `agent` via algum mecanismo futuro. *Registrado 2026-06-24.*

8. **Limpar source tree do Hermes** — gatilho do wiki_review removido de `agent/turn_finalizer.py` (2026-06-24). Ainda há `AGENTS.md` (redirect) em `/usr/local/lib/hermes-agent/` que pode precisar de limpeza. Confirmar com Giovani antes de marcar como concluído.

9. **Resolver prompt de aprovação invisível no Remote Control** — quando Claude pede aprovação de ferramenta, prompt aparece no Termux (pts/0) mas não no app. Sessão fica aparentemente travada. Investigar configurar permissões automáticas para reduzir aprovações manuais. Ver [[systems/termux-ssh-claude.md]]. *Registrado 2026-06-26.*

10. **Cleanup automático de processos zumbi do Claude** — sessões canceladas pelo app deixam processo Claude vivo no Termux. Implementar script que mata processos órfãos ao iniciar nova sessão. Ver [[systems/termux-ssh-claude.md]]. *Registrado 2026-06-26.*

11. **Detecção periódica de processos zumbi do Claude** — criar mecanismo que monitore com certa frequência processos Claude acumulados em background, evitando consumo silencioso de memória sem o usuário saber. Diferente do item 10 (que age só no início de sessão), este deve rodar de forma contínua ou agendada. Ver [[systems/termux-ssh-claude.md]]. *Registrado 2026-06-26.*

12. **[ESTUDO] Abstracts nas páginas da wiki para otimizar contexto do curador** — cada página wiki ganharia um campo `abstract` no frontmatter (3–5 linhas) descrevendo o conteúdo real da página, não só o título. Permite ao agente curador tomar decisões sem ler o arquivo completo. Diferente do `description` (que é genérico): o abstract responde "isso já está documentado aqui?". Requer: definir padrão, gerar abstracts para as ~20 páginas existentes, e manter abstract atualizado a cada edição da página (via hook ou disciplina de commit). Estudar antes de implementar — é uma mudança estrutural no OKF da wiki. *Registrado 2026-06-27.*

15. **[CONSIDERAR] Hook PostToolUse no Claude Code para automatizar log.md** — adicionar hook no `settings.json` do Claude Code que detecta edições em `/root/wiki/` e appenda entrada no `log.md` automaticamente, mesmo que o agente esqueça de fazer manualmente. Complementa (não substitui) a regra no AGENTS.md. Avaliar após o formato do log estar estável. *Registrado 2026-06-28.*

13. **[ESTUDO] FTS5 como pré-filtro para o agente curador** — antes de chamar o `claude -p`, extrair keywords da daily e consultar SQLite FTS5 para identificar as páginas mais relevantes. Injetar só essas páginas (com conteúdo completo ou abstract) no prompt. Reduz tokens e aumenta precisão do contexto. Depende de: FTS5 estar indexando o conteúdo das páginas, e abstracts existirem para o restante. Estudar em conjunto com item 12. *Registrado 2026-06-27.*

### Concluído

- ~~**Criar `wiki_review.py`**~~ — feito (2026-06-23), arquivo em `/root/.hermes/agent/wiki_review.py`

- ~~**Sincronizar wiki com GitHub**~~ — feito (repo `omgiova/wiki`)
- ~~**Conectar Obsidian**~~ — feito (espelhado no Windows via clone)
- ~~**Criar AGENTS.md**~~ — feito ([[AGENTS.md]])
- ~~**Criar estrutura `diary/`**~~ — feito
- ~~**Criar pasta `raw/`**~~ — feito
- ~~**Revisar SOUL.md**~~ — feito (regra bundled skills + referência à wiki em `/root/wiki/`)
- ~~**Migrar de ai-memory para wiki Karpathy**~~ — feito (Docker parado, MCP desabilitado, hook auto-push configurado)
- ~~**Corrigir curl sem --max-time no script de startup**~~ — resolvido em 2026-06-26: hooks do ai-memory (legado) removidos do `settings.json` e pasta `/root/.local/share/ai-memory/` deletada. Problema não se aplicava (curls já tinham timeout), mas limpeza feita.

## 🔗 Conexões entre projetos
- [[wiki/history/crise-update.md|2026-06-18-crise-update]]
- [[wiki/concepts/wiki.md|wiki (histórico + conceito)]]
- [[wiki/systems/vps.md|vps (IPVS, hardware, stack)]]
- [[wiki/concepts/orquestrador.md|orquestrador — panorama de saúde da wiki e débito técnico]]
