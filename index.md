---
type: index
title: wiki
description: Catálogo completo da wiki — uma linha por página, atualizado a cada ingest
timestamp: 2026-06-26T00:00:00-03:00
status: stable
---

# wiki

Catálogo completo. Leia este arquivo primeiro ao responder queries — use os links para navegar às páginas relevantes.

## Estrutura

```
.gitignore
.obsidian/
AGENTS.md
index.md
├── raw/
│   ├── google-okf/
│   │   ├── introducing-the-open-knowledge-format.md
│   │   ├── README.md
│   │   └── SPEC.md
│   └── karpathy-llm-wiki-pattern.md
└── wiki/
    ├── automacao/
    │   ├── firecrawl.md
    │   ├── wiki-review-vs-background-review.md
    │   └── wiki-review.md
    ├── conhecimento/
    │   ├── agent-loop-architectures.md
    │   ├── okf.md
    │   ├── orquestrador.md
    │   └── wiki.md
    ├── diario/
    │   ├── 2026-06-19.md
    │   ├── 2026-06-20.md
    │   ├── 2026-06-22-20260621.md
    │   ├── 2026-06-22-20260622.md
    │   ├── 2026-06-22-descoberta-id-mensagem.md
    │   ├── 2026-06-22-reacoes-telegram.md
    │   ├── 2026-06-23-20260623.md
    │   ├── 2026-06-24-1710.md
    │   ├── 2026-06-24-20260623.md
    │   ├── 2026-06-24-20260624.md
    │   └── 2026-06-24-obsidian-git-setup.md
    ├── historico/
    │   ├── 2026-06-22-modelos-nim-elevenlabs.md
    │   └── crise-update.md
    ├── infraestrutura/
    │   ├── hermes-api.md
    │   ├── hermes.md
    │   ├── obsidian-git.md
    │   ├── telegram-topicos.md
    │   ├── termux-ssh-claude.md
    │   └── vps.md
    └── pendencias/
        └── proximos-passos.md
```

## raiz/

- [[AGENTS.md|AGENTS.md]] — instruções obrigatórias para agentes que trabalham neste repositório; lido automaticamente por Claude Code e Codex CLI
- [[index.md|index.md]] — este arquivo; catálogo completo e ponto de entrada único da wiki

## wiki/

### infraestrutura/

- [[wiki/infraestrutura/vps.md|VPS]] — Hostinger KVM 2 — hardware, serviços rodando, Docker Swarm, IPVS, problemas conhecidos
- [[wiki/infraestrutura/hermes.md|Hermes]] — identidade, regras, stack, modelos ativos e preferências do Hermes Agent
- [[wiki/infraestrutura/telegram-topicos.md|Telegram Tópicos]] — mapa completo de chats, tópicos e IDs do Telegram
- [[wiki/infraestrutura/hermes-api.md|Hermes API]] — referência completa dos ~180 endpoints REST do Hermes Agent (gerada do /openapi.json)
- [[wiki/infraestrutura/termux-ssh-claude.md|Problema SSH/Claude]] — diagnóstico dos 3 problemas que travam sessões Claude via Remote Control (curl sem timeout, processo zumbi, prompt invisível)
- [[wiki/infraestrutura/obsidian-git.md|Obsidian Git]] — todos os problemas já encontrados com o plugin obsidian-git, causas raiz e soluções definitivas

### conhecimento/

- [[wiki/conhecimento/wiki.md|Wiki]] — o que é esta wiki, como funciona, regras de escrita e histórico de fundação
- [[wiki/conhecimento/okf.md|Open Knowledge Format (OKF)]] — padrão open source do Google Cloud para wikis de agentes; nossa wiki já é conformante; repo oficial para consulta futura
- [[wiki/conhecimento/agent-loop-architectures.md|Arquiteturas de Loop em Agentes]] — comparação de detecção de loop, aprovação, circuit breaker entre Hermes, OpenClaw, Claude Code, Codex e Cline
- [[wiki/conhecimento/orquestrador.md|Orquestrador da Memória]] — auditoria completa do vault: OKF, estrutura, diário, automação, coordenação de agentes e roadmap de melhorias

### automacao/

- [[wiki/automacao/firecrawl.md|Firecrawl]] — busca com sintaxe site: para plataformas específicas; quando usar e não usar
- [[wiki/automacao/wiki-review.md|Wiki Review]] — agente background que roda a cada 10 turnos e salva insights no diario/

### historico/

- [[wiki/historico/crise-update.md|Crise update]] — múltiplos /update corromperam state.db; backup salvou; fixes aplicados
- [[wiki/historico/2026-06-22-modelos-nim-elevenlabs.md|Modelos NIM + ElevenLabs]] — migração para Nvidia NIM, rate limit agêntico, MCP ElevenLabs, tentativa Groq

### pendencias/

- [[wiki/pendencias/proximos-passos.md|Próximos passos]] — to-do list ativa da wiki

### diario/

Inbox da wiki — captura bruta automática por sessão (wiki-review). Contém preferências reveladas, correções de comportamento, técnicas descobertas e pendências abertas. Não é destino final — insights estáveis devem ser extraídos e integrados em páginas permanentes.

- [[wiki/diario/2026-06-24-20260624.md|2026-06-24]] — .hermes.md unificado com AGENTS.md, correção sobre follow-ups e memory() sem permissão
- [[wiki/diario/2026-06-23-20260623.md|2026-06-23]] — dashboard basic auth, preferência por ação direta, skill hermes-maintenance
- [[wiki/diario/2026-06-22-20260622.md|2026-06-22]] — reações Telegram, protocolo test-then-act, Crawlee/Apify
- [[wiki/diario/2026-06-22-20260621.md|2026-06-21]] — sessão anterior (ver arquivo para detalhes)
- [[wiki/diario/2026-06-22-reacoes-telegram.md|2026-06-22 reações]] — testes de reação no Telegram, descoberta de message-id
- [[wiki/diario/2026-06-22-descoberta-id-mensagem.md|2026-06-22 message-id]] — técnica de descoberta sequencial de IDs
- [[wiki/diario/2026-06-20.md|2026-06-20]] — fix do /update destruindo dados; migração para wiki Karpathy
- [[wiki/diario/2026-06-19.md|2026-06-19]] — otimizações estruturais da wiki

## raw/

### google-okf/

- [[raw/google-okf/SPEC.md|OKF SPEC.md]] — especificação oficial OKF v0.1 do Google Cloud
- [[raw/google-okf/README.md|OKF README.md]] — intro e agente de referência do repo okf/
- [[raw/google-okf/introducing-the-open-knowledge-format.md|Introducing the Open Knowledge Format]] — blog post do Google Cloud (jun/2026)

### raiz

- [[raw/karpathy-llm-wiki-pattern.md|LLM Wiki Pattern (Karpathy)]] — documento original do padrão que esta wiki segue; fonte imutável
