# AGENTS.md

> ⚠️ **Leia este arquivo completo antes de qualquer ação — pesquisar, criar, editar, renomear, mover ou excluir arquivos.**

---

## Após qualquer operação

> ⚠️ **OBRIGATÓRIO:** após criar, editar, renomear, mover ou excluir **qualquer** arquivo:
>
> 1. Appendar entrada em `log.md` (ver formato abaixo)
> 2. Executar:
> ```bash
> git -C /root/wiki add -A
> git -C /root/wiki commit -m "<tipo>(<escopo>): <descrição>"
> git -C /root/wiki push origin main
> ```
> Nunca deixar o repositório com mudanças não commitadas ou não enviadas ao remoto.

**Tipos:** `edit`, `ingest`, `query`, `lint`, `session`, `chore` — mesmo tipo da entrada do log.  
**Escopo:** nome da pasta ou arquivo principal afetado (ex: `vps`, `wiki`, `firecrawl`)

**Branches:** trabalhar em `main` por padrão. Feature branch só se explicitamente solicitado.

### Formato do log.md

`log.md` é append-only e cronológico. Nunca editar entradas existentes.

```
## [YYYY-MM-DD] <tipo> | <escopo> — <descrição>
- <detalhe relevante>
- <páginas tocadas, se aplicável>
```

| tipo | quando usar |
|---|---|
| `ingest` | nova fonte adicionada à wiki |
| `query` | pergunta gerou página nova arquivada |
| `lint` | health-check executado |
| `edit` | edição significativa em página existente |
| `session` | resumo de encerramento de sessão na wiki |
| `chore` | tarefa técnica sem mudança de conteúdo (trigger, teste, config) |

---

## O que é este repositório

**Hermes AI Memory Wiki** — base de conhecimento persistente do agente Hermes (assistente pessoal do Giovani). Segue o padrão **LLM Wiki de Karpathy**: Markdown puro como fonte da verdade, versionado por Git, indexado por SQLite + FTS5, visualizado no Obsidian.

**Princípio central:** conhecimento durável vai para a wiki. A `memory()` do Hermes (2.200 chars, volátil) é cache de sessão, não fonte da verdade.

---

## Autorização

Toda edição requer autorização explícita do usuário. A autorização vale só para o escopo pedido. Ao encontrar inconsistência: reportar, não corrigir — `"Encontrei [problema] em [arquivo]. Devo corrigir?"`

Arquivos que o agente NUNCA deve modificar: `.git/`, `.obsidian/`, `AGENTS.md`

---

## Estrutura da wiki

Ver [[index.md]] — fonte de verdade da estrutura. Ao criar, renomear ou remover qualquer arquivo ou pasta, atualizar o `index.md`.

---

## Frontmatter obrigatório (OKF)

Todo arquivo `.md` da wiki (exceto `AGENTS.md`, `index.md` e `log.md`) deve começar com:

```yaml
---
type: <tipo>
tags: [tag1, tag2]
title: <título>
description: <uma linha descrevendo o conteúdo>
timestamp: <ISO 8601>
status: <status>
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
| `draft` | em construção, pode estar incompleto ou desatualizado — usar ao criar página nova |
| `stable` | confiável, revisado |
| `deprecated` | obsoleto, não usar como referência |

---

## Regras de escrita

1. **Uma página por conceito** — não duplicar. Se o conceito já existe, atualizar a página existente ou criar um link.
2. **Wikilinks** para conectar páginas relacionadas: `[[caminho/arquivo.md|texto]]`
3. **`raw/`** é imutável — arquivos ali nunca são editados, apenas adicionados
4. **`wiki/diario/`** segue o padrão `wiki/diario/YYYY-MM-DD-sufixo-descritivo.md` com `type: daily`
5. **Tags** em kebab-case, no plural, sem acentos (ex: `sessoes`, não `sessoe` ou `sessão`)

---

## Operações

### Ingest — adicionar arquivo ou conhecimento novo

Checklist obrigatório. Executar **nesta ordem** a cada novo arquivo criado:

1. **Criar o arquivo** no diretório correto (`wiki/automacao/`, `wiki/infraestrutura/`, `wiki/historico/`, `raw/`, etc.)
2. **Adicionar frontmatter OKF completo** — `type`, `tags`, `title`, `description`, `timestamp`, `status`
3. **Adicionar wikilinks** para páginas relacionadas (e atualizar as páginas relacionadas para linkar de volta)
4. **Atualizar `index.md`** — nova entrada com link + descrição na seção correta; árvore sincronizada com `git ls-files`

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
