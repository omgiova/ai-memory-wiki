---
tier: semantic
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
