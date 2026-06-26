---
type: session
tags: [hermes, claude, ssh, termux, remote-control, bug]
title: Problema SSH/Claude — Sessões Travando no Remote Control
description: Diagnóstico de 3 problemas que causam travamento de sessões Claude via Remote Control (Termux/Android) — curl sem timeout, processo zumbi e prompt de aprovação invisível
timestamp: 2026-06-26T00:00:00-03:00
status: draft
---

# Problema SSH/Claude — Sessões Travando no Remote Control

Diagnosticado em 2026-06-26. Explica por que sessões do Claude via Remote Control aparecem como "Executando..." indefinidamente.

---

## Diagnóstico Geral

Execute cada etapa em ordem. Cada comando já combina múltiplas verificações numa só chamada.

**Etapa 1 — Processos Claude ativos e há quanto tempo existem** *(Problema 2)* ✅ FEITO (2026-06-26)
```bash
ps aux | grep -i claude | grep -v grep; echo "---tempo-vivo---"; ps -eo pid,etime,cmd | grep -i claude | grep -v grep
```
*O que observar:* mais de 1 processo = zumbi acumulado. Campo `etime` mostra há quanto tempo cada um existe.

**Resultado (2026-06-26):** Encontrados 2 processos ativos (após kill acidental da sessão VS Code — ver nota abaixo):
- PID 3584042 — sem terminal, 1h05min — claude gerenciado pelo VS Code extensionHost (~146 MB RAM, 1.8%)
- PID 3595909 — pts/1, ~17min — sessão Remote Control ativa (atual)

**Aprendizado importante — processo daemon do VS Code não é zumbi:**
O processo sem terminal com flags `--output-format stream-json --input-format stream-json` é iniciado **automaticamente pelo VS Code extensionHost** ao conectar via SSH, mesmo sem abrir nenhuma aba ou sessão explícita. Ele se comunica via sockets, não via terminal. Não é zumbi — morre junto com o VS Code ao desconectar.

**Erro cometido na sessão anterior:** outra instância do Claude identificou incorretamente esse processo daemon como "sua própria sessão" e deu `kill` no processo interativo (pts/0) achando que era zumbi — derrubando a própria sessão. Regra: **nunca dar kill em processo claude sem confirmar o PID da sessão atual primeiro** (verificar o pts/N da sessão ativa).

**Como identificar corretamente:**
- Processo daemon (sem terminal, `?`) com flags `--output-format stream-json` = extensão VS Code, não matar
- Processo interativo (`pts/N`) = sessão ativa do Claude Code, não matar sem certeza

**Etapa 2 — Hooks configurados, todos os curls e curls sem timeout** *(Problema 1)* ✅ FEITO (2026-06-26)
```bash
cat ~/.claude/settings.json; echo "---curls-encontrados---"; grep -rn "curl" ~/.claude/ 2>/dev/null; echo "---curls-SEM-max-time---"; grep -rn "curl" ~/.claude/ 2>/dev/null | grep -v "max-time"
```
*O que observar:* a seção `hooks` no settings.json; linhas na parte `curls-SEM-max-time` = vulnerabilidades a corrigir.

**Resultado (2026-06-26):** 7 hooks do sistema `ai-memory` (legado) encontrados no `settings.json` — todos apontando para `http://127.0.0.1:49374` (servidor já down). Os curls nos scripts tinham `--max-time 0.5s`, então não travavam sessões. Porém os hooks eram inúteis e disparavam em vão a cada evento.

**Ação tomada:** removidos os 7 hooks do `settings.json` e deletada a pasta `/root/.local/share/ai-memory/`. Settings ficou limpo com apenas `model` e `theme`. Problema 1 = **não se aplicava** (curls já tinham timeout), mas limpeza feita de qualquer forma.

**Etapa 3 — Logs de sessões recentes e erros registrados** *(Problemas 1 e 2)*
```bash
ls -lth ~/.claude/logs/ 2>/dev/null | head -10; echo "---erros-nos-logs---"; grep -rih "timeout\|hang\|error\|killed\|signal\|stuck" ~/.claude/logs/ 2>/dev/null | tail -30
```
*O que observar:* datas dos logs (indicam quando sessões existiram); linhas de erro mostram padrão dos travamentos.

**Etapa 4 — Configuração de permissões automáticas** *(Problema 3)*
```bash
grep -A5 "permissions\|autoApprove\|allow\|approve" ~/.claude/settings.json 2>/dev/null; echo "---settings-completo---"; cat ~/.claude/settings.json
```
*O que observar:* se há ferramentas com aprovação automática configurada. Ausência total = toda tool pede [y/n] no terminal, invisível no app.

---

## Problema 1 — curl sem --max-time no script de startup

**Causa direta do travamento.**

O script de startup do Hermes executava `curl` sem `--max-time` para buscar dados de MCPs/skills/webhooks. Quando a chamada travou (sem resposta do servidor), o Claude ficou bloqueado esperando o bash terminar — indefinidamente.

**Fix:**
```bash
# Antes (problemático)
curl https://...

# Depois
curl --max-time 10 https://...
```

Aplicar em todos os `curl` dentro de scripts que rodam no startup.

**Status:** Pendente aplicar na VPS.

### Diagnóstico — etapas para verificar estado atual

Execute cada etapa em ordem. Cada uma é independente e pode ser pedida como "faça a etapa N".

**Etapa 1 — Ver hooks configurados no Claude Code**
```bash
cat ~/.claude/settings.json | grep -A5 "hooks"
```
*Objetivo:* identificar quais hooks existem e quais scripts eles chamam.

**Etapa 2 — Encontrar todos os `curl` nos scripts de hook**
```bash
grep -rn "curl" ~/.claude/ 2>/dev/null
```
*Objetivo:* listar todas as chamadas curl nos scripts da configuração do Claude.

**Etapa 3 — Listar arquivos na pasta de configuração**
```bash
ls -la ~/.claude/
```
*Objetivo:* ver quais scripts e arquivos existem na pasta de configuração do Claude.

**Etapa 4 — Identificar curls SEM proteção de timeout**
```bash
grep -rn "curl" ~/.claude/ 2>/dev/null | grep -v "max-time"
```
*Objetivo:* filtrar apenas as linhas com `curl` que **não** têm `--max-time`. Cada linha que aparecer é um curl problemático.

**Etapa 5 — Ver conteúdo completo de cada script identificado**

Após a Etapa 2 ou 4 revelarem os arquivos, ler cada um completo:
```bash
cat <caminho-do-script-encontrado>
```
*Objetivo:* entender o contexto de cada curl problemático antes de aplicar o fix.

**Etapa 6 — Aplicar o fix**

Para cada curl sem `--max-time` encontrado, adicionar a flag:
```bash
# Substituir em cada arquivo afetado
sed -i 's/curl \(https\)/curl --max-time 10 \1/g' <caminho-do-script>
```
Ou editar manualmente se a linha for mais complexa.

**Etapa 7 — Verificar que não sobrou nenhum curl sem proteção**
```bash
grep -rn "curl" ~/.claude/ 2>/dev/null | grep -v "max-time"
```
*Objetivo:* confirmar que o output está vazio (nenhum curl desprotegido).

---

## Problema 2 — Processo zumbi da sessão anterior (pts/0)

Quando uma sessão é "cancelada" pelo app, o processo no terminal (pts/0 no Termux) **não morre**. O Claude fica em estado `sleeping`, sem filhos (script já morreu), mas ainda ocupando memória.

**Sintoma:** `ps aux | grep claude` mostra processo de sessão anterior ainda vivo.

**Referência:** PID 3382543 identificado na sessão de diagnóstico.

**Riscos:**
- Consumo de memória acumulado com múltiplas sessões canceladas
- Potencial conflito com sessões futuras

**Fix imediato:**
```bash
kill <PID>
# ou para limpar todos os processos zumbi do claude:
pkill -f "claude"
```

**Fix permanente:** Configurar script de cleanup que mata processos órfãos do Claude ao iniciar nova sessão.

**Status:** Pendente implementar o cleanup automático.

---

## Problema 3 — Prompt de aprovação invisível no Remote Control

**Explica a maioria dos "travamentos" históricos.**

Quando o Claude precisa de aprovação para uma ferramenta, o prompt aparece **no terminal** (pts/0, Termux). Mas o usuário está olhando para o **app Claude no celular**, que só mostra "Executando..." — sem nenhum prompt visível.

```
App Claude (celular) → mostra "Executando..."
Terminal Termux      → mostra [y/n] aguardando resposta
```

Resultado: sessão aparentemente travada sem forma de responder, a não ser abrir o Termux e responder manualmente.

**Workaround atual:** Abrir o Termux e verificar se há prompt pendente.

**Possíveis soluções (não implementadas):**
- Configurar permissões automáticas para tools que não precisam de aprovação manual
- Investigar se Remote Control tem alguma forma de expor o prompt no app

**Status:** Pendente investigar solução definitiva.

---

## Problema Extra — ai-memory server down

O servidor `ai-memory` (porta 49374) estava down no momento do diagnóstico.

- MCP ai-memory desabilitado (migração para wiki Karpathy já feita — ver [[pendencias/proximos-passos.md]])
- Não causa travamento (hooks têm timeout de 0.5s)
- Pode ser ignorado se a migração para wiki estiver completa

**Status:** Provavelmente irrelevante após migração.

---

## Resumo de prioridades

| # | Problema | Impacto | Esforço | Status |
|---|---|---|---|---|
| 1 | curl sem --max-time | Alto — trava sessão | Baixo | Pendente |
| 3 | Prompt invisível no app | Alto — travamento frequente | Médio | Pendente investigar |
| 2 | Processo zumbi | Baixo — memória | Baixo | Pendente |
