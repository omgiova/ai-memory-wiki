---
type: concept
tags: [docker, n8n, vps]
title: Infraestrutura do VPS
description: Hostinger KVM 2 — hardware, serviços rodando, stack e problemas conhecidos
timestamp: 2026-06-18T00:00:00+00:00
---

# Infraestrutura do VPS

## Hardware

- **VPS:** Hostinger KVM 2
- **SO:** Ubuntu 22.04 (Linux 6.8.0)
- **Disco:** 96GB (60GB livre)
- **RAM:** 7.8GB (4.8GB disponível)

## Serviços rodando

| Serviço | Porta | Função |
|---|---|---|
| Hermes Agent | 9119 | Dashboard |
| n8n | — | Automação (MCP) |
| Node-RED | 8800 | Automação residencial + Alexa |
| ai-memory | 49374 | Memória agêntica |

## Stack de desenvolvimento

- **Runtime:** Node.js, Python 3.11
- **Container:** Docker
- **Banco:** SQLite (Hermes, ai-memory)

## Problemas conhecidos

- [[infra-vps/gotchas/ipvs-table-502.md|IPVS Table Vazia + Traefik 502]]
