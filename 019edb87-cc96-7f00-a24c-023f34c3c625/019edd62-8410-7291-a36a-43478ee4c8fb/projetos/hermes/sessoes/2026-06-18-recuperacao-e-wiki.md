---
tier: semantic
---
## Sessão: 2026-06-18 — Recuperação de Infra + Fundação da Wiki

### O que aconteceu
- Série de comandos Docker Swarm errados (network-rm, scale 0, --force) quebraram n8n e Node-RED
- IPVS table foi corrompida → DNS resolvia mas tráfego não chegava → 502
- Solução: `systemctl restart docker` repovoou IPVS
- Instalamos a estrutura de wiki no ai-memory: `projetos/infra-vps/` e `projetos/hermes/`

### O que foi criado na wiki
7 páginas em `projetos/infra-vps/` e `projetos/hermes/` (decisions, gotchas, procedures, rules)

### Pendente pra próxima sessão
1. Migrar conhecimento acumulado de semanas de uso pra wiki
2. Sincronizar wiki com GitHub (repo privado)
3. Conectar Obsidian no Windows ao repo
4. Revisar SOUL.md + criar AGENTS.md
5. Configurar auto-improve loop do ai-memory
6. Garantir handoff entre agentes (Hermes ↔ Codex etc.)

### 5 Pilares do objetivo final
1. Memória de longo prazo pra agentes
2. Markdown como fonte da verdade (versionado, legível, buscável)
3. Padrão LLM Wiki do Karpathy (Raw → Wiki → Schema)
4. Loop de auto-aprendizado (sessão → extração → memória)
5. Handoff entre agentes (trocar de ferramenta sem perder contexto)
