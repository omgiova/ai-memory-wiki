---
type: note
tags: [ai-memory, alexa, docker, mcp, n8n, note, vps]
title: Arquitetura do VPS
description: hermes-config/notes/INDEX.md📂 Voltar para notes
timestamp: 2026-06-18T00:00:00+00:00
---


# Arquitetura do VPS

## Hardware
- **VPS:** Hostinger KVM 2
- **SO:** Ubuntu 22.04 (Linux 6.8.0)
- **Disco:** 96GB (60GB livre)
- **RAM:** 7.8GB (4.8GB disponível)

## Serviços rodando
- **Hermes Agent** — porta 9119 (dashboard)
- **n8n** — automação de workflows (MCP)
- **Node-RED** — automação residencial + Alexa
- **ai-memory** — porta 49374 (memória agêntica)

## Stack de desenvolvimento
- **Runtime:** Node.js, Python 3.11
- **Container:** Docker
- **Banco:** SQLite (Hermes, ai-memory)

## 📂 Navegação
[[hermes-config/notes/INDEX.md|📂 Voltar para notes]]

## 🔗 Relacionados (mesmo grupo)
- [[hermes-config/notes/visao-geral.md|visao-geral]]

## 🔗 Conexões entre projetos
- [[hermes/decisions/memoria-wiki-markdown.md|memoria-wiki-markdown]]
- [[hermes/rules/conhecimento-na-wiki.md|conhecimento-na-wiki]]
- [[hermes/sessoes/2026-06-18-recuperacao-e-wiki.md|2026-06-18-recuperacao-e-wiki]]
- [[infra-vps/decisions/arquitetura-rede-swarm.md|arquitetura-rede-swarm]]
- [[infra-vps/gotchas/docker-swarm-ipvs-overlay.md|docker-swarm-ipvs-overlay]]
- [[infra-vps/gotchas/ipvs-table-vazia.md|ipvs-table-vazia]]

