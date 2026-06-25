---
type: concept
tags: [hermes, api, rest, openapi, infraestrutura]
title: Hermes Agent — API REST (OpenAPI)
description: Referência completa dos endpoints da API do Hermes Agent (gerada automaticamente do /openapi.json)
timestamp: 2026-06-25T00:29:23-03:00
status: stable
---

# Hermes Agent — API REST

> **Gerado automaticamente** a partir de `GET /openapi.json` (Hermes Agent v0.17.0, OAS 3.1).
> Para atualizar: execute `/root/scripts/update-hermes-api-wiki.sh`

- **Base URL:** `http://localhost:9119`
- **Auth:** POST `/auth/password-login` com `{"username":"...","password":"...","provider":"basic"}` → cookie `hermes_session_at`
- **Docs interativas:** `http://localhost:9119/docs`

---

## Índice de grupos

| Grupo | Endpoints |
|---|---|
| [actions](#actions) | status de ações em background |
| [analytics](#analytics) | uso e modelos |
| [audio](#audio) | síntese de voz, transcrição |
| [auth](#auth) | login, sessão, ticket WS |
| [config](#config) | configuração geral |
| [credentials](#credentials) | pool de credenciais |
| [cron](#cron) | jobs agendados |
| [curator](#curator) | curador de memória |
| [dashboard](#dashboard) | plugins, tema, fonte |
| [env](#env) | variáveis de ambiente |
| [files](#files) | arquivos gerenciados |
| [fs](#fs) | filesystem local |
| [gateway](#gateway) | start/stop/restart |
| [hermes](#hermes) | update do Hermes |
| [logs](#logs) | logs do sistema |
| [mcp](#mcp) | servidores MCP |
| [media](#media) | imagens locais |
| [memory](#memory) | provedor de memória |
| [messaging](#messaging) | Telegram, WhatsApp |
| [model](#model) | modelos de IA |
| [ops](#ops) | backup, doctor, hooks |
| [pairing](#pairing) | pareamento de clientes |
| [plugins](#plugins) | kanban, achievements |
| [portal](#portal) | status do portal |
| [profiles](#profiles) | perfis do agente |
| [providers](#providers) | OAuth de provedores LLM |
| [sessions](#sessions) | histórico de sessões |
| [skills](#skills) | habilidades do agente |
| [status](#status) | status geral |
| [system](#system) | stats do sistema |
| [tools](#tools) | toolsets, computer-use |
| [webhooks](#webhooks) | webhooks de entrada |

---

## actions

- `GET /api/actions/{name}/status` **Get Action Status** — Acompanha log de uma ação em background e informa se ainda está rodando.

## analytics

- `GET /api/analytics/models` **Get Models Analytics** — Estatísticas ricas por modelo para o painel de Models.
- `GET /api/analytics/usage` **Get Usage Analytics** — Uso agregado por período.

## audio

- `GET /api/audio/elevenlabs/voices` **Get Elevenlabs Voices** — Lista vozes do ElevenLabs quando a API key está configurada.
- `POST /api/audio/speak` **Speak Text** — Sintetiza fala e retorna áudio como base64 data URL.
- `POST /api/audio/transcribe` **Transcribe Audio Upload** — Transcreve um upload de áudio.

## auth

- `GET /api/auth/me` **Auth Me** — Retorna a sessão verificada como JSON.
- `GET /api/auth/providers` **Auth Providers** — Lista provedores de autenticação disponíveis.
- `POST /api/auth/ws-ticket` **Auth Ws Ticket** — Gera ticket de uso único para conexão WebSocket autenticada.
- `GET /auth/callback` **Auth Callback** — Callback OAuth.
- `GET /auth/login` **Auth Login** — Redireciona para login.
- `POST /auth/logout` **Auth Logout** — Encerra sessão.
- `POST /auth/password-login` **Auth Password Login** — Autentica username/password contra um provedor básico. Body: `{"username","password","provider":"basic"}`

## config

- `GET /api/config` **Get Config** — Retorna config atual.
- `PUT /api/config` **Update Config** — Atualiza config.
- `GET /api/config/defaults` **Get Defaults** — Valores padrão.
- `GET /api/config/raw` **Get Config Raw** — Texto bruto do config.yaml + caminho resolvido.
- `PUT /api/config/raw` **Update Config Raw** — Sobrescreve config.yaml bruto.
- `GET /api/config/schema` **Get Schema** — JSON Schema da configuração.

## credentials

- `GET /api/credentials/pool` **List Credential Pool** — Lista o pool de credenciais rotativas.
- `POST /api/credentials/pool` **Add Credential Pool Entry** — Adiciona entrada ao pool.
- `DELETE /api/credentials/pool/{provider}/{index}` **Remove Credential Pool Entry** — Remove entrada do pool (index base 1).

## cron

- `GET /api/cron/blueprints` **List Cron Blueprints** — Catálogo de blueprints como form schemas.
- `POST /api/cron/blueprints/instantiate` **Instantiate Blueprint** — Preenche slots de um blueprint e cria o job.
- `GET /api/cron/delivery-targets` **Get Cron Delivery Targets** — Destinos de entrega disponíveis no dropdown de cron.
- `POST /api/cron/fire` **Cron Fire Webhook** — Webhook de disparo de cron gerenciado.
- `GET /api/cron/jobs` **List Cron Jobs** — Lista todos os jobs.
- `POST /api/cron/jobs` **Create Cron Job** — Cria novo job.
- `GET /api/cron/jobs/{job_id}` **Get Cron Job** — Detalhes de um job.
- `PUT /api/cron/jobs/{job_id}` **Update Cron Job** — Atualiza job.
- `DELETE /api/cron/jobs/{job_id}` **Delete Cron Job** — Remove job.
- `POST /api/cron/jobs/{job_id}/pause` **Pause Cron Job** — Pausa job.
- `POST /api/cron/jobs/{job_id}/resume` **Resume Cron Job** — Retoma job pausado.
- `GET /api/cron/jobs/{job_id}/runs` **List Cron Job Runs** — Sessões geradas pelo job, mais recentes primeiro.
- `POST /api/cron/jobs/{job_id}/trigger` **Trigger Cron Job** — Dispara job manualmente agora.

## curator

- `GET /api/curator` **Get Curator Status** — Status do curador de memória.
- `PUT /api/curator/paused` **Set Curator Paused** — Pausa/retoma o curador.
- `POST /api/curator/run` **Run Curator** — Dispara revisão do curador agora (em background).

## dashboard

- `POST /api/dashboard/agent-plugins/install` **Post Agent Plugin Install** — Instala plugin de agente.
- `DELETE /api/dashboard/agent-plugins/{name}` **Delete Agent Plugin** — Remove plugin.
- `POST /api/dashboard/agent-plugins/{name}/disable` **Post Agent Plugin Disable** — Desativa plugin.
- `POST /api/dashboard/agent-plugins/{name}/enable` **Post Agent Plugin Enable** — Ativa plugin.
- `POST /api/dashboard/agent-plugins/{name}/update` **Post Agent Plugin Update** — Atualiza plugin.
- `GET /api/dashboard/font` **Get Dashboard Font** — Fonte ativa do dashboard.
- `PUT /api/dashboard/font` **Set Dashboard Font** — Define fonte (persiste em config.yaml).
- `PUT /api/dashboard/plugin-providers` **Put Plugin Providers** — Persiste seleção de provedor de memória/contexto.
- `GET /api/dashboard/plugins` **Get Dashboard Plugins** — Plugins descobertos (exceto ocultos pelo usuário).
- `GET /api/dashboard/plugins/hub` **Get Plugins Hub** — Metadados unificados de agent plugins + dashboard extensions.
- `GET /api/dashboard/plugins/rescan` **Rescan Dashboard Plugins** — Força re-scan de plugins.
- `POST /api/dashboard/plugins/{name}/visibility` **Post Plugin Visibility** — Alterna visibilidade na sidebar.
- `PUT /api/dashboard/theme` **Set Dashboard Theme** — Define tema ativo (persiste em config.yaml).
- `GET /api/dashboard/themes` **Get Dashboard Themes** — Temas disponíveis + tema ativo.

## env

- `GET /api/env` **Get Env Vars** — Lista variáveis de ambiente (valores redacted).
- `PUT /api/env` **Set Env Var** — Define variável.
- `DELETE /api/env` **Remove Env Var** — Remove variável.
- `POST /api/env/reveal` **Reveal Env Var** — Retorna valor real (não redacted) de uma variável.

## files

- `GET /api/files` **List Managed Files** — Lista arquivos gerenciados (parâmetro: `?path=`).
- `DELETE /api/files` **Delete Managed File** — Remove arquivo gerenciado.
- `GET /api/files/download` **Download Managed File** — Stream de arquivo como attachment. Aceita `?token=` para auth em downloads diretos.
- `POST /api/files/mkdir` **Create Managed Directory** — Cria diretório.
- `GET /api/files/read` **Read Managed File** — Lê conteúdo de arquivo (parâmetro: `?path=`).
- `POST /api/files/upload` **Upload Managed File** — Upload de arquivo.
- `POST /api/files/upload-stream` **Upload Managed File Stream** — Upload de arquivo em stream.

## fs

- `GET /api/fs/default-cwd` **Fs Default Cwd** — Diretório de trabalho padrão.
- `GET /api/fs/git-root` **Fs Git Root** — Raiz do repositório git detectado.
- `GET /api/fs/list` **Fs List** — Lista filesystem local (parâmetro: `?path=`).
- `GET /api/fs/read-data-url` **Fs Read Data Url** — Lê arquivo como data URL.
- `GET /api/fs/read-text` **Fs Read Text** — Lê arquivo como texto.

## gateway

- `POST /api/gateway/restart` **Restart Gateway** — Reinicia o hermes-gateway em background.
- `POST /api/gateway/start` **Start Gateway** — Inicia gateway.
- `POST /api/gateway/stop` **Stop Gateway** — Para gateway.

## hermes

- `POST /api/hermes/update` **Update Hermes** — Dispara `hermes update` em background.
- `GET /api/hermes/update/check` **Check Hermes Update** — Verifica se há atualização disponível (sem aplicar).

## logs

- `GET /api/logs` **Get Logs** — Logs do sistema (parâmetros: `?lines=`, `?follow=`).

## mcp

- `GET /api/mcp/catalog` **List Mcp Catalog** — Catálogo de MCPs aprovados pela Nous Research.
- `POST /api/mcp/catalog/install` **Install Mcp Catalog Entry** — Instala MCP do catálogo no config.yaml.
- `GET /api/mcp/servers` **List Mcp Servers** — Lista servidores MCP configurados.
- `POST /api/mcp/servers` **Add Mcp Server** — Adiciona servidor MCP.
- `DELETE /api/mcp/servers/{name}` **Remove Mcp Server** — Remove servidor MCP.
- `PUT /api/mcp/servers/{name}/enabled` **Set Mcp Server Enabled** — Ativa/desativa servidor MCP.
- `POST /api/mcp/servers/{name}/test` **Test Mcp Server** — Conecta, lista ferramentas e desconecta. Retorna lista de tools.

## media

- `GET /api/media` **Get Media** — Retorna imagem local do gateway como base64 data URL. Parâmetro: `?path=`

## memory

- `GET /api/memory` **Get Memory Status** — Status do provedor de memória.
- `PUT /api/memory/provider` **Set Memory Provider** — Define provedor de memória ativo.
- `GET /api/memory/providers/{name}/config` **Get Memory Provider Config** — Config de um provedor específico.
- `PUT /api/memory/providers/{name}/config` **Update Memory Provider Config** — Atualiza config do provedor.
- `POST /api/memory/providers/{provider}/oauth/start` **Start Memory Oauth** — Inicia fluxo OAuth de um provedor de memória.
- `GET /api/memory/providers/{provider}/oauth/status` **Memory Oauth Status** — Polling do OAuth: idle | pending | connected | error.
- `POST /api/memory/reset` **Reset Memory** — Reseta a memória do provedor ativo.

## messaging

- `GET /api/messaging/platforms` **Get Messaging Platforms** — Lista plataformas de mensagens configuradas.
- `PUT /api/messaging/platforms/{platform_id}` **Update Messaging Platform** — Atualiza config de plataforma.
- `POST /api/messaging/platforms/{platform_id}/test` **Test Messaging Platform** — Envia mensagem de teste.
- `POST /api/messaging/telegram/onboarding/start` **Start Telegram Onboarding** — Inicia processo de onboarding do Telegram.
- `GET /api/messaging/telegram/onboarding/{pairing_id}` **Get Telegram Onboarding Status** — Status do onboarding.
- `DELETE /api/messaging/telegram/onboarding/{pairing_id}` **Cancel Telegram Onboarding** — Cancela onboarding.
- `POST /api/messaging/telegram/onboarding/{pairing_id}/apply` **Apply Telegram Onboarding** — Confirma e aplica onboarding.

## model

- `GET /api/model/auxiliary` **Get Auxiliary Models** — Atribuições atuais de modelos auxiliares.
- `GET /api/model/info` **Get Model Info** — Metadados do modelo configurado atualmente.
- `GET /api/model/options` **Get Model Options** — Provedores autenticados + listas de modelos disponíveis.
- `GET /api/model/recommended-default` **Get Recommended Default Model** — Modelo padrão recomendado para um provedor recém-autenticado.
- `POST /api/model/set` **Set Model Assignment** — Atribui modelo ao slot principal ou a um slot auxiliar.

## ops

- `POST /api/ops/backup` **Run Backup** — Executa backup.
- `GET /api/ops/checkpoints` **List Checkpoints** — Lista checkpoints do rollback shadow store (somente leitura).
- `POST /api/ops/checkpoints/prune` **Prune Checkpoints** — Remove checkpoints antigos.
- `POST /api/ops/config-migrate` **Run Config Migrate** — Executa migração de configuração.
- `POST /api/ops/debug-share` **Run Debug Share Endpoint** — Upload de relatório de debug redacted + logs. Retorna URLs.
- `POST /api/ops/doctor` **Run Doctor** — Diagnóstico do sistema.
- `POST /api/ops/dump` **Run Dump** — Dump de estado.
- `GET /api/ops/hooks` **List Hooks** — Lista hooks shell configurados no config.yaml.
- `POST /api/ops/hooks` **Create Hook** — Adiciona hook shell ao config.yaml.
- `DELETE /api/ops/hooks` **Delete Hook** — Remove hook do config.yaml.
- `POST /api/ops/import` **Run Import** — Importa dados.
- `POST /api/ops/prompt-size` **Run Prompt Size** — Calcula tamanho do prompt atual.
- `POST /api/ops/security-audit` **Run Security Audit** — Auditoria de segurança.

## pairing

- `GET /api/pairing` **List Pairing** — Lista pares autorizados.
- `POST /api/pairing/approve` **Approve Pairing** — Aprova solicitação de pareamento.
- `POST /api/pairing/clear-pending` **Clear Pending Pairing** — Limpa solicitações pendentes.
- `POST /api/pairing/revoke` **Revoke Pairing** — Revoga acesso de um par.

## plugins

### kanban

- `GET /api/plugins/kanban/board` **Get Board** — Board completo agrupado por coluna de status.
- `GET /api/plugins/kanban/boards` **List Boards** — Todos os boards com contagem de tasks e slug ativo.
- `POST /api/plugins/kanban/boards` **Create Board Endpoint** — Cria board (idempotente por slug).
- `PATCH /api/plugins/kanban/boards/{slug}` **Rename Board** — Atualiza metadados do board.
- `DELETE /api/plugins/kanban/boards/{slug}` **Delete Board** — Arquiva ou deleta board.
- `POST /api/plugins/kanban/boards/{slug}/switch` **Switch Board** — Define board ativo.
- `GET /api/plugins/kanban/tasks` (via POST) **Create Task** — Cria nova task.
- `GET /api/plugins/kanban/tasks/{task_id}` **Get Task** — Detalhes de task.
- `PATCH /api/plugins/kanban/tasks/{task_id}` **Update Task** — Atualiza task.
- `DELETE /api/plugins/kanban/tasks/{task_id}` **Delete Task** — Remove task.
- `POST /api/plugins/kanban/tasks/bulk` **Bulk Update** — Aplica mesmo patch a múltiplas tasks.
- `POST /api/plugins/kanban/tasks/{task_id}/decompose` **Decompose Task** — Decompõe task em subtasks via LLM auxiliar.
- `POST /api/plugins/kanban/tasks/{task_id}/specify` **Specify Task** — Detalha task via LLM e promove de triage.
- `POST /api/plugins/kanban/tasks/{task_id}/reassign` **Reassign Task** — Reatribui task a outro perfil.
- `POST /api/plugins/kanban/tasks/{task_id}/reclaim` **Reclaim Task** — Libera claim de worker em uma task running.
- `POST /api/plugins/kanban/tasks/{task_id}/terminate` (via runs) **Terminate Run** — Termina processo worker de run ativo.
- `GET /api/plugins/kanban/tasks/{task_id}/log` **Get Task Log** — stdout/stderr do worker.
- `GET /api/plugins/kanban/tasks/{task_id}/attachments` **List Task Attachments**
- `POST /api/plugins/kanban/tasks/{task_id}/attachments` **Upload Task Attachment**
- `POST /api/plugins/kanban/tasks/{task_id}/comments` **Add Comment**
- `GET /api/plugins/kanban/stats` **Get Stats** — Contagens por status/assignee + idade da task mais antiga.
- `GET /api/plugins/kanban/assignees` **Get Assignees** — Perfis com contagem de tasks.
- `GET /api/plugins/kanban/workers/active` **List Active Workers** — Workers em execução no momento.
- `GET /api/plugins/kanban/runs/{run_id}` **Get Run** — Linha da tabela task_runs.
- `GET /api/plugins/kanban/runs/{run_id}/inspect` **Inspect Run** — Stats de processo via psutil.
- `GET /api/plugins/kanban/orchestration` **Get Orchestration Settings**
- `PUT /api/plugins/kanban/orchestration` **Set Orchestration Settings**
- `GET /api/plugins/kanban/profiles` **List Profile Roster**
- `GET /api/plugins/kanban/config` **Get Config**
- `GET /api/plugins/kanban/diagnostics` **List Diagnostics**
- `POST /api/plugins/kanban/dispatch` **Dispatch**
- `GET /api/plugins/kanban/home-channels` **Get Home Channels**
- `POST /api/plugins/kanban/links` **Add Link**
- `DELETE /api/plugins/kanban/links` **Delete Link**
- `POST /api/plugins/kanban/tasks/{task_id}/home-subscribe/{platform}` **Subscribe Home**
- `DELETE /api/plugins/kanban/tasks/{task_id}/home-subscribe/{platform}` **Unsubscribe Home**

### achievements

- `GET /api/plugins/hermes-achievements/achievements` **Achievements** — Lista conquistas.
- `GET /api/plugins/hermes-achievements/recent-unlocks` **Recent Unlocks** — Desbloqueios recentes.
- `GET /api/plugins/hermes-achievements/scan-status` **Scan Status** — Status do scan de conquistas.
- `GET /api/plugins/hermes-achievements/sessions/{session_id}/badges` **Session Badges** — Badges de uma sessão.
- `POST /api/plugins/hermes-achievements/rescan` **Rescan** — Força re-scan.
- `POST /api/plugins/hermes-achievements/reset-state` **Reset State** — Reseta estado de conquistas.

## portal

- `GET /api/portal` **Get Portal Status** — Status do portal externo.

## profiles

- `GET /api/profiles` **List Profiles Endpoint** — Lista perfis instalados.
- `POST /api/profiles` **Create Profile Endpoint** — Cria perfil.
- `GET /api/profiles/active` **Get Active Profile Endpoint** — Perfil ativo no momento.
- `POST /api/profiles/active` **Set Active Profile Endpoint** — Define perfil ativo.
- `GET /api/profiles/sessions` **Get Profiles Sessions** — Lista unificada de sessões de todos os perfis.
- `PATCH /api/profiles/{name}` **Rename Profile Endpoint** — Renomeia perfil.
- `DELETE /api/profiles/{name}` **Delete Profile Endpoint** — Deleta perfil.
- `POST /api/profiles/{name}/describe-auto` **Describe Profile Auto** — Gera descrição via LLM auxiliar.
- `PUT /api/profiles/{name}/description` **Update Profile Description** — Define descrição do perfil.
- `PUT /api/profiles/{name}/model` **Update Profile Model** — Define modelo principal do perfil.
- `POST /api/profiles/{name}/open-terminal` **Open Profile Terminal** — Abre terminal do perfil.
- `GET /api/profiles/{name}/setup-command` **Get Profile Setup Command** — Comando de setup do perfil.
- `GET /api/profiles/{name}/soul` **Get Profile Soul** — Personalidade/soul do perfil.
- `PUT /api/profiles/{name}/soul` **Update Profile Soul** — Atualiza soul do perfil.

## providers

- `GET /api/providers/oauth` **List Oauth Providers** — Provedores LLM com capacidade OAuth e status atual.
- `POST /api/providers/oauth/{provider_id}/start` **Start Oauth Login** — Inicia fluxo OAuth.
- `POST /api/providers/oauth/{provider_id}/submit` **Submit Oauth Code** — Submete código de autorização (PKCE).
- `GET /api/providers/oauth/{provider_id}/poll/{session_id}` **Poll Oauth Session** — Polling do status da sessão OAuth.
- `DELETE /api/providers/oauth/{provider_id}` **Disconnect Oauth Provider** — Desconecta provedor OAuth.
- `DELETE /api/providers/oauth/sessions/{session_id}` **Cancel Oauth Session** — Cancela sessão OAuth pendente.
- `POST /api/providers/validate` **Validate Provider Credential** — Testa credencial de provedor antes de salvar.

## sessions

- `GET /api/sessions` **Get Sessions** — Lista sessões (filtros: `?profile=`, `?archived=`, `?limit=`).
- `GET /api/sessions/stats` **Get Session Stats** — Estatísticas do armazenamento de sessões.
- `GET /api/sessions/search` **Search Sessions** — Busca por ID e conteúdo via FTS5.
- `GET /api/sessions/empty/count` **Count Empty Sessions** — Contagem de sessões vazias encerradas.
- `DELETE /api/sessions/empty` **Delete Empty Sessions** — Remove sessões vazias encerradas.
- `POST /api/sessions/prune` **Prune Sessions** — Remove sessões encerradas com mais de N dias.
- `POST /api/sessions/bulk-delete` **Bulk Delete Sessions** — Remove múltiplas sessões em uma transação.
- `GET /api/sessions/{session_id}` **Get Session Detail** — Detalhes de sessão.
- `DELETE /api/sessions/{session_id}` **Delete Session** — Remove sessão.
- `PATCH /api/sessions/{session_id}` **Rename Session** — Renomeia ou arquiva sessão.
- `GET /api/sessions/{session_id}/messages` **Get Session Messages** — Mensagens da sessão.
- `GET /api/sessions/{session_id}/export` **Export Session** — Exporta sessão como JSON (metadata + messages).
- `GET /api/sessions/{session_id}/latest-descendant` **Get Session Latest Descendant** — Descendente mais recente da sessão.

## skills

- `GET /api/skills` **Get Skills** — Lista skills instaladas.
- `POST /api/skills` **Create Skill** — Cria skill customizada (SKILL.md) pelo editor do dashboard.
- `GET /api/skills/content` **Get Skill Content** — Retorna SKILL.md bruto para o editor.
- `PUT /api/skills/content` **Update Skill Content** — Substitui SKILL.md de uma skill existente.
- `PUT /api/skills/toggle` **Toggle Skill** — Ativa/desativa skill.
- `GET /api/skills/hub/search` **Search Skills Hub** — Busca no hub de skills.
- `GET /api/skills/hub/preview` **Preview Skill Hub** — Pré-visualiza SKILL.md + metadados de uma skill do hub.
- `GET /api/skills/hub/scan` **Scan Skill Hub** — Scan de segurança de skill do hub (sem instalar).
- `POST /api/skills/hub/install` **Install Skill Hub** — Instala skill do hub.
- `POST /api/skills/hub/uninstall` **Uninstall Skill Hub** — Desinstala skill do hub.
- `POST /api/skills/hub/update` **Update Skills Hub** — Atualiza skills instaladas do hub.
- `GET /api/skills/hub/sources` **List Skills Hub Sources** — Fontes configuradas do hub + proveniência de skills instaladas.

## status

- `GET /api/status` **Get Status** — Status geral do Hermes Agent (health check).

## system

- `GET /api/system/stats` **Get System Stats** — Estatísticas de host e processo (CPU, memória, disco).

## tools

- `GET /api/tools/toolsets` **Get Toolsets** — Lista toolsets disponíveis.
- `PUT /api/tools/toolsets/{name}` **Toggle Toolset** — Ativa/desativa toolset para plataforma desktop.
- `GET /api/tools/toolsets/{name}/config` **Get Toolset Config** — Matriz de provedores + status de keys para o painel de config.
- `PUT /api/tools/toolsets/{name}/env` **Save Toolset Env** — Persiste API keys de um toolset.
- `POST /api/tools/toolsets/{name}/post-setup` **Run Toolset Post Setup** — Executa hook de pós-instalação do provedor em background.
- `PUT /api/tools/toolsets/{name}/provider` **Select Toolset Provider** — Persiste seleção de provedor para um toolset.
- `GET /api/tools/computer-use/status` **Get Computer Use Status** — Prontidão do Computer Use na plataforma atual.
- `POST /api/tools/computer-use/permissions/grant` **Grant Computer Use Permissions** — Concede permissões de Computer Use em background.

## webhooks

- `GET /api/webhooks` **List Webhooks** — Lista webhooks configurados.
- `POST /api/webhooks` **Create Webhook** — Cria webhook.
- `POST /api/webhooks/enable` **Enable Webhooks** — Habilita sistema de webhooks.
- `DELETE /api/webhooks/{name}` **Delete Webhook** — Remove webhook.
- `PUT /api/webhooks/{name}/enabled` **Set Webhook Enabled** — Ativa/desativa webhook específico.

---

## Como manter atualizado

```bash
/root/scripts/update-hermes-api-wiki.sh
```

O script autentica no Hermes, busca `/openapi.json`, gera este arquivo e commita na wiki.
Pode ser chamado manualmente após um `hermes update` ou agendado via cron.

---

## Conexões

- [[infraestrutura/hermes.md]] — identidade, stack e preferências do Hermes
- [[infraestrutura/vps.md]] — VPS onde o Hermes roda
- [[pendencias/proximos-passos.md]] — pendências ativas
