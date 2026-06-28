# log.md

Registro cronológico de operações na wiki. Append-only — nunca editar entradas existentes.

---

## [2026-06-28] edit | AGENTS.md — reestruturação e adição do log.md
- Removida linha de compatibilidade do cabeçalho
- Fundidas seções Autorização e Arquivos NUNCA modificar
- Removidas regras de escrita duplicatas (3 e 7)
- Adicionada instrução de leitura completa antes de qualquer ação
- Incorporado log.md como etapa obrigatória dentro da seção Git e commits

## [2026-06-28] edit | proximos-passos — pendência urgente: gráfico Android com nós fantasma
- Adicionado item 0 (urgente): gráfico Android mostra viz.html/README do raw/ como nós fantasma
- PC exibe corretamente; Android não — diferença provavelmente em graph.json (gitignored)

## [2026-06-28] edit | obsidian-git — removido wikilink quebrado para 2026-06-24-obsidian-git-setup
- Seção Conexões tinha link para arquivo que nunca existiu

## [2026-06-28] edit | wikilinks quebrados — removidos de index.md, crise-update.md, 2026-06-24
- index.md: removidos 5 links para diários deletados da seção diario/ e da árvore
- crise-update.md: convertido wikilink para 2026-06-20.md em texto plano
- 2026-06-24-20260624.md: convertidos 2 wikilinks para agent-loop-architectures.md em texto plano

## [2026-06-28] edit | obsidian-git — nós fantasma no gráfico por wikilinks quebrados
- Documentado: gráfico mostra nós fantasma por wikilinks quebrados, não por arquivos no filesystem
- Erro histórico #11: teoria do usuário sobre links quebrados era correta — foi descartada errado
- Lista de notas com wikilinks quebrados identificada (9 arquivos)
- Duas opções de fix documentadas: ocultar no gráfico ou corrigir os links

## [2026-06-28] edit | obsidian-git — limitações git clean no Android e checklist de limpeza
- Documentado: git clean -fd não remove diretórios não rastreados de forma confiável no FAT (Android)
- Adicionado checklist completo de limpeza manual para Android
- Nota sobre necessidade de force-close do Obsidian após limpeza pelo Termux

## [2026-06-28] edit | obsidian-git — Android FAT filesystem e core.hooksPath documentados
- Android: `storage/shared` é FAT/exFAT, não suporta chmod +x — hook deve ir em `~/git-hooks/` (ext4)
- Solução: `core.hooksPath` no git config aponta para armazenamento interno do Termux
- Erros históricos #9 e #10 adicionados
- Status de instalação atualizado: Windows ✅, Android validação pendente

## [2026-06-28] edit | obsidian-git — solução definitiva post-merge hook documentada
- Adicionada seção "Solução definitiva para arquivos fantasma" com post-merge hook
- Comandos de instalação para Windows e Android
- Tabela de status de instalação por device (ambos pendentes)

## [2026-06-28] edit | obsidian-git — lista de comandos confirmada e erro documentado
- Adicionada lista completa de comandos disponíveis na paleta (Android, confirmada via screenshot)
- Documentado: não existe "Sync Method: Reset" nem equivalente a `git clean` na interface do plugin
- Adicionada seção "Arquivos fantasma" com causa raiz e fixes (Termux ou deleção manual)
- Adicionado erro #8 em "Erros históricos do agente": afirmação falsa sobre "Sync Method: Reset"

## [2026-06-28] edit | wiki-review — comparação com background_review incorporada
- Conteúdo de wiki-review-vs-background-review.md movido para wiki-review.md como seção "Comparação completa com background_review (41 itens)"
- wiki-review-vs-background-review.md deletado
- index.md: entrada removida da árvore de diretórios

## [2026-06-28] edit | wiki-review — origem como clone do background_review documentada no topo
- Adicionado parágrafo de abertura explicando que wiki_review nasceu como clone do background_review.py

## [2026-06-28] edit | AGENTS.md — tipos de commit unificados com log e formato do log corrigido
- Tipos de commit: `docs`, `fix`, `feat` substituídos por `edit`, `ingest`, `query`, `lint`, `session`, `chore`
- Formato do log: `<título>` → `<escopo> — <descrição>` para derivação mecânica do commit message
- Adicionado `chore` na tabela de tipos do log
