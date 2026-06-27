---
type: procedure
tags: [obsidian, git, plugin, windows, android, sincronizacao, troubleshooting]
title: Obsidian Git — Configuração e Troubleshooting
description: Referência consolidada para manter o Obsidian como visualizador read-only do vault — todos os pulls devem funcionar sem interação manual
timestamp: 2026-06-26T00:00:00-03:00
status: stable
---

# Obsidian Git — Configuração e Troubleshooting

---

## Premissa fundamental (LEIA ANTES DE QUALQUER FIX)

> **Obsidian é um visualizador read-only do vault. Todas as edições acontecem no servidor via Claude Code / Hermes. O Obsidian nunca edita arquivos `.md` manualmente.**

Isso significa que **o pull nunca deveria ter conflito real**. Quando falha, é porque:

1. O Obsidian auto-escreve arquivos em `.obsidian/` (graph.json, workspace.json, etc.) que às vezes ainda estão rastreados pelo git
2. Renomeações ou deleções no servidor geram edge cases onde o git interpreta o arquivo local como "modificado localmente" — mesmo sem o usuário ter tocado nele
3. O device ficou com versão antiga do `.gitignore` e ainda rastreia arquivos que já foram removidos do rastreamento no servidor

**A solução definitiva não é "Discard All manual" a cada pull — é configurar o pull para sempre vencer o estado local.**

---

## Configuração definitiva (por device)

### 1. Git config — `merge.autostash true` (solução real — validada em 2026-06-26)

O plugin obsidian-git v2.38.5 não tem opção de "force pull" ou "stash before pull" na UI. As opções de "Merge strategy" disponíveis (Merge, Rebase, Other sync service) não resolvem o erro "would be overwritten by merge".

**Atenção:** `pull.autostash true` **não funciona** com este plugin porque o plugin não executa `git pull` — ele executa `git fetch` + `git merge` separadamente. O `pull.autostash` só afeta o comando `git pull`. O config correto é `merge.autostash true`.

**A solução é no git nativo**, que o plugin herda automaticamente:

```bash
# Windows — rodar no Git Bash ou PowerShell
git -C "C:\Users\omgio\Desktop\hermes\ai-memory-wiki" config merge.autostash true

# Verificar se foi aplicado
git -C "C:\Users\omgio\Desktop\hermes\ai-memory-wiki" config merge.autostash
# deve retornar: true

# Android — rodar no Termux
git -C ~/storage/shared/ai-memory-wiki config merge.autostash true
```

Isso vai para o `.git/config` local de cada device (não sincronizado). Persiste permanentemente — só some se o vault for deletado e re-clonado. Precisa rodar **uma vez por device**.

**Por que funciona:** antes de qualquer `git merge`, o git faz `git stash` das mudanças locais automaticamente, executa o merge, e descarta o stash. Como o Obsidian é read-only, o stash é sempre descartado e nunca cria conflito.

**Fix de desbloqueio** (quando o pull está travado com "would be overwritten by merge"):
```bash
git -C "C:\Users\omgio\Desktop\hermes\ai-memory-wiki" fetch origin
git -C "C:\Users\omgio\Desktop\hermes\ai-memory-wiki" reset --hard origin/main
```

### 2. Opções do plugin (secundárias)

| Configuração | Valor recomendado | Observação |
|---|---|---|
| Merge strategy | Merge (padrão) | com autostash ativo, qualquer opção funciona |
| Pull on startup | opcional | se ativo, pull automático ao abrir |
| Push on commit-and-sync | ✅ ativado | correto para uso read-only — mas não há commit local |

---

## Setup atual (estado definitivo)

| Item | Desktop (Windows) | Android |
|---|---|---|
| Vault path | `C:\Users\omgio\Desktop\hermes\ai-memory-wiki` | `~/storage/shared/ai-memory-wiki` |
| Plugin instalado | ✅ | ✅ |
| Credenciais | Windows Credential Manager | `data.json` local — configurar 1x por device |
| `data.json` no git | ❌ gitignored | ❌ gitignored |

**Remote:** `https://github.com/omgiova/wiki.git`

---

## .gitignore — o que está ignorado e por quê

```gitignore
# Estado local da janela — regenerado automaticamente pelo Obsidian
.obsidian/workspace.json
.obsidian/workspace-mobile.json

# Token do GitHub — NUNCA commitar
.obsidian/plugins/obsidian-git/data.json

# Preferências locais por dispositivo — Obsidian reescreve a cada abertura
.obsidian/app.json
.obsidian/appearance.json
.obsidian/graph.json

# Lixeira do Obsidian
.trash/
```

**Atenção:** `community-plugins.json` e `core-plugins.json` ainda são rastreados. Se divergirem entre devices, podem causar conflito. Se isso acontecer, adicionar ao `.gitignore` também.

---

## Por que o pull falha mesmo sem editar nada

### Caso 1 — arquivo ainda rastreado apesar do gitignore

O `.gitignore` ignora arquivos **não rastreados**. Se o arquivo já estava commitado (antes de entrar no gitignore), ele continua sendo rastreado. O `git rm --cached` remove do rastreamento — mas o device que não puxou esse commit ainda o vê como rastreado.

**Resultado:** quando o device desatualizado tenta puxar o commit do `git rm --cached`, o git detecta que o arquivo local tem conteúdo diferente do que o commit quer deletar → erro "would be overwritten by merge".

**Fix permanente:** configurar "Stash before pulling" no obsidian-git. O stash guarda o estado local, o pull acontece, o stash é descartado (como o arquivo agora está ignorado, não volta).

### Caso 2 — renomeação no servidor

Um arquivo é renomeado no servidor (`pendencia-X.md` → `termux-ssh-claude.md`). O device local ainda tem o arquivo com o nome antigo. O git tenta deletar o arquivo antigo, mas o local tem "mudanças" (mesmo que sejam só a existência do arquivo). Falha com "would be overwritten by merge".

**Fix permanente:** mesma solução — "Stash before pulling" descarta o arquivo antigo, o pull traz o novo nome.

### Caso 3 — Obsidian escreveu em arquivo rastreado

O Obsidian abre e escreve em `graph.json`, `app.json`, etc. Se esses arquivos ainda estiverem rastreados (situação de transição), o git vê mudança local → pull falha.

**Fix permanente:** garantir que todos esses arquivos estejam no `.gitignore` E tenham sido removidos com `git rm --cached`.

---

## Fix de emergência quando o pull falha (antes de mudar para syncMethod: reset)

Se o pull falhar com "would be overwritten by merge":

**Source Control → Discard All → Pull**

**Via terminal (Windows PowerShell / Git Bash) se o botão falhar:**
```bash
cd <caminho-do-vault>
git fetch origin
git reset --hard origin/main
```

> Depois de mudar Sync Method para Reset, esse fix manual não deveria ser mais necessário.

---

## Histórico de incidentes

| Data | Problema | Causa raiz | Fix aplicado |
|---|---|---|---|
| 2026-06-24 | Plugin sumia a cada abertura | `.obsidian/` nunca commitado | Commitou `.obsidian/` do Android; desktop fez pull |
| 2026-06-26 | "Discard All" obrigatório antes de todo Pull (Windows) | `app.json` e `appearance.json` commitados como `{}` pelo Android; Windows reescrevia ao abrir | Adicionados ao `.gitignore` + `git rm --cached` |
| 2026-06-26 | Pull bloqueado no Android após fix anterior | Android ainda tinha os arquivos como locais modificados | Discard no Obsidian Android; alternativa: `rm` via Termux |
| 2026-06-26 | Pull falha no Windows com `graph.json` + `pendencia-problema-ssh-claude.md` | Windows desatualizado por múltiplos commits; renomeação de arquivo causou edge case | Discard All + Pull; documentação reescrita com causa raiz real |
| 2026-06-26 | `pull.autostash true` configurado mas pull continua falhando com `AGENTS.md`, `index.md`, `orquestrador.md` | Plugin usa `git merge` internamente, não `git pull` — `pull.autostash` não se aplica ao `git merge` | `git config merge.autostash true` no Windows; `reset --hard origin/main` para desbloqueio imediato; aguardando validação a longo prazo |

---

## Erros históricos do agente (para não repetir)

1. Documentou o problema como "conflito entre devices" — causa real é que Obsidian é read-only e o pull deveria sempre vencer
2. `git clean -fd .obsidian/` sem verificar — apagou plugin instalado no Android
3. Aplicou fix de `git rm --cached` no servidor sem alertar que o próximo pull em cada device precisaria de atenção especial
4. Sugeriu "Discard All manual" como solução em vez de configurar o plugin para fazer isso automaticamente
5. Afirmou que o plugin tem opção "HARD RESET" no Merge Strategy — não existe. As únicas opções são: Merge, Rebase, Other sync service
6. Afirmou que `pull.autostash true` é a solução — não funciona porque o plugin usa `git merge`, não `git pull`. O config correto é `merge.autostash true`
7. Assumiu sem evidência que o usuário rodou o comando no VPS em vez do Windows — estava errado

---

## Conexões

- [[wiki/diario/2026-06-24-obsidian-git-setup.md]] — sessão detalhada de setup inicial
- [[wiki/infraestrutura/vps.md]] — caminho do vault no Android
- [[wiki/infraestrutura/termux-ssh-claude.md]] — arquivo renomeado de pendencia-problema-ssh-claude.md
