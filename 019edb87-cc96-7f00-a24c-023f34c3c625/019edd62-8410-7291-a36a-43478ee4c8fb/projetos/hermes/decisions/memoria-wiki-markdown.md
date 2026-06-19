---
tier: semantic
---
## Decisão: Memória Baseada em Markdown + FTS5 + SQLite + Git

**Data:** 2026-06-17  
**Contexto:** A memory() do Hermes tem limite de 2.200 chars e é uma caixa preta sem estrutura.

### O problema
- memory() do Hermes = cache de sessão, não é memória durável
- Sem taxonomia (decisão, gotcha, procedimento, regra ficam tudo misturado)
- Sem versionamento (não dá diff, não dá rollback)
- Sem portabilidade (preso ao Hermes)

### A solução: ai-memory + wiki markdown

| Camada | Tecnologia | Função |
|---|---|---|
| Fonte da verdade | **Markdown puro** | Editável no Obsidian, grep, diff, git |
| Índice | **SQLite + FTS5** | Busca全文 rápida, embeddings opcionais |
| Versionamento | **Git** | Histórico, diff, rollback |
| Servidor | **Rust** | Rápido, baixo consumo |
| Acesso | **MCP tools** | Hermes consulta/escreve automaticamente |
| Visual | **Web UI** | Browser + Obsidian (Windows) |

### Estrutura adotada
```
projetos/
  infra-vps/      → Docker, Swarm, Traefik
  n8n/            → workflows
  node-red/       → flows, Alexa
  remotion/       → vídeos
  hermes/         → config, skills, regras
_global/          → conhecimento geral
```

Dentro de cada projeto: `decisions/`, `gotchas/`, `procedures/`, `rules/`.

### Próximos passos
- Sincronizar wiki com GitHub (repo privado)
- Conectar Obsidian no Windows ao mesmo repo
- Popular com conhecimento acumulado
