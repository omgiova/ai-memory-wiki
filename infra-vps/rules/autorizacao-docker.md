---
type: rule
tags: [docker, infra-vps, projetos, rule, traefik, wiki]
title: Regra: Autorização Obrigatória Para Comandos Docker Swarm
description: 1. NUNCA executar docker service update sem autorização explícita do usuário - Principalmente: --network-rm, --network-add, --force, scale
timestamp: 2026-06-18T00:00:00+00:00
---


## Regra: Autorização Obrigatória Para Comandos Docker Swarm

**Data:** 2026-06-18

### Regras

1. **NUNCA** executar `docker service update` sem autorização explícita do usuário
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

5. **Sempre** salvar conhecimento na wiki markdown, não na memory() do Hermes
   - Memory() = cache de sessão (2.200 chars)
   - Wiki = fonte da verdade (ilimitada, versionada, portável)

### Penalidade por violação
Perda de confiança do usuário + horas de troubleshooting desnecessário. Já aconteceu.

## 📂 Navegação
[[infra-vps/rules/INDEX.md|📂 Voltar para rules]]

## 🔗 Conexões entre projetos
- [[hermes/decisions/memoria-wiki-markdown.md|memoria-wiki-markdown]]
- [[hermes/rules/conhecimento-na-wiki.md|conhecimento-na-wiki]]
- [[hermes/sessoes/2026-06-18-recuperacao-e-wiki.md|2026-06-18-recuperacao-e-wiki]]
- [[hermes/todo/proximos-passos.md|proximos-passos]]
- [[infra-vps/decisions/arquitetura-rede-swarm.md|arquitetura-rede-swarm]]
- [[infra-vps/gotchas/docker-swarm-ipvs-overlay.md|docker-swarm-ipvs-overlay]]

