---
type: index
title: wiki
description: Catálogo completo da wiki — uma linha por página, atualizado a cada ingest
timestamp: 2026-06-23T00:00:00+00:00
status: stable
---

# wiki

Catálogo completo. Leia este arquivo primeiro ao responder queries — use os links para navegar às páginas relevantes.

## 🖥 Infraestrutura

- [[infraestrutura/vps.md|VPS]] — Hostinger KVM 2 — hardware, serviços rodando, Docker Swarm, IPVS, problemas conhecidos
- [[infraestrutura/hermes.md|Hermes]] — identidade, regras, stack, modelos ativos e preferências do Hermes Agent

## 🧠 Conhecimento

- [[conhecimento/wiki.md|Wiki]] — o que é esta wiki, como funciona, regras de escrita e histórico de fundação

## 🔧 Automação

- [[automacao/firecrawl.md|Firecrawl]] — busca com sintaxe site: para plataformas específicas; quando usar e não usar
- [[automacao/wiki-review.md|Wiki Review]] — agente background que roda a cada 10 turnos e salva insights no diario/

## 📜 Histórico

- [[historico/crise-update.md|Crise update]] — múltiplos /update corromperam state.db; backup salvou; fixes aplicados
- [[historico/2026-06-22-modelos-nim-elevenlabs.md|Modelos NIM + ElevenLabs]] — migração para Nvidia NIM, rate limit agêntico, MCP ElevenLabs, tentativa Groq

## 📋 Pendências

- [[pendencias/proximos-passos.md|Próximos passos]] — to-do list ativa da wiki

## 📦 Raw

- [[raw/karpathy-llm-wiki-pattern.md|LLM Wiki Pattern (Karpathy)]] — documento original do padrão que esta wiki segue; fonte imutável

## 📓 Diário

Inbox da wiki — captura bruta automática por sessão (wiki-review). Contém preferências reveladas, correções de comportamento, técnicas descobertas e pendências abertas. Não é destino final — insights estáveis devem ser extraídos e integrados em páginas permanentes.

- [[diario/2026-06-23-20260623.md|2026-06-23]] — dashboard basic auth, preferência por ação direta, skill hermes-maintenance
- [[diario/2026-06-22-20260622.md|2026-06-22]] — reações Telegram, protocolo test-then-act, Crawlee/Apify
- [[diario/2026-06-22-20260621.md|2026-06-21]] — sessão anterior (ver arquivo para detalhes)
- [[diario/2026-06-22-reacoes-telegram.md|2026-06-22 reações]] — testes de reação no Telegram, descoberta de message-id
- [[diario/2026-06-22-descoberta-id-mensagem.md|2026-06-22 message-id]] — técnica de descoberta sequencial de IDs
- [[diario/2026-06-20.md|2026-06-20]] — fix do /update destruindo dados; migração para wiki Karpathy
- [[diario/2026-06-19.md|2026-06-19]] — otimizações estruturais da wiki
