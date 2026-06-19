---
type: gotcha
tags: [docker, infra-vps, n8n, traefik]
title: IPVS Table Vazia + Traefik 502
description: Docker Swarm perde roteamento IPVS após scale=0, network-rm/add ou --force. DNS resolve, tráfego não chega. Solução: systemctl restart docker.
timestamp: 2026-06-18T00:00:00+00:00
---

# IPVS Table Vazia + Traefik 502

**Data:** 2026-06-18

## Como o tráfego chega nos serviços (arquitetura)

```
Usuário → Traefik (porta 443)
         → DNS do Swarm resolve service name (projetos_n8n_editor)
         → DNS retorna VIP (10.11.x.x)
         → IPVS roteia do VIP pro container real (10.11.x.x:5678)
```

- Traefik usa **config estática** em `/etc/easypanel/traefik/config/main.yaml`
- URLs usam nomes DNS do Swarm: `http://projetos_n8n_editor:5678/`
- Node-RED: `PORT=8800`, Docker mapeia `8800→1880` — funciona porque Traefik acessa via overlay direto na porta 8800
- Redes overlay: `easypanel` (10.11.x.x) e `easypanel-projetos` (10.0.1.x)
- Traefik só está na rede `easypanel`

### Por que essa arquitetura
- EasyPanel gerencia o Traefik automaticamente
- DNS do Swarm + IPVS = load balancing transparente entre réplicas
- Config estática evita dependência de service discovery externo

## Sintomas

- DNS do Swarm resolve nomes de serviço (ex: `projetos_n8n_editor` → 10.11.25.x)
- Ping/curl pros IPs da overlay dá **"Host is unreachable"**
- Traefik retorna **502 Bad Gateway**
- `ipvsadm -Ln` retorna tabela **vazia** (nenhuma regra de forwarding)

## O que causa

A tabela IPVS (responsável por rotear tráfego dos VIPs pros containers reais) não é repovoada após:

- `docker service scale <serviço>=0` — mata container original, IPVS não repovoa
- `docker service update --network-rm/--network-add` — recria container com novo IP
- `docker service update --force` — recria container

## Diagnóstico (passo a passo)

1. **Testar o endpoint direto via Traefik**
   ```bash
   curl -sI https://projetos-n8n-editor.igkokh.easypanel.host/
   ```
   Se 200 → não é problema de rota. Se 502 → continuar.

2. **Testar conectividade de dentro do Traefik**
   ```bash
   docker exec easypanel-traefik wget -qO- http://projetos_n8n_editor:5678/
   ```
   - Se **"Host is unreachable"** → overlay ou IPVS
   - Se conectar → problema no backend

3. **Verificar IPVS**
   ```bash
   ipvsadm -Ln
   ```
   - Se tabela vazia → `systemctl restart docker`
   - Se tem regras → verificar se os IPs dos backends estão corretos

4. **Verificar containers estão rodando**
   ```bash
   docker service ls
   docker service ps <serviço>
   ```

5. **Verificar logs do serviço**
   ```bash
   docker service logs <serviço> --tail 20
   ```

## Solução

```bash
systemctl restart docker
```

- Não perde volumes, imagens, configs ou networks
- Todos os serviços do Swarm sobem de novo com IPVS repovoado
- Downtime de ~20-30 segundos

### Quando chamar o restart
- IPVS table vazia + DNS resolve + containers rodando
- Apenas `systemctl restart docker` — não perde dados

## Prevenção (regras)

1. **NUNCA** executar `docker service update` sem autorização explícita
   - Principalmente: `--network-rm`, `--network-add`, `--force`, `scale`
2. **NUNCA** usar `docker service scale <serviço>=0`
   - Mata o container permanentemente
   - Perde histórico do container (logs, métricas de uptime)
   - IPVS table pode não repovoar no novo container
3. **Sempre** verificar IPVS primeiro ao diagnosticar 502
   - `ipvsadm -Ln` antes de qualquer alteração
   - Se tabela vazia → restart docker, não mexer em redes
4. **Sempre** testar conectividade de dentro do Traefik
   - `docker exec easypanel-traefik wget -qO- http://<serviço>:<porta>/`
   - Isso separa problema de overlay de problema de aplicação

## Serviços no host

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

## Penalidade por violação
Perda de confiança do usuário + horas de troubleshooting desnecessário. Já aconteceu.
