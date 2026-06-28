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
- index.md: removidos 5 links para diários deletados da seção diary/ e da árvore
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

## [2026-06-28] edit | AGENTS.md — regra de derivação do commit message explicitada
- Bloco Tipos/Escopo reescrito com "Commit message — derivado diretamente da entrada do log: tipo(escopo): descrição"

## [2026-06-28] edit | concepts/plano-implementacao-loop — Curador da Wiki movido para ✅ Validado
- Seção "🚧 A validar" removida (único item validado)
- Nova subsection "Curador da Wiki — v1" em "✅ Validado" com arquitetura real, decisões de design e arquivos da v1
- Frontmatter: timestamp atualizado para 2026-06-28, status draft → stable

## [2026-06-28] edit | concepts/plano-implementacao-loop — Curador da Wiki: separado "automação validada" de "loop validado"
- Restaurada seção "🚧 A validar (como loop)" com framing correto
- Curador v1 descrito como automação funcionando, mas comportamento como loop (cron, idempotência, cobertura) ainda não testado

## [2026-06-28] edit | tools/telegram — consolidação de 4 arquivos em página única
- Criado wiki/tools/telegram.md com todo o conteúdo migrado literalmente
- Deletados: telegram-bot-api.md, telegram-topicos.md, telegram-send-rich-message.md, telegram-reacoes.md
- Atualizado index.md: 4 entradas → 1 (árvore e lista)
- Atualizado curador-wiki.md: links de conexões apontam para telegram.md
- Seções: Chats e IDs / HERMES_SESSION_MESSAGE_ID / Envio de Mensagens / Reações / Integrações

## [2026-06-28] edit | AGENTS.md — append do log sempre no fim do arquivo
- Explicitado que entradas mais recentes vão no final (tail -5 = mais recentes)

## [2026-06-28] edit | AGENTS.md — taxonomia de pastas e correções de conflitos
- Adicionada seção "Taxonomia de pastas" com critérios técnicos e seções obrigatórias por pasta
- Tipos OKF expandidos: adicionados `system` e `tool`; mapeados para pastas correspondentes
- Removida regra 4 de Regras de escrita (diary/ — específico de procedure, não de AGENTS)
- Checklist de Ingest: item 1 reescrito para obrigar verificação de taxonomia antes de criar arquivo
- Lint: adicionadas verificações de pasta errada e type inconsistente com pasta

## [2026-06-28] chore | wiki — refatoração completa da taxonomia de pastas
- Renomeadas/criadas: infraestrutura/ → systems/ e tools/, automacao/ → procedures/ e tools/, conhecimento/ → concepts/, historico/ → history/, pendencias/ → todo/, diario/ → diary/
- Todos os arquivos movidos com git mv (histórico preservado)
- Todos os wikilinks atualizados em todos os arquivos
- index.md: árvore, seções e entradas sincronizadas com nova estrutura
- AGENTS.md: referências a pastas atualizadas (ingest checklist + regra diary/)
- curador-wiki.md e curador-wiki-historico.md: prompts ativos atualizados com novos nomes de pastas
- Referências em history/ e orquestrador.md preservadas como registro histórico

## [2026-06-28] chore | wiki — remoção das pastas antigas após refatoração de taxonomia
- Deletadas: automacao/, conhecimento/, diario/, historico/, infraestrutura/, pendencias/
- .gitkeep movido de diario/ para diary/ (git mv)
- wiki/ agora contém apenas as 7 pastas da nova taxonomia: concepts/, diary/, history/, procedures/, systems/, todo/, tools/

## [2026-06-28] ingest | procedures/auditor-wiki — documentação do auditor v1
- Criado wiki/procedures/auditor-wiki.md: documenta arquitetura 3 fases, escopo dos agentes, 9 pontos de validação
- Criados /root/auditor-wiki-agent-prompt-v1.md e /root/auditor-wiki-coord-prompt-v1.md
- Criado /root/auditor-wiki-v1.sh: script principal bash
- index.md atualizado: entrada adicionada em procedures/

## [2026-06-28] edit | systems/hermes — reestruturação para type: system com seções obrigatórias
- hermes.md: type concept → system; adicionadas seções O que é, Stack e configuração, Interface, Operação, Erros conhecidos, Conexões
- hermes-api.md: type concept → system; description atualizada para refletir papel como seção Interface do sistema
- index.md: descrição de hermes.md atualizada

## [2026-06-28] edit | systems/hermes-api — renomear para hermes-endpoints.md
- hermes-api.md renomeado para hermes-endpoints.md (nome mais preciso — só endpoints REST, não toda a interface)
- Links atualizados em: index.md, hermes.md, elevenlabs-mcp.md, telegram.md, orquestrador.md
- log.md preservado (append-only, entradas históricas mantidas)

## [2026-06-28] edit | procedures/auditor-wiki — reescrita completa para arquitetura v2
- auditor-wiki-v1.sh: reescrito — descoberta dinâmica de arquivos, agentes por pasta, overlap, links, validação Telegram com inline keyboard, correções por finding com commit imediato
- auditor-wiki-agent-prompt-v1.md: adicionados checks 7 (qualidade do nome) e 8 (labels de wikilinks)
- auditor-wiki-coord-prompt-v1.md: output alterado de markdown para JSON (executive_summary + findings com correctable)
- auditor-wiki-corrector-prompt-v1.md: novo — gera old_string/new_string por finding
- auditor-wiki-overlap-prompt-v1.md: novo — detecta sobreposição semântica cross-folder
- auditor-wiki-links-prompt-v1.md: novo — valida wikilinks quebrados e labels
- procedures/auditor-wiki.md: documentação atualizada para refletir arquitetura completa

## [2026-06-28] edit | procedures/auditor-wiki — corrigir numeração de fases (0→1-based) e sugestões de nomes de prompt
- Fases renumeradas: 0–5 → 1–6 no script e na documentação
- auditor-wiki.md atualizado para refletir nova numeração

## [2026-06-28] chore | auditor-wiki — renomear prompts para padrão prompt-awv1-<papel>
- auditor-wiki-agent-prompt-v1.md → prompt-awv1-pasta.md
- auditor-wiki-coord-prompt-v1.md → prompt-awv1-coordenador.md
- auditor-wiki-corrector-prompt-v1.md → prompt-awv1-corretor.md
- auditor-wiki-overlap-prompt-v1.md → prompt-awv1-overlap.md
- auditor-wiki-links-prompt-v1.md → prompt-awv1-links.md
- auditor-wiki-v1.sh e auditor-wiki.md atualizados com novos caminhos

## [2026-06-28] edit | procedures/auditor-wiki.md — adiciona seção de validações pré-produção (V1–V17)
- Restauradas 15 validações perdidas em sessão anterior (V1–V15)
- Corrigida referência "Fase 0" → "Fase 1" em V1
- Corrigido "5 fases" → "6 fases" no parágrafo de abertura
- Adicionadas V16 (autenticação claude CLI standalone) e V17 (dois findings no mesmo arquivo)

## [2026-06-28] edit | procedures/auditor-wiki.md — V9: documentado script de teste inline keyboard
- Expandido item V9 com descrição do script /root/test-v9-inline-keyboard.sh
- Script criado: drena offset, sendMessage com reply_markup, poll callback_query, answerCallbackQuery, editMessageReplyMarkup
- Resultado esperado documentado: status ok, answer_ok True, botões removidos após clique

## [2026-06-28] edit | procedures/auditor-wiki.md — protocolo obrigatório de execução de validações
- Adicionado bloco de aviso com os 4 passos obrigatórios: criar → documentar → commitar → rodar
- Regra explícita: nunca rodar sem documentar antes; autorização do usuário vem após ver a documentação commitada

## [2026-06-28] edit | procedures/auditor-wiki.md — V18: poll_text com filtro de prefixo "!" documentado
- Contexto do conflito Hermes vs auditor documentado (mesmo token, long-polling sem webhook)
- Script criado: /root/test-v18-poll-text-prefix.sh — duas fases + verificação manual do Hermes
- Solução proposta: prefixo "!" nas respostas ao auditor; strip antes de usar como new_string

## [2026-06-28] edit | procedures/auditor-wiki.md — V9 marcado como validado (✅)

## [2026-06-28] edit | procedures/auditor-wiki.md — V18: resultado real documentado (falhou verificação manual)
- Prefixo "!" não protege: Hermes recebeu e respondeu à mensagem, gastando token
- Ambos (script e Hermes) consumiram o mesmo update simultaneamente
- V18 NÃO validado; abordagem de prefixo simples descartada

## [2026-06-28] edit | procedures/auditor-wiki.md — V18b: prefixo "/" documentado como alternativa ao "!"
- Hipótese: "/" trata mensagens como comando no Hermes, sem acionar LLM
- Script criado: /root/test-v18b-slash-prefix.sh
- Verificação manual: observar se Hermes aciona LLM ou só responde "desconhecido"

## [2026-06-28] edit | procedures/auditor-wiki.md — V9 desmarcado (parcial); D1 e D2 documentados
- V9: voltou para ⚠️ — testado com 2 botões, script real envia 3
- D1: fluxo "Ajustar" inviável — usuário não sabe new_string de cabeça; proposta: dropdown com opções válidas + "Outro"
- D2: V9 precisa ser refeito para 3 botões (findings corrigíveis) e 2 botões (não-corrigíveis)

## [2026-06-28] edit | procedures/auditor-wiki.md — V9 consolidado (7 interações); V18/V18b colapsados
- V9 reescrito com script único test-v9-completo.sh cobrindo todas as 7 interações
- V18/V18b colapsados em nota histórica (prefixo "/" validado, incorporado ao V9)
- D2 fechado (coberto pelo V9 completo)

## [2026-06-28] edit | procedures/auditor-wiki.md — V9 reestruturado em V9a/V9b/V9c; fix Recomeçar documentado
- V9 dividido em 3 scripts por tipo de mensagem (resumo, corrigível, não-corrigível)
- Botão Recomeçar (restart/exit 2/exec) adicionado ao Resumo executivo
- poll_text com filtro "/" incorporado ao script real e aos testes
- D1 e D2 atualizados

## [2026-06-28] chore | scripts — renomeados de test-vN para valid-N-papel
- Padrão antigo "test-v9" parecia número de versão; novo padrão: valid-<validação>-<papel>
- valid-9-resumo.sh, valid-9-corrigivel.sh, valid-9-nao-corrigivel.sh
- valid-18-prefixo-exclamacao.sh, valid-18-prefixo-barra.sh (históricos)
- Wiki atualizada com novos caminhos

## [2026-06-28] edit | procedures/auditor-wiki.md — V9a validado (3/3)

## [2026-06-28] edit | procedures/auditor-wiki.md — V9b validado (3/3)

## [2026-06-28] edit | procedures/auditor-wiki.md — V9c validado (2/2); validação 9 completa
- V9c: finding não-corrigível, 2/2 passaram
- Validação 9 inteira concluída: V9a 3/3, V9b 3/3, V9c 2/2
- Fix aplicado: botão Recomeçar, poll_text com filtro /, prompts atualizados

## [2026-06-28] edit | procedures — removida V18 de auditor-wiki.md
- V18 era resquício redundante do V9 (já documentado e validado como V9a/V9b/V9c)

## [2026-06-28] query | auditor-wiki — documentado primeiro run real (desastre)
- Todos os 8 agentes produziram prosa em vez de JSON — findings = []
- Coordenador recebeu input vazio e falhou com parse error
- Mensagem Telegram enviada sem conteúdo útil
- Gastou ~75% do limite Claude free em ~5 minutos (normalmente leva horas para consumir isso)
- Adicionada seção "Primeira execução real — 2026-06-28 (desastre)" com timeline, falhas, consumo, análise de causa raiz e lições
- Nenhuma correção aplicada — só documentação

## [2026-06-28] query | auditor-wiki — análise de consumo de tokens (causa raiz do desastre)
- Adicionada seção "Análise do consumo de tokens" com decomposição do gasto por agente
- Identificados 3 fatores principais: 8 agentes paralelos × multi-turn Read × contexto acumulativo
- Cada agente faz 2-5 Read calls (AGENTS.md + arquivos da pasta), contexto cresce a cada turno
- struct_str (~10KB) enviado a TODOS os agentes mesmo os que só precisam de 1 arquivo
- Overlap/link agents podem ler centenas de KB se ativarem Read em múltiplos arquivos
- Comparação: uso normal = ~5-15 requests/h sequencial; auditor = ~35-50 requests em 5 min paralelo
- Total estimado: ~700KB-1,5MB tokens de entrada + ~40-160KB saída em 8 sessões simultâneas
- Claude Code free tier foi desenhado para uso sequencial, não para 8 sessões paralelas em rajada

## [2026-06-28] edit | procedures/auditor-wiki — corrige suposição de "free tier" para Claude Pro
- 5 ocorrências de "free tier" / "Claude free" removidas do auditor-wiki.md
- Substituídas por linguagem factual: "cota do Claude Code CLI", "consumo em rajada", sem especular sobre plano
- O usuário é assinante Claude Pro — informação incorreta documentada no log histórico (não editado, append-only)

## [2026-06-28] edit | procedures/auditor-wiki — restaura dado factual de 75% que foi removido por engano
- O 75% é dado observado pelo usuário, não suposição de plano — foi restaurado como "~75% da cota disponível"
- O erro anterior foi associar o percentual a "free tier" em vez de mantê-lo como fato
