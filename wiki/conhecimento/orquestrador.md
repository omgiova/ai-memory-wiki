---
type: concept
tags: [wiki, manutencao, okf, padrao, auditoria, karpathy, memoria]
title: Orquestrador da Memória — Panorama de Saúde da Wiki
description: Auditoria profunda e estruturada da wiki — mapa de todos os pontos de atenção em estrutura, OKF, diário, automação, coordenação de agentes e cross-links, com roadmap de execução.
timestamp: 2026-06-26T14:00:00-03:00
status: stable
---

# Orquestrador da Memória — Panorama de Saúde da Wiki

> Este documento é o mapa de saúde da wiki. Cada seção é uma área de atenção — com diagnóstico preciso, referência ao arquivo afetado e ação recomendada. Use-o como guia de manutenção contínua.

---

## Estado Geral

A wiki está **funcionalmente operacional**: OKF frontmatter presente, Git auto-push ativo, wiki_review rodando, index.md navegável, AGENTS.md como schema. O padrão Karpathy foi seguido corretamente desde a fundação.

O que existe agora é **débito técnico acumulado** — inconsistências pequenas que sozinhas não travam nada, mas juntas aumentam a carga cognitiva dos agentes e degradam a confiabilidade da wiki como fonte da verdade. Este documento mapeia tudo.

**Escopo:** 25 páginas wiki + 10 entradas de diário + 4 arquivos raw + AGENTS.md + index.md.  
**Data da auditoria:** 2026-06-26.

---

## 1. Conformidade OKF

**Objetivo:** garantir que o bundle seja conformante com OKF v0.1 em todos os pontos da spec.

### 1.1 — index.md com frontmatter completo (viola §6 da spec)

**Arquivo:** `index.md`  
**Problema:** O OKF SPEC §6 é explícito: "Index files contain no frontmatter." A única exceção é `okf_version: "0.1"` na raiz do bundle (§11). Nosso `index.md` tem frontmatter completo (`type`, `title`, `description`, `timestamp`, `status`) — viola a spec.  
**Ação:** Remover todo o frontmatter do `index.md`, deixando só o corpo. Se quiser declarar a versão OKF, adicionar apenas `okf_version: "0.1"` numa linha de frontmatter mínima.

### 1.2 — okf_version não declarado em nenhum lugar

**Arquivo:** `index.md`  
**Problema:** A spec §11 define que bundles *podem* declarar a versão com `okf_version: "0.1"` no `index.md` raiz. Atualmente nada declara a versão, o que impede consumidores de saber qual versão do OKF o bundle segue.  
**Ação:** Após corrigir 1.1, adicionar `okf_version: "0.1"` como único campo de frontmatter no `index.md`.

### 1.3 — Cross-links em formato Obsidian, não em Markdown padrão

**Arquivos:** Todos (diário, infraestrutura, automação, conhecimento).  
**Problema:** O OKF §5 define dois formatos válidos de cross-link: absoluto bundle-relative (`[texto](/caminho/arquivo.md)`) e relativo (`[texto](./outro.md)`). Nossa wiki usa Obsidian wikilinks (`[[path/arquivo.md|texto]]`). Isso é legível no Obsidian, mas não é cross-link OKF — um consumidor OKF genérico não constrói grafo a partir de wikilinks.  
**Impacto:** Agentes não-Obsidian (ex: futuro visualizador de grafo OKF) não reconhecerão os links. A spec toleraria isso (§9 diz "consumidores devem tolerar links quebrados"), mas o valor do grafo se perde.  
**Decisão a tomar:** Manter wikilinks (prioridade Obsidian) ou migrar para markdown standard (prioridade portabilidade OKF). Não há resposta certa — mas a decisão deve ser documentada no AGENTS.md como escolha intencional.

### 1.4 — Campo `resource` ausente em páginas de recursos reais

**Arquivos:** `infraestrutura/hermes-api.md`, `infraestrutura/vps.md`, `infraestrutura/telegram-topicos.md`  
**Problema:** OKF §4.1 recomenda `resource: <URI>` para conceitos vinculados a ativos reais. `hermes-api.md` descreve endpoints reais (tem um `/openapi.json` de origem). `vps.md` descreve um servidor com IP fixo. `telegram-topicos.md` tem chat IDs e thread IDs.  
**Ação:** Adicionar `resource:` nas três páginas. Exemplos:
  - `hermes-api.md` → `resource: http://2.24.121.135:8000/openapi.json`
  - `vps.md` → `resource: ssh://root@2.24.121.135`
  - `telegram-topicos.md` → `resource: https://t.me/c/...` (quando IDs forem preenchidos)

### 1.5 — Nenhum `log.md` existe na wiki

**Problema:** O OKF §7 define `log.md` como registro cronológico de mudanças — append-only, mais recente primeiro. Atualmente o `diario/` cumpre parte dessa função, mas de forma episódica e não estruturada. Um `log.md` na raiz da wiki registraria ingest, lint, reestruturação e marcos importantes de forma parseável.

**Implementação detalhada:**

**Localização:** `/root/wiki/log.md` — raiz da wiki, ao lado de `index.md` e `AGENTS.md`. Na raiz porque cobre a wiki inteira.

**Frontmatter:** nenhum. OKF §3.1 define `log.md` como arquivo reservado sem frontmatter — exceção explícita à regra geral da wiki.

**Formato:**
```markdown
# Wiki Update Log

## YYYY-MM-DD
* **Tipo:** [Título](/caminho/relativo.md) — uma linha descrevendo a mudança

## YYYY-MM-DD
* ...
```
Grupos de data em ISO 8601, mais recente primeiro. Múltiplas entradas por dia são normais.

**Tipos de entrada (exaustivo — sem ambiguidade):**

| Tipo | Quando usar |
|---|---|
| `Creation` | arquivo novo criado na wiki |
| `Update` | conteúdo significativo alterado em página existente |
| `Deprecation` | página marcada como `status: deprecated` |
| `Rename` | arquivo renomeado ou movido para outro diretório |
| `Deletion` | arquivo deletado da wiki |
| `Promotion` | entrada do diário promovida a página permanente (custom nosso) |

**O que NÃO registrar** (deve estar explícito no AGENTS.md para evitar ruído):
- Correção de typo ou gramática
- Atualização de `timestamp` no frontmatter
- Adição ou correção de wikilinks
- Ajuste de tags
- Commit de push/sync sem mudança de conteúdo

**Quando aciona:** o `log.md` deve ser um passo obrigatório no checklist de Ingest do AGENTS.md — após commitar, antes de encerrar a operação. Toda criação, renomeação, deleção e promoção de página gera entrada. Updates apenas quando o conteúdo mudou de forma substancial.

**Retroativo:** criar o log já com os marcos históricos da wiki desde a fundação (2026-06-18), reconstruídos via `git log --oneline`. Não precisa ser exaustivo — só os eventos de criação de páginas e reestruturações relevantes.

**Não confundir com o diário:** o `log.md` é sobre a **wiki** (o que foi criado, alterado, removido), não sobre sessões ou conversas. O diário é memória episódica; o log é changelog estrutural.

---

## 2. Estrutura da Wiki — Links Quebrados e Sync

### 2.1 — Link quebrado crítico no index.md

**Arquivo:** `index.md`, linha 19  
**Problema:** A entrada da seção Infraestrutura aponta para `wiki/infraestrutura/pendencia-problema-ssh-claude.md` — este arquivo **não existe**. O arquivo real é `wiki/infraestrutura/termux-ssh-claude.md`.  
**Ação:** Corrigir o link no `index.md` para `[[wiki/infraestrutura/termux-ssh-claude.md|Problema SSH/Claude]]`.

### 2.2 — AGENTS.md e index.md não estão em sync

**Arquivos:** `AGENTS.md` (linhas 85-86), `index.md` (linha 19)  
**Problema:** O AGENTS.md lista `termux-ssh-claude.md` corretamente na árvore da wiki. O `index.md` lista `pendencia-problema-ssh-claude.md` (nome errado). Estão fora de sync — viola a regra do próprio AGENTS.md ("esta seção deve estar sempre sincronizada com `git ls-files`").  
**Ação:** Corrigir o `index.md` (ver 2.1). Verificar se existem outras discrepâncias com `git ls-files`.

### 2.3 — wiki/diario/.gitkeep e wiki/raw/.gitkeep desnecessários

**Arquivos:** `wiki/diario/.gitkeep`, `wiki/raw/.gitkeep`  
**Problema:** Os `.gitkeep` existem para rastrear pastas vazias. `wiki/diario/` já tem 10 arquivos. `wiki/raw/` tem `.gitkeep` dentro da pasta, mas a pasta `raw/` em si tem conteúdo. São arquivos mortos.  
**Ação:** Deletar ambos. Não aparecem em `git ls-files` como arquivos relevantes, mas poluem `git status`.

### 2.4 — Página de perfil do usuário prometida mas não criada

**Arquivo:** Deveria existir `wiki/perfil/giovani.md`  
**Problema:** O diário de 2026-06-24 registra (09:09): "Giovani perguntou 'qual arquivo da wiki é sobre mim' — ofertei criar `wiki/perfil/giovani.md`". A decisão ficou em aberto. Há 3 arquivos de perfil espalhados em sistemas diferentes (`/root/.hermes/memories/USER.md`, `/root/.claude/projects/-root/memory/user_profile.md`, `/root/hermes-user.backup.md`) — nenhum deles na wiki.  
**Problema secundário:** O `/root/hermes-user.backup.md` contém dados inventados ("engenheiro civil") — confirmado no diário como incorreto.  
**Ação dupla:** (1) Criar `wiki/perfil/giovani.md` como fonte da verdade do perfil do usuário. (2) Deletar `/root/hermes-user.backup.md` após aprovação.

### 2.5 — Pasta `wiki/pendencias/` tem apenas um arquivo

**Arquivo:** `wiki/pendencias/proximos-passos.md`  
**Problema:** Estrutura de pasta para arquivo único é ruído. À medida que a wiki crescer, faz sentido — mas agora é overhead.  
**Opção:** Não é urgente. Mas se a pasta nunca ganhar mais arquivos, cogitar mover `proximos-passos.md` para a raiz de `wiki/` e remover a pasta. Registrar como decisão diferida.

---

## 3. Padronização de Frontmatter

### 3.1 — Tags em inglês misturadas com tags em português

**Problema geral:** As regras do AGENTS.md dizem "tags em kebab-case, no plural, sem acentos". Mas não define idioma. O resultado é uma mistura:
  - Inglês: `identity` (`hermes.md`), `reference` (`wiki-review-vs-background-review.md`), `concept` (vários), `harness` (`agent-loop-architectures.md`), `bug` (`termux-ssh-claude.md`)
  - Português: `sessoes`, `padrao`, `automacao`, `diario`, `manutencao`

**Impacto:** Busca por tag falha quando o agente usa o idioma errado. `grep -r "tags:" wiki/` vai mostrar a inconsistência.  
**Ação:** Definir no AGENTS.md se tags são em PT-BR ou EN. Recomendação: **PT-BR** (idioma do usuário e da maioria das tags existentes). Migrar as tags em inglês:
  - `identity` → `identidade`
  - `reference` → `referencia`
  - `harness` → `frameworks`
  - `bug` → `bugs`

### 3.2 — Campo `type` usa valores inconsistentes

**Problema:** O tipo `reference` aparece em `wiki-review-vs-background-review.md`. Os tipos válidos definidos no AGENTS.md são: `concept`, `procedure`, `session`, `todo`, `index`, `raw`, `daily`. O tipo `reference` não está na lista.  
**Ação:** Mudar `type: reference` para `type: concept` em `wiki-review-vs-background-review.md`, que é uma tabela comparativa — cabe em `concept`.

### 3.3 — Timestamps inconsistentes com o arquivo

**Arquivo:** `historico/crise-update.md`  
**Problema:** `timestamp: 2026-06-17T21:00:00-03:00` mas o título é "Sessão: 2026-06-18". A sessão aconteceu em 18/06, o timestamp diz 17/06.  
**Ação:** Corrigir `timestamp` para `2026-06-18T00:00:00-03:00`.

**Arquivo:** `wiki/diario/2026-06-20.md`  
**Problema:** `timestamp: 2026-06-19T21:00:00-03:00` para uma daily note de 2026-06-20. O timestamp em BRT reflete a meia-noite (20/06 às 00:00 BRT = 19/06 às 21:00 UTC) — tecnicamente correto mas confuso. Considerar usar início do dia BRT (`2026-06-20T00:00:00-03:00`).

### 3.4 — Status `draft` em páginas que deveriam ser `stable`

**Arquivos com `status: draft` que são estáveis:**
  - `wiki/conhecimento/agent-loop-architectures.md` — conteúdo completo e revisado, pesquisa profunda realizada, comparativo com código-fonte real. Pode mudar para `stable`.
  - `wiki/diario/2026-06-24-20260624.md` — diários ficam em `draft` por convenção (inbox), mas o conteúdo está finalizado há dias. Considerar `stable` para diários com mais de 24h sem alterações.

### 3.5 — Datas hardcoded no corpo das páginas

**Arquivo:** `wiki/conhecimento/wiki.md` (linhas 11-12)  
**Problema:** "Criado: 2026-06-17 / Última atualização: 2026-06-19" como texto plano no corpo. O `timestamp` no frontmatter já serve para isso — duplicação que vai ficar desatualizada.  
**Ação:** Remover as linhas de data do corpo. O frontmatter é a fonte de verdade de timestamps.

### 3.6 — Diário 2026-06-23 com tags mínimas

**Arquivo:** `wiki/diario/2026-06-23-20260623.md`  
**Problema:** `tags: [sessao]` — singular em vez do padrão plural (`sessoes`). Além disso, tags muito genéricas para uma entrada que tem conteúdo técnico específico (dashboard, basic-auth, hermes).  
**Ação:** Corrigir para `tags: [sessoes, hermes, dashboard, basic-auth]`.

---

## 4. Diário & Memória Episódica

### 4.1 — Nomenclatura inconsistente dos arquivos de diário

**Problema:** Três padrões de nome coexistem na pasta `wiki/diario/`:
  - `YYYY-MM-DD.md` → `2026-06-19.md`, `2026-06-20.md` (padrão antigo)
  - `YYYY-MM-DD-HHMM.md` → `2026-06-24-1710.md` (wiki_review v5/v6)
  - `YYYY-MM-DD-YYYYMMDD.md` → `2026-06-22-20260621.md`, `2026-06-22-20260622.md` (artefato de migração)
  - Nomes temáticos: `2026-06-22-reacoes-telegram.md`, `2026-06-22-descoberta-id-mensagem.md`

**Impacto:** Agentes não conseguem construir sequência cronológica por nome. O wiki_review assume um padrão; entradas manuais seguem outro.  
**Ação:** Definir **um** padrão no AGENTS.md. Recomendação: `YYYY-MM-DD-HHMM.md` para entradas automáticas (wiki_review), `YYYY-MM-DD.md` para daily notes manuais. Arquivos com HHMM duplicado no nome (`2026-06-22-20260621.md`) devem ser renomeados para o padrão correto.

### 4.2 — Entradas temáticas que não são daily notes

**Arquivos:** `wiki/diario/2026-06-22-reacoes-telegram.md`, `wiki/diario/2026-06-22-descoberta-id-mensagem.md`  
**Problema:** Estes arquivos são descobertas técnicas específicas (teste de reações no Telegram, técnica de descoberta sequencial de message-id), não daily notes episódicas. Estão no `diario/` mas deveriam ser páginas permanentes.  
**Ação:** Mover para `wiki/conhecimento/` ou `wiki/infraestrutura/` com frontmatter adequado. O índice deve ser atualizado. Se o conteúdo já está coberto em `telegram-topicos.md`, verificar redundância antes de mover.

### 4.3 — Sem processo de promoção diário → página permanente

**Problema:** O próprio `index.md` (seção Diário) alerta: "Não é destino final — insights estáveis devem ser extraídos e integrados em páginas permanentes." Mas **não existe workflow definido** para isso no AGENTS.md. O diário de 24/06 tem 562 linhas de insights, muitos já estabilizados, mas nenhum foi promovido.  

**Conteúdo do diário 2026-06-24 que deveria ser promovido:**
  - Descobertas sobre ElevenLabs SFX (loop=true, duration_seconds, créditos) → página `wiki/conhecimento/elevenlabs-sfx.md` ou skill
  - Perfil técnico do Giovani (método socrático, barreira de competência, autonomia sobre fontes) → `wiki/perfil/giovani.md`
  - Arquitetura de loop do OpenClaw (já promovida para `agent-loop-architectures.md` ✅)
  - Peter Steinberger (@steipete) e o tweet "design loops, not prompts" → merece menção em `agent-loop-architectures.md`
  - Problema de escrita concorrente entre instâncias → issue de arquitetura, pode ir em `wiki/automacao/wiki-review.md`

**Ação:** Criar operação "Promote" no AGENTS.md (ao lado de Ingest, Query, Lint): varrer o diário a cada X dias, extrair insights estáveis, criar ou enriquecer páginas permanentes, e marcar a entrada do diário como promovida com `<!-- promoted: 2026-06-XX -->`.

### 4.4 — Diário com pendências resolvidas não marcadas

**Arquivo:** `wiki/diario/2026-06-24-20260624.md`  
**Problema:** O arquivo tem dezenas de seções "## Pendência: X" onde X foi resolvido na mesma sessão (ou sessões seguintes). Por exemplo: "Pendência: tweet do Peter Steinberger visto e discutido — 11:20" aparece como resolução, mas ainda há 5+ seções de pendência sobre o mesmo tema. O arquivo acumulou um histórico de pensamento, não uma lista limpa de abertos.  
**Ação:** Não é necessário limpar retroativamente (seria perda de contexto histórico). Mas o processo de promoção (4.3) deve **não promover** pendências já resolvidas — só insights estáveis.

### 4.5 — Escrita concorrente entre instâncias de Hermes

**Problema documentado:** O diário de 24/06 (10:22) registra: "múltiplas instâncias do Hermes escrevendo no mesmo arquivo simultaneamente — produziu duplicatas de seções inteiras e formatação inconsistente". O arquivo `2026-06-24-20260624.md` tem seções duplicadas como evidência.  
**Impacto:** Corrupção de conteúdo, duplicação, inconsistência de formatação.  
**Ação de curto prazo:** Nada a fazer agora (o wiki_review v5/v6 usa arquivo temp + merge feito pelo Python — isso já mitiga parte do problema). A questão persiste quando dois agentes humanos (Hermes em dois tópicos Telegram) escrevem ao mesmo tempo.  
**Ação de longo prazo:** Registrar como item em `proximos-passos.md`. Solução possível: fila de escrita com lock de arquivo, ou cada tópico Telegram ter seu próprio arquivo de diário.

---

## 5. wiki_review — Automação

### 5.1 — Documentação em v5 mas implementação em v6

**Arquivo:** `wiki/automacao/wiki-review.md`  
**Problema:** O header diz "Implementação atual (v6 — sessão por session_id, contador por sessão)" mas a seção de fluxo logo abaixo descreve v4/v5 ("Nome do arquivo: `YYYY-MM-DD-HHMM.md`", "estado em `wiki_review_session.json`"). A seção de "implementação anterior (v4/v5)" tem mais conteúdo técnico que a v6.  
**Ação:** Atualizar a seção principal para descrever v6 corretamente. O que mudou em v6: dicionário `{session_id: {diary_path, last_activity, counter}}` vs o JSON plano da v5. Verificar `/root/.hermes/plugins/wiki-review/__init__.py` para confirmar o estado atual e documentar fielmente.

### 5.2 — Tabela de parâmetros com valor desatualizado

**Arquivo:** `wiki/automacao/wiki-review.md`, linha 97 (tabela de Parâmetros)  
**Problema:** A tabela mostra `memory.nudge_interval` com "Padrão: 10". O valor real no config.yaml é `2`. O diário de 24/06 (10:47) documenta exatamente esse erro: "documentação estava desatualizada mostrando '10' como valor real".  
**Ação:** Corrigir a tabela para mostrar o valor configurado (`2`) e distinguir entre valor padrão do código (`10`) e valor configurado no `config.yaml` (`2`). Adicionar nota: "verificar sempre o config.yaml em runtime — o padrão do código difere do valor configurado".

### 5.3 — Sem workflow de promoção de diário no plugin

**Arquivo:** `wiki/automacao/wiki-review.md`  
**Problema:** O plugin captura insights mas não tem mecanismo de promoção para páginas permanentes. O gap entre "capturar no diário" e "integrar na wiki" é manual e não documentado.  
**Ação:** Criar seção "Limitações conhecidas / Roadmap" na página do wiki-review documentando: (1) escrita concorrente, (2) ausência de promoção automática, (3) prompt só captura, não sintetiza entre sessões.

### 5.4 — wiki-review-vs-background-review.md com itens ❌ pendentes sem followup

**Arquivo:** `wiki/automacao/wiki-review-vs-background-review.md`  
**Problema:** A tabela tem 4 itens `❌ pendentes` de alta prioridade (itens #15+16 cache parity, #22 digest logic, #3+26+32 cleanup defensivo). Foram identificados em 2026-06-24 e ainda não resolvidos. A comparação é rigorosa mas a resolução não foi acompanhada.  
**Ação:** Adicionar um campo `last_reviewed:` no frontmatter e uma seção de status de cada pendência. Ou mover os ❌ para `proximos-passos.md` com data e contexto.

---

## 6. Coordenação entre Agentes

### 6.1 — Três arquivos de perfil do usuário, zero coordenação

**Problema:** O diário de 24/06 (09:16) documenta a fragmentação:
  - `/root/.hermes/memories/USER.md` — fonte de verdade do Hermes, acurada
  - `/root/.claude/projects/-root/memory/user_profile.md` — desatualizada ("leigo", "quer autorizar cada ação")
  - `/root/hermes-user.backup.md` — dados inventados ("engenheiro civil"), identificada como incorreta

**Impacto:** Claude Code e Hermes têm visões diferentes do usuário. A cada nova sessão, o agente pode agir com premissas erradas.  
**Ação:** (1) Criar `wiki/perfil/giovani.md` como fonte de verdade compartilhada entre todos os agentes. (2) Atualizar `user_profile.md` do Claude Code para apontar para a wiki como referência. (3) Deletar `/root/hermes-user.backup.md`. (4) Mencionar no AGENTS.md que `wiki/perfil/giovani.md` é o perfil canônico.

### 6.2 — AGENTS.md proíbe modificação mas Claude Code ignora Hermes e vice-versa

**Arquivo:** `AGENTS.md`, seção "Como os agentes usam este repositório"  
**Problema:** A tabela lista Hermes com acesso via `memory_query/memory_read_page/memory_write` (MCP tools). Mas o `ai-memory` MCP está **desabilitado** (`wiki/infraestrutura/hermes.md`: "ai-memory — disabled — não usar"). Hermes lê a wiki via `read_file / search_files`, não via MCP.  
**Ação:** Atualizar a tabela de agentes no AGENTS.md para refletir o método real de acesso do Hermes.

### 6.3 — Sem protocolo de handoff entre sessões de Claude Code

**Problema:** Claude Code usa `/root/.claude/projects/-root/memory/` como memória persistente, mas não há instrução no AGENTS.md de como integrar com a wiki. Cada sessão de Claude Code começa com o contexto do MEMORY.md, não com a wiki completa.  
**Ação:** Adicionar no AGENTS.md uma seção "Claude Code — uso da wiki" especificando: (1) ler `index.md` antes de qualquer operação, (2) `wiki/perfil/giovani.md` é o perfil canônico do usuário, (3) não duplicar na memória do Claude Code informações que já estão na wiki.

---

## 7. Páginas Faltantes & Gaps de Conteúdo

### 7.1 — Sem página de perfil do usuário

**Gap:** `wiki/perfil/giovani.md` não existe.  
**Conteúdo a incluir** (extraído de diários e hermes.md):
  - Quem é (produtor musical experimental, curiosidade técnica, mão-na-massa)
  - Estilo de trabalho (método socrático de ensino, barreira de competência, loop em vez de prompt)
  - Preferências de comunicação (ação > explicação, sem follow-up não solicitado, sem suposição)
  - Stack pessoal (Hermes, Telegram, VPS, n8n, Node-RED, ElevenLabs)
  - Projetos ativos

### 7.2 — Descobertas de ElevenLabs SFX sem página permanente ✅

**Resolvido em 2026-06-27:** criada `wiki/infraestrutura/elevenlabs-mcp.md` com capabilities, limites do free tier, comportamento de `duration_seconds`, `loop=true`, créditos e prompt language. Linkada de `hermes.md` (MCP Servers) e `hermes-api.md`.

### 7.3 — Técnica de message-id do Telegram sem página consolidada

**Gap:** `wiki/diario/2026-06-22-descoberta-id-mensagem.md` existe mas está no diário, não no conhecimento. `telegram-topicos.md` tem IDs mas não tem a técnica de descoberta sequencial.  
**Ação:** Integrar a técnica em `telegram-topicos.md` numa nova seção "Técnicas de descoberta de IDs".

### 7.4 — Sem documentação de configuração do Firecrawl no Hermes

**Gap:** `wiki/automacao/firecrawl.md` descreve uso do Firecrawl (sintaxe site:, quando usar). Mas o diário de 24/06 documenta um gotcha crítico: `web.search_backend` e `web.extract_backend` estavam vazios no config.yaml mesmo com a API key presente — ferramenta não funcionava.  
**Ação:** Adicionar seção "Configuração no Hermes" em `firecrawl.md` com: variáveis do .env, campos do config.yaml que precisam ser setados, e o pitfall "key no .env não é suficiente — setar o backend também".

### 7.5 — Tweet "design loops, not prompts" sem registro permanente

**Gap:** O tweet do Peter Steinberger (@steipete) é citado extensamente no diário como paradigma central de design de agentes. Mas `agent-loop-architectures.md` não o menciona.  
**Ação:** Adicionar seção "Filosofia: Design Loops, Not Prompts" em `agent-loop-architectures.md` com o tweet, a análise (prompt = dar corda no pião; loop = relógio que anda sozinho) e o paralelo com o comportamento real do usuário (repetição do mesmo comando como loop).

### 7.6 — Técnica de pesquisa factual vs temática sem registro

**Gap:** O diário de 24/06 (10:19-10:20) documenta uma distinção importante: "pesquisa temática" (`openclaw agent loop`) vs "pesquisa factual" (`openclaw creator founder`). Cada tipo exige query diferente. Essa técnica não está em nenhuma página permanente.  
**Ação:** Adicionar em `firecrawl.md` uma seção "Tipos de busca" que documente a distinção.

---

## 8. Cross-links & Grafo

### 8.1 — Páginas órfãs (sem links apontando para elas)

Auditoria de quem linka para quem:

| Página | Recebe links de |
|---|---|
| `automacao/wiki-review-vs-background-review.md` | Apenas de `wiki-review.md` |
| `historico/2026-06-22-modelos-nim-elevenlabs.md` | Apenas do `index.md` |
| `infraestrutura/telegram-topicos.md` | Apenas do `index.md` |
| `infraestrutura/hermes-api.md` | Apenas do `index.md` |
| `infraestrutura/termux-ssh-claude.md` | `index.md` (com link errado), `pendencias/proximos-passos.md` |

**Páginas de diário:** Nenhuma das daily notes recebe links de páginas permanentes (apenas do index.md). Isso é esperado — diários são inbox.  
**Ação:** `historico/2026-06-22-modelos-nim-elevenlabs.md` e `infraestrutura/telegram-topicos.md` e `hermes-api.md` merecem ser linkados de `infraestrutura/hermes.md` nas seções relevantes (modelos, telegram, API).

### 8.2 — Wikilinks com prefixo relativo inconsistente

**Problema:** Alguns links usam o prefixo completo `wiki/pasta/arquivo.md`, outros usam apenas `pasta/arquivo.md` (relativo à pasta `wiki/`). Exemplos:
  - `wiki/conhecimento/wiki.md` usa `[[wiki/infraestrutura/vps.md|vps]]`
  - `wiki/automacao/wiki-review.md` usa `[[infraestrutura/hermes.md|Hermes Config]]` (sem `wiki/`)
  - `wiki/diario/2026-06-24-20260624.md` usa `[[pendencias/proximos-passos.md]]` (sem `wiki/`)

O Obsidian resolve os dois, mas consumidores OKF ou buscas por path absoluto falhariam no segundo caso.  
**Ação:** Padronizar todos os wikilinks internos com o prefixo `wiki/` explícito, ou todos sem. Recomendação: **sem prefixo** (caminho relativo à pasta `wiki/`) é mais limpo e ainda funciona no Obsidian, mas tornar isso uma regra explícita no AGENTS.md.

### 8.3 — `proximos-passos.md` numerado fora de ordem

**Arquivo:** `wiki/pendencias/proximos-passos.md`  
**Problema:** Os itens estão numerados: 1, 2, 3, 6, 5, 4, 8, 9, 10. Os números foram atribuídos ad hoc conforme os itens foram adicionados em sessões diferentes, nunca renumerados.  
**Impacto:** Agentes que se referem a "item 4" podem estar falando de coisas diferentes. O diário usa "item #6", o `termux-ssh-claude.md` usa "item 8".  
**Ação:** Renumerar sequencialmente. Atualizar referências nos diários e em `termux-ssh-claude.md`.

---

## 9. Nomenclatura & Convenções

### 9.1 — Convenção de nomes de arquivo inconsistente

**Observação:** A wiki tem dois padrões coexistindo:
  - **kebab-case** (maioria): `agent-loop-architectures.md`, `wiki-review-vs-background-review.md`, `proximos-passos.md`
  - **snake_case parcial** (nenhum, eliminado)
  - **Datas**: `YYYY-MM-DD.md` vs `YYYY-MM-DD-HHMM.md` vs `YYYY-MM-DD-YYYYMMDD.md` (ver 4.1)

O padrão kebab-case é consistente nas páginas não-diário. O problema está só no diário — coberto em 4.1.

### 9.2 — Título "Diário YYYY-MM-DD-YYYYMMDD" como `title` no frontmatter

**Arquivos:** `2026-06-22-20260621.md`, `2026-06-22-20260622.md`  
**Problema:** `title: Diário 2026-06-22-20260621` inclui o timestamp duplicado como parte do título — artifact do período de migração.  
**Ação:** Simplificar para `title: Diário 2026-06-21` e `title: Diário 2026-06-22`, e renomear os arquivos para `2026-06-21.md` e `2026-06-22.md` (ou com HHMM, conforme decisão em 4.1).

---

## 10. Roadmap de Execução

Prioridades organizadas por impacto e custo de execução.

### Crítico (impacta funcionamento de agentes)

| # | Ação | Arquivo | Custo |
|---|---|---|---|
| C1 | Corrigir link quebrado `pendencia-problema-ssh-claude.md` → `termux-ssh-claude.md` | `index.md` linha 19 | Baixo |
| C2 | Corrigir `type: reference` → `type: concept` | `wiki-review-vs-background-review.md` | Baixo |
| C3 | Atualizar tabela nudge_interval (10→2) | `wiki-review.md` | Baixo |
| C4 | Atualizar tabela de acesso de agentes (Hermes não usa MCP ai-memory) | `AGENTS.md` | Baixo |

### Alta prioridade (padronização e OKF)

| # | Ação | Arquivo | Custo |
|---|---|---|---|
| A1 | Remover frontmatter completo do index.md, manter só `okf_version: "0.1"` | `index.md` | Baixo |
| A2 | Criar `log.md` na raiz da wiki | novo arquivo | Médio |
| A3 | Padronizar idioma de tags (EN → PT-BR) | múltiplos arquivos | Médio |
| A4 | Criar `wiki/perfil/giovani.md` com perfil canônico | novo arquivo | Médio |
| A5 | Deletar `/root/hermes-user.backup.md` | arquivo externo | Baixo (requer aprovação) |
| A6 | Renumerar itens de `proximos-passos.md` | `proximos-passos.md` | Baixo |

### Média prioridade (conteúdo e promoção)

| # | Ação | Arquivo | Custo |
|---|---|---|---|
| M1 | Adicionar `resource:` em hermes-api.md, vps.md, telegram-topicos.md | 3 arquivos | Baixo |
| M2 | Mover entradas temáticas do diário para páginas permanentes | 2 arquivos diário | Médio |
| M3 | Criar seção "Promote" no AGENTS.md | `AGENTS.md` | Médio |
| M4 | Adicionar tweet "design loops" em agent-loop-architectures.md | `agent-loop-architectures.md` | Baixo |
| M5 | Criar seção "Configuração Hermes" em firecrawl.md | `firecrawl.md` | Baixo |
| M6 | Linkar historico/ e infraestrutura/ de hermes.md | `hermes.md` | Baixo |
| M7 | Documentar escrita concorrente como issue em wiki-review.md | `wiki-review.md` | Baixo |

### Baixa prioridade (refinamento e futuro)

| # | Ação | Arquivo | Custo |
|---|---|---|---|
| B1 | Definir idioma de wikilinks (com/sem prefixo `wiki/`) e documentar no AGENTS.md | AGENTS.md | Baixo |
| B2 | Decidir formato padrão de diário e renomear arquivos legacy | diário/ | Médio |
| B3 | ~~Criar `wiki/conhecimento/elevenlabs-sfx.md`~~ ✅ criada como `infraestrutura/elevenlabs-mcp.md` | novo arquivo | Médio |
| B4 | Adicionar seção "Tipos de busca" (factual vs temática) em firecrawl.md | `firecrawl.md` | Baixo |
| B5 | Avaliar se `wiki/pendencias/` vale como pasta (só 1 arquivo) | estrutura | Baixo |
| B6 | Atualizar `user_profile.md` do Claude Code | memória externa | Baixo |

---

## Notas de Manutenção

- **Este documento** deve ser revisado após cada ciclo de melhorias — marcar itens como ✅ e atualizar o `timestamp`.
- **AGENTS.md** define as regras — qualquer mudança de convenção deve ir para lá primeiro.
- **Lint periódico:** executar `git ls-files wiki/ | sort` vs a árvore do AGENTS.md para detectar novos desvios.
- **Promoção do diário:** revisitar semanalmente o diário mais recente e avaliar o que merece promoção.

---

## Conexões

- [[AGENTS.md]] — schema da wiki, regras de escrita e operações
- [[index.md]] — catálogo completo
- [[conhecimento/wiki.md]] — fundação e princípios
- [[conhecimento/okf.md]] — especificação OKF e conformidade
- [[automacao/wiki-review.md]] — plugin de captura automática
- [[pendencias/proximos-passos.md]] — to-do list ativa
