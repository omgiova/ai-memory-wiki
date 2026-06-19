---
tags:
- ai-memory
- hermes
- obsidian
- wiki
pinned: true
tier: semantic
---
# Wiki

**Criado:** 2026-06-17  
**Última atualização:** 2026-06-19

## Contexto

A `memory()` do Hermes tem limite de 2.200 chars e é uma caixa preta sem estrutura. Sem taxonomia (decisão, gotcha, procedimento, regra ficam tudo misturado), sem versionamento, sem portabilidade.

A solução: **ai-memory** — um servidor Rust que indexa markdown puro em SQLite + FTS5, versionado por Git.

| Camada | Tecnologia | Função |
|---|---|---|
| Fonte da verdade | **Markdown puro** | Editável no Obsidian, grep, diff, git |
| Índice | **SQLite + FTS5** | Busca rápida, embeddings opcionais |
| Versionamento | **Git** | Histórico, diff, rollback |
| Servidor | **Rust** | Rápido, baixo consumo |
| Acesso | **MCP tools** (Hermes) | Consulta/escreve automaticamente |
| Visual | **Web UI** (`:49374/web`) + Obsidian | Navegação visual |

## Estrutura atual do vault

```
wiki.md                        → este arquivo (conceito central)
hermes-config/                 → configuração do Hermes
  hermes-config.md             → identidade, regras, stack
hermes/                        → Hermes Agent
  conceitos/                   → conceitos transversais
  procedures/                  → procedures
  sessoes/                     → registro de sessões
  todo/                        → próximos passos
infra-vps/vps.md               → Docker, Swarm, Traefik, IPVS, VPS
```

## Como funciona

1. **Escrever:** agente cria/atualiza markdown no vault
2. **Indexar:** o ai-memory server indexa automaticamente em SQLite + FTS5
3. **Consultar:** Hermes usa MCP tools (`memory_query`, `memory_read_page`)
4. **Versionar:** Git. Vault sincronizado com GitHub e Obsidian no celular

## Regras

1. **Conhecimento durável vai pra wiki, não pra `memory()`**
2. **Memory() do Hermes** é cache de sessão (2.200 chars)
3. **Uma página por conceito** — OKF: type, title, description, tags, timestamp
4. **Cross-links** entre páginas com wikilinks
5. **Todo arquivo** tem frontmatter OKF

## Histórico

### Fundação (2026-06-18)

Após comandos Docker Swarm errados que quebraram n8n e Node-RED (IPVS corrompida → 502), foi criada a estrutura inicial da wiki com 7 páginas — o embrião do vault atual.

### 5 Pilares

1. Memória de longo prazo para agentes
2. Markdown como fonte da verdade
3. Padrão LLM Wiki do Karpathy (Raw → Wiki → Schema)
4. Loop de auto-aprendizado (sessão → extração → memória)
5. Handoff entre agentes

### Pendente

- Migrar conhecimento acumulado pra wiki
- Configurar auto-improve loop
- Garantir handoff entre agentes
- Revisar SOUL.md + criar AGENTS.md
