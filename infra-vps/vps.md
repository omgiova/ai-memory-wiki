---
type: concept
tags: [docker, n8n, vps, infra]
title: Infraestrutura do VPS
description: Hostinger KVM 2 — hardware, serviços rodando, stack Docker Swarm, IPVS, problemas conhecidos
timestamp: 2026-06-18T00:00:00+00:00
---

# Infraestrutura do VPS

## Hardware

- **VPS:** Hostinger KVM 2
- **SO:** Ubuntu 22.04 (Linux 6.8.0)
- **Disco:** 96GB (60GB livre)
- **RAM:** 7.8GB (4.8GB disponível)

## Serviços rodando

| Serviço | Porta | Função |
|---|---|---|
| Hermes Agent | 9119 | Dashboard |
| n8n | — | Automação (MCP) |
| Node-RED | 8800 | Automação residencial + Alexa |
| ai-memory | 49374 | Memória agêntica |

## Stack de desenvolvimento

- **Runtime:** Node.js, Python 3.11
- **Container:** Docker
- **Banco:** SQLite (Hermes, ai-memory)

## Docker Swarm

O VPS roda Docker Swarm (modo cluster single-node). Todo serviço roda em uma overlay network gerenciada pelo EasyPanel.

### Redes overlay

- `easypanel` (10.11.x.x) — rede principal, Traefik incluso
- `easypanel-projetos` (10.0.1.x) — rede secundária

### IPVS (IP Virtual Server)

**Conceito:** IPVS é um módulo do kernel Linux que faz balanceamento de carga em nível de rede (camada 4, TCP/UDP). O Docker Swarm usa IPVS para rotear tráfego dos VIPs (Virtual IPs) pros containers reais.

```
Usuário → Traefik (443)
         → DNS do Swarm resolve nome do serviço (ex: n8n_editor)
         → DNS retorna VIP (10.11.x.x)
         → IPVS roteia do VIP pro container real (10.11.x.x:5678)
```

#### A tabela IPVS pode ficar vazia após:

- `docker service scale <serviço>=0` — mata container, IPVS não repovoa
- `docker service update --network-rm/--network-add` — recria container com novo IP
- `docker service update --force` — recria container

#### Sintomas

- ✅ DNS resolve nomes de serviço (ex: `n8n_editor` → 10.11.25.x)
- ❌ Ping/curl pros IPs da overlay dá **"Host is unreachable"**
- 💀 Traefik retorna **502 Bad Gateway**
- `ipvsadm -Ln` retorna tabela **vazia** (nenhuma regra de forwarding)

#### Diagnóstico

1. **Testar endpoint direto via Traefik**
   ```bash
   curl -sI https://projetos-n8n-editor.igkokh.easypanel.host/
   ```
   200 → não é IPVS. 502 → continuar.

2. **Testar conectividade de dentro do Traefik**
   ```bash
   docker exec easypanel-traefik wget -qO- http://n8n_editor:5678/
   ```
   "Host is unreachable" → overlay ou IPVS.

3. **Verificar IPVS**
   ```bash
   ipvsadm -Ln
   ```
   Vazia → `systemctl restart docker`. Tem regras → verificar IPs.

4. **Verificar containers e logs**
   ```bash
   docker service ls
   docker service ps <serviço>
   docker service logs <serviço> --tail 20
   ```

#### Solução

```bash
systemctl restart docker
```

- Não perde volumes, imagens, configs ou networks
- Todos os serviços do Swarm sobem de novo com IPVS repovoado
- Downtime de ~20-30 segundos

#### Prevenção (regras)

1. **NUNCA** executar `docker service update` sem autorização explícita
   - Principalmente: `--network-rm`, `--network-add`, `--force`, `scale`
2. **NUNCA** usar `docker service scale <serviço>=0`
   - Mata o container permanentemente, IPVS pode não repovoar
3. **Sempre** verificar IPVS primeiro ao diagnosticar 502
   - `ipvsadm -Ln` antes de qualquer alteração
4. **Sempre** testar conectividade de dentro do Traefik
   - `docker exec easypanel-traefik wget -qO- http://<serviço>:<porta>/`

### Serviços nas redes overlay

| Serviço | Rede easypanel | Rede easypanel-projetos |
|---|---|---|
| Traefik | ✅ | ❌ |
| n8n_editor | ✅ | ✅ |
| n8n_webhook | ✅ | ✅ |
| n8n_worker | ❌ | ✅ |
| Node-RED | ✅ | ✅ |
| Postgres | ✅ | ✅ |
| Redis | ✅ | ✅ |
| Evolution API | ✅ | ❌ |

## Visão geral do ecossistema

A stack roda num único VPS Hostinger KVM 2 com Ubuntu. O Hermes Agent é o orquestrador de automação pessoal, n8n e Node-RED cuidam de workflows, e o ai-memory mantém a base de conhecimento persistente.
