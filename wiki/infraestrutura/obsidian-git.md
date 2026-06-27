---
type: procedure
tags: [obsidian, git, plugin, windows, android, sincronizacao, troubleshooting]
title: Obsidian Git — Configuração e Troubleshooting
description: Referência consolidada de todos os problemas já encontrados com o plugin obsidian-git, causas raiz e soluções definitivas
timestamp: 2026-06-26T00:00:00-03:00
status: stable
---

# Obsidian Git — Configuração e Troubleshooting

Página única de referência. **Sempre atualize aqui antes de tentar qualquer solução nova.**

---

## Setup atual (estado definitivo)

| Item | Desktop (Windows) | Android |
|---|---|---|
| Vault path | onde o Obsidian apontar | `~/storage/shared/ai-memory-wiki` = `/storage/emulated/0/ai-memory-wiki` |
| Plugin instalado | ✅ | ✅ |
| Credenciais | Windows Credential Manager (automático) | `data.json` local (gitignored, precisa configurar 1x por device) |
| `data.json` | ❌ gitignored (correto) | ❌ gitignored (correto) |

**Remote atual:** `https://github.com/omgiova/wiki.git`
(repo foi renomeado de `ai-memory-wiki` para `wiki` — o redirect funciona, mas atualizar o remote é recomendado via `git remote set-url origin https://github.com/omgiova/wiki.git`)

---

## .gitignore — o que está ignorado e por quê

```gitignore
# Estado local da janela — não sincronizar
.obsidian/workspace.json
.obsidian/workspace-mobile.json

# Token do GitHub — NUNCA commitar
.obsidian/plugins/obsidian-git/data.json

# Preferências locais por dispositivo (cada device tem as suas)
.obsidian/app.json
.obsidian/appearance.json
.obsidian/graph.json

# Lixeira do Obsidian
.trash/
```

> `app.json`, `appearance.json` e `graph.json` foram adicionados ao gitignore em 2026-06-26 e removidos do rastreamento com `git rm --cached`. Antes disso estavam commitados como `{}` vindos do Android, causando diff imediato no Windows a cada abertura.

---

## Problemas já encontrados e soluções

### Problema 1 — Plugin sumia a cada abertura (Windows e Android)

**Sintoma:** plugin obsidian-git desaparecia após abrir o Obsidian. Precisava reinstalar e reconfigurar manualmente.

**Causa raiz:** a pasta `.obsidian/` nunca foi commitada. Qualquer operação git que tocasse a pasta apagava os arquivos do plugin.

**Solução aplicada (2026-06-24):**
1. Commitou `.obsidian/` completo a partir do Android (plugin já estava instalado lá)
2. Windows fez `git pull` — restaurou os arquivos
3. Plugin passou a persistir em ambos os devices

---

### Problema 2 — "Discard All" obrigatório antes de cada Pull (Windows)

**Sintoma:** toda vez que o Obsidian era aberto no Windows, era necessário dar "Discard All" antes de conseguir Pull.

**Causa raiz:** `app.json` e `appearance.json` estavam commitados como `{}` (vindos do Android). Ao abrir no Windows, o Obsidian populava esses arquivos com preferências locais — gerando diff imediato.

**Solução aplicada (2026-06-26):**
1. `app.json`, `appearance.json` e `graph.json` adicionados ao `.gitignore`
2. `git rm --cached .obsidian/app.json .obsidian/appearance.json .obsidian/graph.json`
3. Commit + push do fix

---

### Problema 3 — Pull bloqueado no Android após o fix do Problema 2

**Sintoma:**
```
Your local changes to the following files would be overwritten by checkout:
.obsidian/graph.json
```

**Causa:** Android ainda tinha `graph.json` como arquivo local modificado. O pull tentou deletar o arquivo do rastreamento, mas o git se recusou a sobrescrever arquivo com mudanças locais.

**Solução (2026-06-26):**
- Usar o botão "Discard" no Source Control do Obsidian Android para descartar `graph.json`
- Alternativa via Termux se o botão não aparecer:
  ```bash
  rm ~/storage/shared/ai-memory-wiki/.obsidian/app.json \
     ~/storage/shared/ai-memory-wiki/.obsidian/appearance.json \
     ~/storage/shared/ai-memory-wiki/.obsidian/graph.json
  ```
- Depois Pull no Obsidian. O Obsidian recria os arquivos automaticamente e o git os ignora.

---

### Problema 4 — Pull falha no Windows com dois arquivos (2026-06-26)

**Sintoma:**
```
Pull failed (merge): Updating 466631f..4702b5e
error: Your local changes to the following files would be overwritten by merge:
  .obsidian/graph.json
  wiki/infraestrutura/pendencia-problema-ssh-claude.md
Please commit your changes or stash them before you merge. Aborting
```

**Causa raiz (dois problemas distintos):**
1. **`graph.json`** — ainda estava sendo rastreado no Windows porque o Windows nunca tinha puxado o commit do `git rm --cached` (fix do Problema 2). O Windows ficou desatualizado por vários commits.
2. **`pendencia-problema-ssh-claude.md`** — o arquivo foi **renomeado** para `termux-ssh-claude.md` no servidor (commit `d099004`), mas o Windows ainda tinha o arquivo com o nome antigo e com edições locais não commitadas.

**Solução:**
1. No Obsidian Windows → Source Control → verificar se `pendencia-problema-ssh-claude.md` tem conteúdo valioso localmente
2. Se não tiver (ou já estiver no servidor): **Discard All** → **Pull**
3. O pull vai apagar `pendencia-problema-ssh-claude.md` e trazer `termux-ssh-claude.md` (versão final) + desrastrear `graph.json`

**Por que o Windows ficou tão desatualizado?** O problema 2 foi corrigido no servidor, mas o Windows não fez Pull depois. Os pulls subsequentes falhavam por causa do `graph.json` local, criando um ciclo.

---

## Padrão de erro — quando ver "would be overwritten by merge"

Este erro sempre significa: **git quer sobrescrever um arquivo que você tem localmente modificado, mas não commitado**.

**Diagnóstico rápido:**
1. Quais arquivos estão listados no erro?
2. São arquivos de configuração do Obsidian (`.obsidian/`)? → pode descartar com segurança
3. São arquivos `.md` do vault? → verificar se têm conteúdo local valioso antes de descartar
4. Commitar o que for valioso → Pull

**Fix genérico (quando não há nada valioso para salvar):**
Source Control → **Discard All** → **Pull**

---

## Configuração recomendada do obsidian-git (data.json — local, não commitado)

Para evitar que pulls falhem por mudanças locais não commitadas, configurar no plugin:
- **"Commit all changes before pulling"** → ativado
- **"Pull interval"** → 0 (pull manual, não automático — evita conflitos inesperados)

> `data.json` é gitignored — cada device configura o seu. Esta recomendação vale para todos os devices.

---

## Erros históricos do agente (para não repetir)

1. **`git clean -fd .obsidian/`** sem verificar o que estava na pasta — apagou o plugin instalado no Android
2. Não commitar imediatamente após editar arquivos (violação do AGENTS.md)
3. Afirmar que não haveria conflito entre devices sem verificar — contradição na mesma sessão
4. Fazer `git rm --cached` no servidor sem alertar que o Windows precisaria de atenção especial no próximo pull

---

## Pontos de atenção contínuos

- **`core-plugins.json` ainda é rastreado** — se divergir entre PC e Android (plugin ativado num e não no outro), pode gerar conflito. Monitorar.
- **`data.json` precisa ser configurado 1x por device novo** — não está no git
- **Nunca rodar `git clean -fd .obsidian/`** sem confirmar que os arquivos estão commitados
- **Remote URL:** repo foi renomeado. Redirect funciona mas atualizar é recomendado:
  ```bash
  git remote set-url origin https://github.com/omgiova/wiki.git
  ```
- **Quando um fix é aplicado no servidor**, sempre alertar que o próximo pull em cada device pode precisar de atenção (especialmente se o fix mexeu com arquivos gitignored ou renomeações)

---

## Conexões

- [[wiki/diario/2026-06-24-obsidian-git-setup.md]] — registro detalhado da sessão de setup inicial e erros cometidos
- [[wiki/infraestrutura/vps.md]] — caminho do vault no Android documentado aqui
- [[wiki/infraestrutura/termux-ssh-claude.md]] — arquivo renomeado de pendencia-problema-ssh-claude.md
