---
type: concept
tags: [elevenlabs, mcp, infraestrutura, tts, sfx, limites]
title: ElevenLabs MCP — Capabilities e Limites
description: Capacidades e limitações do MCP ElevenLabs no free tier — o que funciona, o que não funciona e o que evitar
timestamp: 2026-06-27T18:29:00-03:00
status: stable
---

# ElevenLabs MCP — Capabilities e Limites

MCP instalado via `uvx elevenlabs-mcp`, habilitado no Hermes. API key em `.env` como `ELEVENLABS_API_KEY`.

## O que funciona no free tier

- **`text_to_sound_effects`** — usa créditos de SFX separados dos chars TTS; pode ser gerado livremente dentro do saldo
- **`text_to_speech`** — saldo de 10.000 chars/mês
- 26 ferramentas disponíveis no total

## O que não funciona no free tier

| Ferramenta | Erro | Motivo |
|---|---|---|
| `compose_music` | `402 paid_plan_required` | exclusivo de planos pagos |
| `can_use_instant_voice_cloning` | `false` | free tier não inclui clonagem instantânea |

## Limitação crítica: `text_to_sound_effects` e duração

O modelo **não foi projetado para melodias ou sons musicais contínuos**. O que acontece:

- Sons com mais de ~1–2s de conteúdo ativo ficam em silêncio no tempo restante
- `duration_seconds` define o tamanho do arquivo, não o tempo de som ativo
- `loop=true` é a ÚNICA forma confiável de preencher a duração completa
- Texto de duração no prompt ("a 5 second continuous X") não tem efeito — o modelo ignora

## Prompt language importa

Qualidade do output em `text_to_sound_effects` por idioma do prompt:

| Prompt | Qualidade |
|---|---|
| Português puro | menor |
| Inglês literal (tradução direta) | intermediária |
| Inglês descritivo | maior |

## Créditos e limites do free plan

| Item | Valor |
|---|---|
| TTS | 10.000 chars/mês |
| SFX | 50 gerações/mês (créditos separados do TTS) |
| Custo por geração SFX (AI decide duração) | 200 créditos |
| Custo por segundo (custom duration) | 40 créditos/segundo |
| Rate limit 429 `system_busy` | servidor congestionado — não é limite do plano |
| Reset mensal | ~dia 22 do mês |

## Conexões

- [[infraestrutura/hermes.md|Hermes]] — MCP listado como `uvx elevenlabs-mcp (enabled)`
- [[infraestrutura/hermes-api.md|Hermes API]] — endpoint `GET /api/audio/elevenlabs/voices`
