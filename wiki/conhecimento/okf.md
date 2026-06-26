---
type: concept
tags: [okf, open-knowledge-format, frontmatter, wiki, padrao, google]
title: Open Knowledge Format (OKF)
description: Padrão open source do Google Cloud para representar conhecimento como arquivos markdown com YAML frontmatter — o mesmo padrão que esta wiki já adota.
timestamp: 2026-06-26T00:00:00-03:00
status: stable
---

# Open Knowledge Format (OKF)

OKF é uma especificação aberta lançada pelo Google Cloud em junho de 2026 que formaliza o padrão [LLM Wiki do Karpathy](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) como um formato interoperável e vendor-neutral.

**Nossa wiki já é um bundle OKF conformante** — chegamos ao mesmo formato independentemente, assim como Obsidian, CLAUDE.md/AGENTS.md e dezenas de outras ferramentas.

## O que é OKF

Um **bundle OKF** é um diretório de arquivos markdown onde cada arquivo representa um **conceito**. Todo conceito tem:

1. Um bloco de **YAML frontmatter** com campos estruturados
2. Um **corpo markdown** com conteúdo livre

**Único campo obrigatório:** `type`. O resto é recomendado mas opcional.

```yaml
---
type: concept          # OBRIGATÓRIO
title: Nome do conceito
description: Uma linha resumindo.
resource: https://uri-do-ativo-real  # para conceitos vinculados a um recurso
tags: [tag1, tag2]
timestamp: 2026-06-26T00:00:00-03:00
---
```

## Campos recomendados (em ordem de prioridade)

| Campo | Uso |
|---|---|
| `title` | Nome legível. Se omitido, consumidores derivam do nome do arquivo |
| `description` | Uma frase para snippets, `index.md` e previews |
| `resource` | URI do ativo real (tabela BigQuery, endpoint de API, etc.) — omitir para conceitos abstratos |
| `tags` | Lista de strings para categorização cruzada |
| `timestamp` | ISO 8601 da última modificação significativa |

Campos extras são permitidos — consumidores OKF devem preservá-los sem reclamar.

## Arquivos reservados

| Arquivo | Função |
|---|---|
| `index.md` | Listagem do diretório para navegação progressiva (sem frontmatter, exceto na raiz do bundle) |
| `log.md` | Histórico cronológico de mudanças naquele escopo |

## Cross-linking

Links entre conceitos usam markdown padrão. Dois formatos:

- **Absoluto (relativo à raiz do bundle):** `[customers](/tables/customers.md)` — recomendado, estável quando arquivos mudam de subdiretório
- **Relativo:** `[outro conceito](./outro.md)`

Consumidores devem tolerar links quebrados — representam conhecimento ainda não escrito.

## Conformance OKF v0.1

Um bundle é conformante se:
1. Todo `.md` não-reservado tem frontmatter YAML parseável
2. Todo frontmatter tem `type` não-vazio
3. `index.md` e `log.md` seguem a estrutura definida na spec quando presentes

## Nossa wiki vs OKF oficial

| Campo | Nossa wiki | OKF oficial |
|---|---|---|
| `type` | sim | obrigatório |
| `title` | sim | recomendado |
| `description` | sim | recomendado |
| `tags` | sim | recomendado |
| `timestamp` | sim | recomendado |
| `status` | **sim (extra)** | não previsto — extensão nossa |
| `resource` | não usamos | recomendado para ativos reais |
| `index.md` | sim (raiz) | opcional em qualquer nível |
| `log.md` | não usamos | opcional |

**Nossa wiki está acima do mínimo OKF** — `status` é uma extensão válida. Poderíamos adicionar `resource` em páginas que descrevem ativos reais (ex: endpoints do Hermes API).

## Repositório oficial

> **Consultar sempre para updates, novos bundles de exemplo e evolução da spec:**
> https://github.com/GoogleCloudPlatform/knowledge-catalog

Estrutura relevante do repo:
- `okf/SPEC.md` — especificação completa (v0.1 Draft)
- `okf/README.md` — introdução e como usar o agente de referência
- `okf/bundles/` — exemplos reais: GA4, StackOverflow, Bitcoin
- `okf/src/reference_agent/` — agente Python que gera bundles OKF a partir de BigQuery
- `toolbox/` — visualizador HTML interativo de bundles

## Documentação salva localmente

Arquivos em `raw/google-okf/` (fontes imutáveis):

- [[raw/google-okf/SPEC.md|SPEC.md]] — especificação oficial completa (v0.1)
- [[raw/google-okf/README.md|README.md]] — README do repo okf/ com instruções do agente de referência
- [[raw/google-okf/introducing-the-open-knowledge-format.md|Introducing the Open Knowledge Format]] — blog post do Google Cloud (jun/2026): motivação, princípios de design, ecossistema

## Três princípios do design (do blog oficial)

1. **Minimamente opinativo** — só `type` é obrigatório; o resto é livre ao produtor
2. **Independência produtor/consumidor** — qualquer um produz, qualquer um consome; o formato é o contrato
3. **Formato, não plataforma** — sem SDK obrigatório, sem conta proprietária, sem lock-in

## Navegação

- [[wiki/conhecimento/wiki.md|Wiki como Fonte da Verdade]] — como nossa wiki funciona
- [[raw/google-okf/SPEC.md|SPEC.md oficial]]
- [[index.md|🏠 Index]]
