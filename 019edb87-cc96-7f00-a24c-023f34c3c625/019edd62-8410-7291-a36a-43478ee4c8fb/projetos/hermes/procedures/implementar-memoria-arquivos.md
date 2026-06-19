---
tier: semantic
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
