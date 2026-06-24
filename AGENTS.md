# AGENTS.md

Instruções para agentes de IA que trabalham neste repositório.  
Compatível com: Claude Code, OpenAI Codex CLI, Manus, Cursor, Windsurf, Gemini CLI.

---

## Git e commits

> ⚠️ **OBRIGATÓRIO:** após criar ou modificar **qualquer** arquivo, executar imediatamente:
> ```bash
> git -C /root/wiki add -A
> git -C /root/wiki commit -m "<tipo>(<escopo>): <descrição>"
> git -C /root/wiki push origin main
> ```
> Nunca deixar o repositório com mudanças não commitadas ou não enviadas ao remoto.

**Formato de commit:**
```
<tipo>(<escopo>): <descrição curta>

- detalhe 1
- detalhe 2
```

**Tipos de commit:** `docs`, `chore`, `fix`, `feat`  
**Escopo:** nome da pasta ou arquivo principal afetado (ex: `vps`, `wiki`, `firecrawl`)

**Exemplos:**
```
docs(vps): adiciona seção de troubleshooting de rede overlay
chore(wiki): atualiza estrutura do vault após criação de diario/
fix(crise-update): corrige typo na tag sessoes
feat(diario): adiciona daily note 2026-06-19
```

**Branches:** trabalhar em `main` por padrão. Feature branch só se explicitamente solicitado.

---

## O que é este repositório

**Hermes AI Memory Wiki** — base de conhecimento persistente do agente Hermes (assistente pessoal do Giovani). Segue o padrão **LLM Wiki de Karpathy**: Markdown puro como fonte da verdade, versionado por Git, indexado por SQLite + FTS5, visualizado no Obsidian.

**Princípio central:** conhecimento durável vai para a wiki. A `memory()` do Hermes (2.200 chars, volátil) é cache de sessão, não fonte da verdade.

---

## Estrutura do vault

> Esta seção deve estar sempre sincronizada com `git ls-files`. Ao criar, renomear ou remover qualquer arquivo ou pasta, atualize aqui.

```
index.md                          → ponto de entrada único (catálogo)
AGENTS.md                         → este arquivo (schema)
│
├── raw/                          → fontes brutas imutáveis
│   └── karpathy-llm-wiki-pattern.md → fonte original do padrão LLM Wiki (Karpathy)
│
└── wiki/                         → páginas geradas e mantidas pelo LLM
    ├── automacao/
    │   ├── firecrawl.md          → busca multi-plataforma com sintaxe site:
    │   └── wiki-review.md        → agente background que salva insights no diario/ a cada 10 turnos
    │
    ├── conhecimento/
    │   ├── wiki.md               → conceito central, regras e histórico da wiki
    │   └── agent-loop-architectures.md  → comparação de loops entre Hermes, OpenClaw, Claude Code, Codex e Cline
    │
    ├── diario/                   → daily notes / memória episódica (YYYY-MM-DD.md)
    │
    ├── historico/
    │   └── crise-update.md       → recuperação de sessões após /update corromper state.db
    │
    ├── infraestrutura/
    │   ├── hermes.md             → identidade, regras, stack e preferências do Hermes
    │   ├── telegram-topicos.md   → mapa completo de chats, tópicos e IDs do Telegram
    │   └── vps.md                → hardware, serviços, Docker Swarm, IPVS
    │
    └── pendencias/
        └── proximos-passos.md    → to-do list ativa
```

---

## Frontmatter obrigatório (OKF)

Todo arquivo `.md` do vault (exceto este) deve começar com:

```yaml
---
type: <tipo>         # veja tipos abaixo
tags: [tag1, tag2]   # kebab-case, plural, sem typos
title: <título>
description: <uma linha descrevendo o conteúdo>
timestamp: <ISO 8601>
status: <status>     # veja status abaixo
---
```

**Tipos válidos:**

| type | uso |
|---|---|
| `concept` | explicação de um conceito ou sistema |
| `procedure` | passo a passo executável |
| `session` | registro de uma sessão ou incidente |
| `todo` | lista de pendências |
| `index` | ponto de entrada / mapa de navegação |
| `raw` | fonte bruta imutável |
| `daily` | daily note episódica |

**Status válidos:**

| status | significado |
|---|---|
| `draft` | em construção, pode estar incompleto ou desatualizado |
| `stable` | confiável, revisado |
| `deprecated` | obsoleto, não usar como referência |

---

## Regras de escrita

1. **Uma página por conceito** — não duplicar. Se o conceito já existe, atualizar a página existente ou criar um link.
2. **Wikilinks** para conectar páginas relacionadas: `[[caminho/arquivo.md|texto]]`
3. **Toda página** tem frontmatter OKF completo + seção de navegação/conexões no final
4. **`raw/`** é imutável — arquivos ali nunca são editados, apenas adicionados
5. **`wiki/diario/`** segue o padrão `wiki/diario/YYYY-MM-DD.md` com `type: daily`
6. **Tags** em kebab-case, no plural, sem acentos (ex: `sessoes`, não `sessoe` ou `sessão`)
7. **`status: draft`** ao criar uma página nova; mudar para `stable` quando revisada

---

## Operações

### Ingest — adicionar arquivo ou conhecimento novo

Checklist obrigatório. Executar **nesta ordem** a cada novo arquivo criado:

1. **Criar o arquivo** no diretório correto (`wiki/automacao/`, `wiki/infraestrutura/`, `wiki/historico/`, `raw/`, etc.)
2. **Adicionar frontmatter OKF completo** — `type`, `tags`, `title`, `description`, `timestamp`, `status`
3. **Adicionar wikilinks** para páginas relacionadas (e atualizar as páginas relacionadas para linkar de volta)
4. **Atualizar `index.md`** — nova entrada com link + descrição de uma linha na seção correta
5. **Atualizar a estrutura do vault** em `AGENTS.md` — deve bater com `git ls-files`
6. **Commitar tudo junto** — um commit por operação de ingest

### Query — responder a uma pergunta com base na wiki

1. Ler `index.md` para identificar páginas relevantes
2. Ler as páginas identificadas
3. Sintetizar resposta com referências (`[[página]]`)
4. Se a resposta for valiosa (análise, comparação, decisão), **criar uma página nova** com ela — bons insights não devem ficar só no chat

### Lint — health-check periódico

Executar quando solicitado pelo usuário:

- Páginas órfãs sem nenhum link apontando pra elas
- Contradições entre páginas (`status: stable` conflitando com info mais recente)
- Conceitos mencionados em várias páginas mas sem página própria
- Entradas no `index.md` sem correspondente em `git ls-files` (e vice-versa)
- Árvore do vault em `AGENTS.md` fora de sync com `git ls-files`

---

## Arquivos que o agente NUNCA deve modificar

```
.git/
.obsidian/
AGENTS.md          ← só o humano edita este arquivo
```

---

## Como os agentes usam este repositório

| Agente | Acesso | Uso típico |
|---|---|---|
| **Hermes** | MCP tools (`memory_query`, `memory_read_page`, `memory_write`) | consultar e escrever conhecimento durante sessões |
| **Claude Code** | lê este arquivo automaticamente na raiz | editar vault, estruturar conhecimento, commits |
| **Manus** | lê AGENTS.md como schema | pesquisar, sintetizar e registrar novos conhecimentos |
| **Codex CLI** | lê AGENTS.md automaticamente | tarefas de escrita e refatoração de páginas |

---

## Referências internas

- [[conhecimento/wiki.md]] — visão geral do sistema, histórico e pilares
- [[infraestrutura/hermes.md]] — identidade e preferências do Hermes
- [[pendencias/proximos-passos.md]] — o que está pendente agora
