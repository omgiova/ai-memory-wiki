# Docker Swarm: IPVS table vazia após recriação de containers

## Sintomas
- DNS do Swarm resolve nomes de serviço (ex: `projetos_n8n_editor` → 10.11.25.x)
- Ping/curl dá **"Host is unreachable"** nos IPs da overlay (10.11.x.x, 10.0.1.x)
- Traefik retorna 502 Bad Gateway
- `ipvsadm -Ln` retorna tabela **vazia**

## Causa
A tabela IPVS (responsável por rotear tráfego dos VIPs pros containers reais) não foi repovoada após:
- `docker service scale <serviço>=0` (mata container original)
- `docker service update --network-rm/--network-add` (recria container com novo IP)
- `docker service update --force` (recria container)

## Solução
```bash
systemctl restart docker
```
- Não perde volumes, imagens, configs ou networks
- Todos os serviços do Swarm sobem de novo com IPVS repovoado
- Downtime de ~20-30 segundos

## Arquitetura do roteamento
```
Usuário → Traefik (DNS: projetos-n8n-editor.igkokh.easypanel.host)
         → Traefik resolve service name via DNS do Swarm
         → DNS devolve VIP (ex: 10.11.25.32)
         → IPVS roteia do VIP pro container real (ex: 10.11.25.33:5678)
```

Traefik usa config estática em `/etc/easypanel/traefik/config/main.yaml`:
```yaml
projetos_n8n_editor: "http://projetos_n8n_editor:5678/"
projetos_nodered:    "http://projetos_nodered:8800/"
```

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

## Lição aprendida
**NUNCA** mexer em redes overlay ou escalar serviços pra 0 sem entender o IPVS. O correto pra resolver 502 com overlay quebrado é:
1. `ipvsadm -Ln` (verificar tabela)
2. `systemctl restart docker` (repovoar)
