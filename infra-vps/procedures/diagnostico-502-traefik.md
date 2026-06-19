---
type: procedure
tags: [docker, infra-vps, n8n, procedure, projetos, traefik]
title: Procedimento: Diagnosticar 502 Bad Gateway no Traefik
description: 1. Testar o endpoint direto via Traefik bash curl -sI https://projetos-n8n-editor.igkokh.easypanel.host/ Se 200 → não é problema de rota. Se 502 → continuar.
resource: https://projetos-n8n-editor.igkokh.easypanel.host/
timestamp: 2026-06-18T00:00:00+00:00
---


## Procedimento: Diagnosticar 502 Bad Gateway no Traefik

**Data:** 2026-06-18

### Passo a passo

1. **Testar o endpoint direto via Traefik**
   ```bash
   curl -sI https://projetos-n8n-editor.igkokh.easypanel.host/
   ```
   Se 200 → não é problema de rota. Se 502 → continuar.

2. **Testar conectividade de dentro do Traefik**
   ```bash
   docker exec easypanel-traefik wget -qO- http://projetos_n8n_editor:5678/
   ```
   - Se "Host is unreachable" → overlay ou IPVS
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

### Quando chamar o restart do Docker
- IPVS table vazia + DNS resolve + containers rodando
- Apenas `systemctl restart docker` — não perde dados

## 📂 Navegação
[[infra-vps/procedures/INDEX.md|📂 Voltar para procedures]]

## 🔗 Conexões entre projetos
- [[hermes/decisions/memoria-wiki-markdown.md|memoria-wiki-markdown]]
- [[hermes/procedures/implementar-memoria-arquivos.md|implementar-memoria-arquivos]]
- [[hermes/rules/conhecimento-na-wiki.md|conhecimento-na-wiki]]
- [[hermes/sessoes/2026-06-18-recuperacao-e-wiki.md|2026-06-18-recuperacao-e-wiki]]
- [[hermes-config/notes/arquitetura-vps.md|arquitetura-vps]]
- [[infra-vps/decisions/arquitetura-rede-swarm.md|arquitetura-rede-swarm]]

