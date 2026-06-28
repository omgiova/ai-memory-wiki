---
type: concept
tags: [hermes, config, identity, rules, modelos, mcp]
title: Configuração do Hermes
description: Identidade, regras, stack, modelos ativos e preferências do Hermes Agent — assistente pessoal do Giovani
timestamp: 2026-06-23T21:00:00-03:00
status: stable
---

# Configuração do Hermes

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

## Regras de Comportamento

- **Sempre carregar a skill relevante** (`skill_view(nome)`) antes de qualquer ação
- **Se a skill descreve o comando, usar exatamente como está** — não adaptar, não testar alternativas
- **Só investigar se o comando da skill falhar** — partindo da skill como fonte da verdade
- **Nunca modificar uma skill sem autorização explícita** do usuário

> Origem: sessão 2026-06-18 — agente ignorou skill `alexa-notifications` e desperdiçou dezenas de comandos desnecessários.

## Stack

- **VPS:** Hostinger KVM 2 (Ubuntu)
- **Automação:** n8n + Node-RED
- **Mídia:** React + Remotion
- **Gateway:** Telegram (celular)

Detalhes de infra no [[wiki/infraestrutura/vps.md|vps]].

## Componentes do Sistema

- `SOUL.md` — system prompt fixo do Hermes
- `config.yaml` — configurações do Hermes
- `plugins/` — plugins habilitados (spotify)
- `skills/` — skills instaladas
- `cron/` — jobs agendados

## Skills — Bundled vs User

| Tipo | Como identificar | Pode editar? |
|---|---|---|
| Bundled | `git -C /root/.hermes ls-files skills/<path>` retorna o path | **NÃO** — causa conflito no `/update` |
| User | `git -C /root/.hermes ls-files skills/<path>` retorna vazio | Sim |

**Regra obrigatória:** sempre verificar com `git ls-files` antes de editar qualquer arquivo em `skills/`. Editar uma skill bundled causa `M` no git status → stash conflict no `/update` → `git reset --hard HEAD` apagando dados do usuário.

**Proteção permanente:** `.git/info/exclude` em `/root/.hermes/.git/info/exclude` exclui arquivos críticos de runtime do stash. É local — nunca commitado, nunca sobrescrito por `hermes update`.

## Modelos (2026-06-22)

- **Principal:** `moonshotai/kimi-k2.6` via Nvidia NIM
- **Vision / auxiliary.vision:** `minimaxai/minimax-m3` via Nvidia NIM (suporta vídeo até 30min)
- **Provedor:** Nvidia NIM (`https://integrate.api.nvidia.com/v1`)
- **Fallback:** não configurado

### Web Search

- **Backend:** Firecrawl (`search_backend: firecrawl`, `extract_backend: firecrawl`)
- API key e URL configurados no `.env` (`FIRECRAWL_API_KEY`, `FIRECRAWL_API_URL`)
- Usar `web_search` para buscas rápidas, `firecrawl search` via terminal para buscas avançadas

### APIs configuradas no .env

- `NVIDIA_API_KEY` — Nvidia NIM (modelos principais e auxiliares)
- `GOOGLE_API_KEY` — Gemini
- `ELEVENLABS_API_KEY` — ElevenLabs TTS/STT
- `GROQ_API_KEY` — Groq (STT whisper; LLM pendente fix de compatibilidade)

### Slots auxiliares

13 slots fixos no Hermes (`auxiliary.*` no config.yaml). Cada slot aceita `fallback_chain` próprio e independente.
NIM free tier funciona bem em slots auxiliares (1 chamada por ativação), mas não como modelo principal agêntico (15–20 chamadas/tarefa → 429).

## MCP Servers

- **n8n** — `/root/.hermes/mcp-installs/n8n/` (enabled)
- **ElevenLabs** — `uvx elevenlabs-mcp` (enabled) — ver [[infraestrutura/elevenlabs-mcp.md]]
- **ai-memory** — `http://127.0.0.1:49374/mcp` (disabled — não usar)

## Conexões

- [[wiki/infraestrutura/vps.md|vps]] — hardware e serviços da stack
- [[wiki/conhecimento/wiki.md|wiki]] — base de conhecimento
- [[wiki/automacao/firecrawl.md|Firecrawl]] — busca multi-plataforma
- [[wiki/historico/crise-update.md|Crise update]] — recuperação de sessões
- [[wiki/pendencias/proximos-passos.md|Próximos passos]]
