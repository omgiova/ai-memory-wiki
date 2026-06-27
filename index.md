---
okf_version: "0.1"
---

# wiki

Catálogo completo. Leia este arquivo primeiro ao responder queries — use os links para navegar às páginas relevantes.

## Estrutura

> Esta seção é a fonte de verdade da estrutura da wiki — deve estar sempre sincronizada com `git ls-files`. Ao criar, renomear ou remover qualquer arquivo ou pasta, atualize aqui.

```
.gitignore
.obsidian/
│   ├── community-plugins.json
│   ├── core-plugins.json
│   └── plugins/obsidian-git/
│       ├── main.js
│       ├── manifest.json
│       └── styles.css
AGENTS.md
index.md
log.md
├── raw/
│   ├── google-okf/
│   │   ├── introducing-the-open-knowledge-format.md
│   │   ├── README.md
│   │   └── SPEC.md
│   └── karpathy-llm-wiki-pattern.md
└── wiki/
    ├── automacao/
    │   ├── curador-wiki.md
    │   ├── firecrawl.md
    │   ├── wiki-review-vs-background-review.md
    │   └── wiki-review.md
    ├── conhecimento/
    │   ├── plano-implementacao-loop.md
    │   ├── okf.md
    │   ├── orquestrador.md
    │   └── wiki.md
    ├── diario/
    │   ├── 2026-06-19.md
    │   ├── 2026-06-20.md
    │   ├── 2026-06-22-20260621.md
    │   ├── 2026-06-22-20260622.md
    │   ├── 2026-06-22-reacoes-telegram.md
    │   ├── 2026-06-23-20260623.md
    │   ├── 2026-06-24-1710.md
    │   ├── 2026-06-24-20260623.md
    │   ├── 2026-06-24-20260624.md
    ├── historico/
    │   ├── 2026-06-22-modelos-nim-elevenlabs.md
    │   └── crise-update.md
    ├── infraestrutura/
    │   ├── hermes-api.md
    │   ├── hermes.md
    │   ├── obsidian-git.md
    │   ├── telegram-bot-api.md
    │   ├── telegram-reacoes.md
    │   ├── telegram-send-rich-message.md
    │   ├── telegram-topicos.md
    │   ├── termux-ssh-claude.md
    │   └── vps.md
    └── pendencias/
        └── proximos-passos.md
```

## raiz/

- [[AGENTS.md|AGENTS.md]] — instruções obrigatórias para agentes que trabalham neste repositório; lido automaticamente por Claude Code e Codex CLI
- [[index.md|index.md]] — este arquivo; catálogo completo e ponto de entrada único da wiki
- [[log.md|log.md]] — changelog estrutural da wiki; registro cronológico de criações, atualizações, renomeações e deleções de páginas

## wiki/

### infraestrutura/

- [[wiki/infraestrutura/vps.md|VPS]] — Hostinger KVM 2 — hardware, serviços rodando, Docker Swarm, IPVS, problemas conhecidos
- [[wiki/infraestrutura/hermes.md|Hermes]] — identidade, regras, stack, modelos ativos e preferências do Hermes Agent
- [[wiki/infraestrutura/telegram-topicos.md|Telegram Tópicos]] — mapa completo de chats, tópicos e IDs do Telegram
- [[wiki/infraestrutura/hermes-api.md|Hermes API]] — referência completa dos ~180 endpoints REST do Hermes Agent (gerada do /openapi.json)
- [[wiki/infraestrutura/termux-ssh-claude.md|Problema SSH/Claude]] — diagnóstico dos 3 problemas que travam sessões Claude via Remote Control (curl sem timeout, processo zumbi, prompt invisível)
- [[wiki/infraestrutura/obsidian-git.md|Obsidian Git]] — todos os problemas já encontrados com o plugin obsidian-git, causas raiz e soluções definitivas
- [[wiki/infraestrutura/telegram-bot-api.md|Telegram Hub]] — ponto de entrada único para tudo sobre Telegram: Bot API, tópicos, reações, HERMES_SESSION_MESSAGE_ID, integrações
- [[wiki/infraestrutura/telegram-send-rich-message.md|Telegram sendRichMessage]] — documentação do endpoint Bot API 10.1: parâmetros, payload, limites e exemplo validado
- [[wiki/infraestrutura/telegram-reacoes.md|Telegram Reações]] — procedimento para setMessageReaction via HERMES_SESSION_MESSAGE_ID; status: não validado de ponta a ponta até 2026-06-27

### conhecimento/

- [[wiki/conhecimento/wiki.md|Wiki]] — o que é esta wiki, como funciona, regras de escrita e histórico de fundação
- [[wiki/conhecimento/okf.md|Open Knowledge Format (OKF)]] — padrão open source do Google Cloud para wikis de agentes; nossa wiki já é conformante; repo oficial para consulta futura
- [[wiki/conhecimento/plano-implementacao-loop.md|Plano de Implementação — Loops Agênticos]] — base de conhecimento e planejamento para construção de sistema de loop autônomo; fundamentado em fontes primárias (Steinberger, Boris Cherny, Addy Osmani, OpenClaw)
- [[wiki/conhecimento/orquestrador.md|Orquestrador da Memória]] — auditoria completa da wiki: OKF, estrutura, diário, automação, coordenação de agentes e roadmap de melhorias

### automacao/

- [[wiki/automacao/firecrawl.md|Firecrawl]] — busca com sintaxe site: para plataformas específicas; quando usar e não usar
- [[wiki/automacao/wiki-review.md|Wiki Review]] — agente background que roda a cada 10 turnos e salva insights no diario/
- [[wiki/automacao/curador-wiki.md|Curador da Wiki]] — agente read-only MVP que sorteia uma daily, roda curadoria com claude -p e envia ao Telegram Geral

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
- [[wiki/diario/2026-06-20.md|2026-06-20]] — fix do /update destruindo dados; migração para wiki Karpathy
- [[wiki/diario/2026-06-19.md|2026-06-19]] — otimizações estruturais da wiki

## raw/

### google-okf/

- [[raw/google-okf/SPEC.md|OKF SPEC.md]] — especificação oficial OKF v0.1 do Google Cloud
- [[raw/google-okf/README.md|OKF README.md]] — intro e agente de referência do repo okf/
- [[raw/google-okf/introducing-the-open-knowledge-format.md|Introducing the Open Knowledge Format]] — blog post do Google Cloud (jun/2026)

### raiz

- [[raw/karpathy-llm-wiki-pattern.md|LLM Wiki Pattern (Karpathy)]] — documento original do padrão que esta wiki segue; fonte imutável
