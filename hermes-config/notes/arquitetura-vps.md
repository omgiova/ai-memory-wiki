---
pinned: true
tier: semantic
tags: [ai-memory, alexa, docker, mcp, n8n, note, vps]
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