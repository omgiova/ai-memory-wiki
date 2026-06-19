---
type: procedure
tags: [ai-memory, hermes, obsidian, procedure, projetos]
title: Procedimento: Implementar Memória Baseada em Arquivos (Obsidian + Markdown)
description: 1. Organizar o vault Obsidian 00 - Inbox/ → ideias soltas 01 - Projetos/ → um diretório por projeto 02 - Conhecimento/ → regras, conceitos 03 - Logs/ → registro de sessões
timestamp: 2026-06-17T00:00:00+00:00
---


## Procedimento: Implementar Memória Baseada em Arquivos (Obsidian + Markdown)

**Data:** 2026-06-17  
**Fonte:** Pesquisa do usuário sobre alternativas de memória agêntica

### Passo a passo didático

1. **Organizar o vault Obsidian**
   ```
   00 - Inbox/          → ideias soltas
   01 - Projetos/        → um diretório por projeto
   02 - Conhecimento/    → regras, conceitos
   03 - Logs/            → registro de sessões
   ```

2. **Criar arquivo de contexto por projeto**
   Em `01 - Projetos/PROJETO_X.md`:
   ```markdown
   # Projeto X
   ## Objetivo Principal
   ## Tecnologias
   ## Decisões Chave
   ## Próximos Passos
   ```

3. **Criar arquivo de regras do agente**
   Em `02 - Conhecimento/Regras_do_Agente.md`:
   ```markdown
   ## Identidade
   ## Estilo de Comunicação
   ## Preferências Técnicas
   ```

4. **Implementar log de sessão**
   Ao final de cada interação:
   ```bash
   ai-memory hook finish
   ```
   Ou manualmente: salvar resumo em `03 - Logs/YYYY-MM-DD_Sessao.md`

5. **Instruir o agente a carregar contexto**
   Iniciar sessão com:
   > "Leia o arquivo do projeto e as regras do agente antes de responder"

## 📂 Navegação
[[hermes/procedures/INDEX.md|📂 Voltar para procedures]]

## 🔗 Conexões entre projetos
- [[geral/sessoes/2026-06-18-crise-update-recuperacao.md|2026-06-18-crise-update-recuperacao]]
- [[hermes/decisions/memoria-wiki-markdown.md|memoria-wiki-markdown]]
- [[hermes/rules/conhecimento-na-wiki.md|conhecimento-na-wiki]]
- [[hermes/sessoes/2026-06-18-recuperacao-e-wiki.md|2026-06-18-recuperacao-e-wiki]]
- [[hermes/todo/proximos-passos.md|proximos-passos]]
- [[infra-vps/procedures/diagnostico-502-traefik.md|diagnostico-502-traefik]]

