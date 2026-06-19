---
tier: semantic
---
## Regra: Conhecimento Durável Vai pra Wiki, Não pra Memory()

**Data:** 2026-06-18

### O que vai pra wiki (markdown)
- Decisões de arquitetura
- Gotchas/armadilhas
- Procedimentos (passo a passo)
- Regras (nunca fazer X)
- Preferências do usuário
- Configuração do ambiente

### O que fica na memory() do Hermes (atalho técnico)
- TL;DR de regras importantes (pra eu não repetir erro)
- IDs de chat, portas, caminhos fixos
- Preferências de comunicação (tom, estilo, idioma)
- Flag de ferramentas que não funcionam (web_extract, etc.)

### Como salvar
1. Escrever markdown no filesystem: `/root/ai-memory-wiki/wiki/<projeto>/<tipo>/<nome>.md`
2. Indexar via: `docker exec ai-memory ai-memory write-page --path <path> --body "<markdown>"`
3. Memory() do Hermes = só o essencial pra não errar na próxima sessão
