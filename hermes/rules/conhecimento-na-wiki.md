---
type: rule
tags: [ai-memory, docker, hermes, projetos, rule, wiki]
title: Regra: Conhecimento Durável Vai pra Wiki, Não pra Memory()
description: 1. Escrever markdown no filesystem: /root/ai-memory-wiki/wiki/<projeto>/<tipo>/<nome>.md 2. Indexar via: docker exec ai-memory ai-memory write-page --path <path> --body "<markdown>" 3. Memory do He...
timestamp: 2026-06-18T00:00:00+00:00
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

## 📂 Navegação
[[hermes/rules/INDEX.md|📂 Voltar para rules]]

## 🔗 Conexões entre projetos
- [[geral/sessoes/2026-06-18-crise-update-recuperacao.md|2026-06-18-crise-update-recuperacao]]
- [[hermes/decisions/memoria-wiki-markdown.md|memoria-wiki-markdown]]
- [[hermes/procedures/implementar-memoria-arquivos.md|implementar-memoria-arquivos]]
- [[hermes/sessoes/2026-06-18-recuperacao-e-wiki.md|2026-06-18-recuperacao-e-wiki]]
- [[hermes/todo/proximos-passos.md|proximos-passos]]
- [[hermes-config/notes/arquitetura-vps.md|arquitetura-vps]]

