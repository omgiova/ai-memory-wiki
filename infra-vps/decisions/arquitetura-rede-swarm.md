---
type: decision
tags: [decision, docker, infra-vps, n8n, projetos, traefik]
title: Decisão: Arquitetura de Rede do Docker Swarm
description: Usuário → Traefik porta 443 → DNS do Swarm resolve service name projetosn8neditor → DNS retorna VIP 10.11.x.x → IPVS roteia do VIP pro container real 10.11.x.x:5678
timestamp: 2026-06-18T00:00:00+00:00
---


## Decisão: Arquitetura de Rede do Docker Swarm

**Data:** 2026-06-18  
**Contexto:** Após troubleshooting de 502 no Traefik, mapeamos como o roteamento realmente funciona.

### Como o tráfego chega nos serviços

```
Usuário → Traefik (porta 443)
         → DNS do Swarm resolve service name (projetos_n8n_editor)
         → DNS retorna VIP (10.11.x.x)
         → IPVS roteia do VIP pro container real (10.11.x.x:5678)
```

### Descobertas
- Traefik usa **config estática** em `/etc/easypanel/traefik/config/main.yaml`
- URLs usam **nomes DNS do Swarm**: `http://projetos_n8n_editor:5678/`
- Node-RED: `PORT=8800`, mas Docker mapeia `8800→1880`. Funciona porque Traefik acessa via overlay direto na porta 8800 do container
- Redes overlay: `easypanel` (10.11.x.x) e `easypanel-projetos` (10.0.1.x)
- Traefik só está na rede `easypanel`

### Por que essa arquitetura
- EasyPanel gerencia o Traefik automaticamente
- DNS do Swarm + IPVS = load balancing transparente entre réplicas
- Config estática evita dependência de service discovery externo

## 📂 Navegação
[[infra-vps/decisions/INDEX.md|📂 Voltar para decisions]]

## 🔗 Conexões entre projetos
- [[hermes/decisions/memoria-wiki-markdown.md|memoria-wiki-markdown]]
- [[hermes/rules/conhecimento-na-wiki.md|conhecimento-na-wiki]]
- [[hermes/sessoes/2026-06-18-recuperacao-e-wiki.md|2026-06-18-recuperacao-e-wiki]]
- [[hermes-config/notes/arquitetura-vps.md|arquitetura-vps]]
- [[infra-vps/gotchas/docker-swarm-ipvs-overlay.md|docker-swarm-ipvs-overlay]]
- [[infra-vps/gotchas/ipvs-table-vazia.md|ipvs-table-vazia]]

