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
│   │   ├── google-okf-README.md
│   │   ├── introducing-the-open-knowledge-format.md
│   │   └── SPEC.md
│   ├── agents-cli-README.md
│   ├── akshay-pachaar-karpathy-agentic-engineering-tooling.md
│   ├── karpathy-llm-wiki-pattern.md
│   └── libertas.md
└── wiki/
    ├── concepts/
    │   ├── okf.md
    │   ├── orquestrador.md
    │   ├── plano-implementacao-loop.md
    │   └── wiki.md
    ├── diario/
    │   ├── 2026-06-28-2012-20260628_195.md
    │   └── 2026-06-28-2155-20260628_212.md
    ├── diary/
    ├── history/
    │   ├── 2026-06-22-modelos-nim-elevenlabs.md
    │   ├── 2026-06-24-20260624.md
    │   └── crise-update.md
    ├── procedures/
    │   ├── auditor-wiki.md
    │   ├── curador-wiki-historico.md
    │   ├── curador-wiki.md
    │   └── wiki-review.md
    ├── systems/
    │   ├── hermes-endpoints.md
    │   ├── hermes.md
    │   ├── termux-ssh-claude.md
    │   └── vps.md
    ├── todo/
    │   └── proximos-passos.md
    └── tools/
        ├── elevenlabs-mcp.md
        ├── firecrawl.md
        ├── obsidian-git.md
        └── telegram.md
```

## raiz/

- [[AGENTS.md|AGENTS.md]] — instruções obrigatórias para agentes que trabalham neste repositório; lido automaticamente por Claude Code e Codex CLI
- [[index.md|index.md]] — este arquivo; catálogo completo e ponto de entrada único da wiki
- [[log.md|log.md]] — changelog estrutural da wiki; registro cronológico de criações, atualizações, renomeações e deleções de páginas

## wiki/

### systems/

- [[wiki/systems/hermes.md|Hermes]] — sistema principal: identidade, stack, modelos, interface REST e operação do Hermes Agent
- [[wiki/systems/hermes-endpoints.md|Hermes API]] — referência completa dos ~180 endpoints REST do Hermes Agent (gerada do /openapi.json)
- [[wiki/systems/vps.md|VPS]] — Hostinger KVM 2 — hardware, serviços rodando, Docker Swarm, IPVS, problemas conhecidos
- [[wiki/systems/termux-ssh-claude.md|Termux + SSH + Claude]] — setup completo de acesso remoto: configuração, erros conhecidos e troubleshooting

### tools/

- [[wiki/tools/telegram.md|Telegram]] — referência completa: IDs, tópicos, HERMES_SESSION_MESSAGE_ID, sendMessage, sendRichMessage, reações e integrações com o Hermes
- [[wiki/tools/elevenlabs-mcp.md|ElevenLabs MCP]] — capabilities e limites do free tier: text_to_sound_effects, compose_music (pago), limitação de duração, créditos
- [[wiki/tools/obsidian-git.md|Obsidian Git]] — todos os problemas já encontrados com o plugin obsidian-git, causas raiz e soluções definitivas
- [[wiki/tools/firecrawl.md|Firecrawl]] — busca com sintaxe site: para plataformas específicas; quando usar e não usar

### procedures/

- [[wiki/procedures/auditor-wiki.md|Auditor da Wiki]] — auditoria completa multi-agente em 3 fases: estrutural, semântica e coordenação; entrega página de findings priorizada
- [[wiki/procedures/curador-wiki.md|Curador da Wiki]] — papel, comportamento, arquitetura atual (v6/v5), formato de output e como executar
- [[wiki/procedures/curador-wiki-historico.md|Curador da Wiki — Histórico]] — registro completo de todas as tentativas, scripts e decisões de design desde a v1
- [[wiki/procedures/wiki-review.md|Wiki Review]] — agente background que roda a cada 10 turnos e salva insights no diary/

### concepts/

- [[wiki/concepts/wiki.md|Wiki]] — o que é esta wiki, como funciona, regras de escrita e histórico de fundação
- [[wiki/concepts/okf.md|Open Knowledge Format (OKF)]] — padrão open source do Google Cloud para wikis de agentes; nossa wiki já é conformante; repo oficial para consulta futura
- [[wiki/concepts/plano-implementacao-loop.md|Plano de Implementação — Loops Agênticos]] — base de conhecimento e planejamento para construção de sistema de loop autônomo; fundamentado em fontes primárias (Steinberger, Boris Cherny, Addy Osmani, OpenClaw)
- [[wiki/concepts/orquestrador.md|Orquestrador da Memória]] — auditoria completa da wiki: OKF, estrutura, diário, automação, coordenação de agentes e roadmap de melhorias

### history/

- [[wiki/history/2026-06-24-20260624.md|Diário 2026-06-24]] — daily longa (64KB); sessão com ElevenLabs SFX, agent loops, OpenClaw, escrita concorrente entre instâncias
- [[wiki/history/crise-update.md|Crise update]] — múltiplos /update corromperam state.db; backup salvou; fixes aplicados
- [[wiki/history/2026-06-22-modelos-nim-elevenlabs.md|Modelos NIM + ElevenLabs]] — migração para Nvidia NIM, rate limit agêntico, MCP ElevenLabs, tentativa Groq
- [[wiki/history/2026-06-28-libertas-carrossel-7sweeps.md|Revisão Libertas — 7 Sweeps]] — aplicação dos 7 Sweeps do conversion-copywriting no carrossel Libertas v2/v3, 10 intervenções documentadas contra o guia de voz

### todo/

- [[wiki/todo/proximos-passos.md|Próximos passos]] — to-do list ativa da wiki

### diario/

Inbox da wiki — captura bruta automática por sessão (wiki-review). Contém preferências reveladas, correções de comportamento, técnicas descobertas e pendências abertas. Não é destino final — insights estáveis devem ser extraídos e integrados em páginas permanentes.

- [[wiki/diario/2026-06-28-2012-20260628_195.md|Diário 2026-06-28 (20:12)]] — estrutura wiki, auditor v1 desastre (75% cota), decisões de taxonomia
- [[wiki/diario/2026-06-28-2155-20260628_212.md|Diário 2026-06-28 (21:55)]] — carrossel: regras de naming, fluxo narrativo, CTA personalizado, tela de fechamento

## raw/

### google-okf/

- [[raw/google-okf/google-okf-README.md|OKF README.md]] — intro e agente de referência do repo okf/
- [[raw/google-okf/SPEC.md|OKF SPEC.md]] — especificação oficial OKF v0.1 do Google Cloud
- [[raw/google-okf/introducing-the-open-knowledge-format.md|Introducing the Open Knowledge Format]] — blog post do Google Cloud (jun/2026)

### raiz
- [[raw/karpathy-llm-wiki-pattern.md|LLM Wiki Pattern (Karpathy)]] — documento original do padrão que esta wiki segue; fonte imutável
- [[raw/libertas.md|Guia de Voz — Libertas Assessoria Financeira]] — documento de referência para copywriter com tom de voz, exemplos aprovados e estrutura de conteúdo
- [[raw/akshay-pachaar-karpathy-agentic-engineering-tooling.md|Karpathy's Agentic Engineering Finally Has Proper Tooling]] — artigo de Akshay Pachaar sobre Google Agents CLI (jun/2026)
- [[raw/agents-cli-README.md|Google Agents CLI — README oficial]] — README do repositório google/agents-cli
