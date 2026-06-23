---
tier: semantic
---
---
type: session
tags: [hermes, nvidia-nim, elevenlabs, modelos, mcp, tts]
title: "Sessão 2026-06-22 — Modelos NIM + ElevenLabs MCP"
description: Configuração de modelos via Nvidia NIM, diagnóstico de rate limit agêntico, e instalação do MCP ElevenLabs.
timestamp: 2026-06-22T00:00:00+00:00
---

## Sessão: 2026-06-22 — Modelos NIM + ElevenLabs MCP

### O que foi configurado

- Provedor principal migrado para Nvidia NIM (`base_url: https://integrate.api.nvidia.com/v1`)
- Modelo principal: `z-ai/glm-5.1`
- `auxiliary.vision` + `vision`: `minimaxai/minimax-m3` (multimodal: texto, imagem, vídeo até 30min)
- `NVIDIA_API_KEY` adicionado ao `.env`
- `GOOGLE_API_KEY` (Gemini) adicionado ao `.env`
- `ELEVENLABS_API_KEY` adicionado ao `.env`
- MCP ElevenLabs instalado via `uvx elevenlabs-mcp` (`mcp_servers.elevenlabs` no `config.yaml`)

### Diagnóstico: NIM + Hermes agêntico não combinam

- Hermes gera 15–20 chamadas por tarefa → NIM free tier estoura em 429
- Causa: loop agêntico (cada tool call = 1 request LLM)
- Solução: usar NIM nos slots auxiliares (1 chamada cada) ou via n8n para chamadas únicas
- 13 slots auxiliares disponíveis no Hermes, cada um com `fallback_chain` próprio e independente

### ElevenLabs

- 21 vozes gratuitas premade (inglês, falam PT-BR via `eleven_multilingual_v2` com sotaque estrangeiro)
- 3 vozes BR nativas (Lucke, Matheus, Gustavo) — exigem plano pago (Starter $6/mês)
- Capacidades free: TTS, STT, sound effects, music, voice design
- TTS ativo no Hermes: ainda `edge` (Microsoft) — ElevenLabs disponível mas não ativado como padrão
- Gateway precisa reiniciar para carregar o MCP ElevenLabs
