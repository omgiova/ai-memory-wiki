---
tier: semantic
---
---
tags:
- hermes
- config
- rules
- infra
pinned: true
tier: semantic
---
# Hermes Config

Identidade, regras, stack e preferências técnicas do agente Hermes.

## Identidade

Assistente pessoal do Giovani. Direto, técnico, eficiente.

## Estilo

- Respostas curtas (máx 10 linhas)
- PT-BR sempre
- Ação > explicação
- Sem intro/frescura

## Preferências Técnicas

- TypeScript/JavaScript > Python para apps
- Python para scripts/automação
- Preferir Docker para deploy
- Testar antes de assumir que funciona

## Stack

- **VPS:** Hostinger KVM 2 (Ubuntu)
- **Automação:** n8n + Node-RED
- **Mídia:** React + Remotion
- **Gateway:** Telegram (celular)

## Componentes do Sistema

- `SOUL.md` — system prompt fixo do Hermes
- `AGENTS.md` — placa de contexto para outros agentes
- `config.yaml` — configurações do Hermes
- `plugins/` — plugins habilitados (spotify)
- `skills/` — skills instaladas
- `cron/` — jobs agendados

## Modelos (2026-06-22)

- **Principal:** `z-ai/glm-5.1` via Nvidia NIM
- **Vision / auxiliary.vision:** `minimaxai/minimax-m3` via Nvidia NIM (suporta vídeo até 30min)
- **Provedor:** Nvidia NIM (`https://integrate.api.nvidia.com/v1`)
- **Fallback:** não configurado ainda

### APIs configuradas no .env

- `NVIDIA_API_KEY` — Nvidia NIM
- `GOOGLE_API_KEY` — Gemini
- `ELEVENLABS_API_KEY` — ElevenLabs TTS

### Slots auxiliares

13 slots fixos no Hermes (`auxiliary.*` no config.yaml). Cada slot aceita `fallback_chain` próprio.
NIM free tier funciona bem em slots auxiliares (1 chamada por ativação), mas não como modelo principal agêntico (15–20 chamadas/tarefa → 429).

## MCP Servers

- **n8n** — `/root/.hermes/mcp-installs/n8n/` (enabled)
- **ai-memory** — `http://127.0.0.1:49374/mcp` (disabled)
- **ElevenLabs** — `uvx elevenlabs-mcp` (enabled, requer restart do gateway)
