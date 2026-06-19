---
type: gotcha
tags: [docker, gotcha, infra-vps, n8n, projetos, traefik]
title: 1. Testar se o backend responde pela overlay
description: bash systemctl restart docker - Não perde volumes, imagens, configs ou networks - Todos os serviços do Swarm sobem de novo com IPVS repovoado - Downtime de ~20-30 segundos
timestamp: 2026-06-18T00:00:00+00:00
---


## Gotcha: IPVS Table Vazia Após Recriação de Containers

**Data:** 2026-06-18

### Sintomas
- DNS do Swarm resolve nomes de serviço (ex: `projetos_n8n_editor` → 10.11.25.x)
- Ping/curl pros IPs overlay dá **"Host is unreachable"**
- Traefik retorna **502 Bad Gateway**
- `ipvsadm -Ln` retorna tabela **vazia** (nenhuma regra de forwarding)

### O que causa
- `docker service scale <serviço>=0` → mata container original, IPVS não repovoa
- `docker service update --network-rm/--network-add` → recria container com novo IP
- `docker service update --force` → recria container

### Solução
```bash
systemctl restart docker
```
- Não perde volumes, imagens, configs ou networks
- Todos os serviços do Swarm sobem de novo com IPVS repovoado
- Downtime de ~20-30 segundos

### Diagnóstico rápido
```bash
# 1. Testar se o backend responde pela overlay
docker exec easypanel-traefik wget -qO- http://projetos_n8n_editor:5678/ 2>&1

# 2. Verificar IPVS
ipvsadm -Ln

# 3. Se tabela vazia → restart docker
```

## 📂 Navegação
[[infra-vps/gotchas/INDEX.md|📂 Voltar para gotchas]]

## 🔗 Relacionados (mesmo grupo)
- [[infra-vps/gotchas/docker-swarm-ipvs-overlay.md|docker-swarm-ipvs-overlay]]

## 🔗 Conexões entre projetos
- [[hermes/decisions/memoria-wiki-markdown.md|memoria-wiki-markdown]]
- [[hermes/rules/conhecimento-na-wiki.md|conhecimento-na-wiki]]
- [[hermes/sessoes/2026-06-18-recuperacao-e-wiki.md|2026-06-18-recuperacao-e-wiki]]
- [[hermes-config/notes/arquitetura-vps.md|arquitetura-vps]]
- [[infra-vps/decisions/arquitetura-rede-swarm.md|arquitetura-rede-swarm]]
- [[infra-vps/procedures/diagnostico-502-traefik.md|diagnostico-502-traefik]]

