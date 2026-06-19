---
tier: semantic
---
---
type: gotcha
tags: [docker, infra-vps, n8n, traefik]
title: IPVS Table Vazia + Traefik 502
description: Docker Swarm perde roteamento IPVS após scale=0, network-rm/add ou --force. DNS resolve, tráfego não chega. Solução: systemctl restart docker.
timestamp: 2026-06-18T00:00:00+00:00
---

# IPVS Table Vazia + Traefik 502

## Como o tráfego chega nos serviços (arquitetura)

Usuário → Traefik (porta 443) → DNS do Swarm resolve service name → DNS retorna VIP (10.11.x.x) → IPVS roteia do VIP pro container real

## Sintomas
- DNS do Swarm resolve nomes de serviço
- Ping/curl pros IPs overlay dá "Host is unreachable"
- Traefik retorna 502 Bad Gateway
- ipvsadm -Ln retorna tabela vazia

## Causas
- docker service scale <serviço>=0 — mata container, IPVS não repovoa
- docker service update --network-rm/--network-add — recria container
- docker service update --force — recria container

## Diagnóstico
1. curl -sI https://projetos-n8n-editor.igkokh.easypanel.host/
2. docker exec easypanel-traefik wget -qO- http://projetos_n8n_editor:5678/
3. ipvsadm -Ln
4. docker service ls / docker service ps <serviço>

## Solução
systemctl restart docker — downtime ~20-30s, não perde dados

## Prevenção
- NUNCA docker service update sem autorização
- NUNCA docker service scale <serviço>=0
- Sempre verificar IPVS primeiro ao diagnosticar 502
- Sempre testar conectividade de dentro do Traefik
