# Karpathy's Agentic Engineering Finally Has Proper Tooling

**Build by Google, explained as step-by-step guide.**

Fonte: https://blog.dailydoseofds.com/p/karpathys-agentic-engineering-finally  
Autor: Akshay Pachaar (@akshay_pachaar)  
Data: 2026-06-28

---

Karpathy defined agentic engineering at Sequoia Ascent 2026 as "the discipline that separates production-grade agent work from vibe coding." The core competencies he identified were spec design, eval loops, and security oversight.

However, practical agentic engineering has lacked unified tooling, requiring developers to juggle multiple interfaces: code editors, terminal scaffolding, browsers for testing, cloud consoles for deployment, and separate eval frameworks.

## The Solution: Google's Agents CLI

"Google's Agents CLI" now provides an integrated solution covering the entire workflow in a single environment for scaffolding, evaluating, and deploying ADK agents. The tool injects seven bundled skills into coding agents, teaching them ADK patterns, eval structures, and deployment targets.

![Diagrama de fluxo de trabalho do Google Agents CLI](https://substackcdn.com/image/fetch/$s_!I-Xr!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F838132fb-d50c-458c-89a2-3b976bd20b34_2752x757.jpeg)

![Interface do Google Agents CLI](https://substackcdn.com/image/fetch/$s_!DoKq!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fc602df15-278f-48d1-8cda-9d3c82ea596b_1190x740.png)

## Step-by-Step Implementation

### Step 1: Install Agents CLI

```
uvx google-agents-cli setup
```

This installation distributes skills across all coding agents simultaneously, including Claude Code, Cursor, Codex, and others.

![Skills injetadas nos coding agents](https://substackcdn.com/image/fetch/$s_!5qTM!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F83dacae6-52bd-469f-8f87-8ee9a14ceeb7_3033x2247.png)

![Compatibilidade com coding agents (Claude Code, Cursor, Codex)](https://substackcdn.com/image/fetch/$s_!lN6d!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fee8dea4a-566c-4093-88bf-f412b5a02131_2963x1281.png)

### Step 2: Build the RAG Agent

Developers provide natural language instructions to scaffold a complete RAG project. The coding agent activates its injected ADK skills and automatically:
- Creates project structure using the agentic_rag template
- Configures Vector Search as the datastore
- Implements citation support
- Ingests sample content
- Runs smoke tests

### Step 3: Test Locally

The coding agent launches an interactive web UI for real-world testing. The system should correctly retrieve cited answers and refuse to answer questions outside the knowledge base.

### Step 4: Evaluate Before Deploying

A critical step many tutorials omit, developers can request comprehensive test coverage. The agent generates 20 scenarios across four categories:
- Six for correct retrieval
- Five for insufficient context
- Five for multi-hop reasoning
- Four for citation accuracy

Karpathy noted that while 89% of teams maintain observability, only 52% implement evaluation frameworks.

![Resultados da avaliação — 20 cenários gerados](https://substackcdn.com/image/fetch/$s_!iYDR!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F8cac557a-406b-4090-b72c-a8437047e587_3268x1744.png)

![Métricas de performance](https://substackcdn.com/image/fetch/$s_!3lGz!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F47dbb7bc-d5e8-46f8-89f4-94930c35e9b1_3290x2044.png)

### Step 5: Deploy to Agent Runtime

```
Deploy this to Agent Runtime in us-central1.
```

The process automatically adds deployment configuration and provisions cloud infrastructure within 2-3 minutes, with Cloud Trace enabled by default.

### Step 6: Register to Gemini Enterprise

Registration makes the agent discoverable across the organization through the Gemini Enterprise platform, with built-in IAM controls and observability dashboards.

![Gemini Enterprise — descoberta e IAM controls](https://substackcdn.com/image/fetch/$s_!HeWT!,w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fee653e3c-9e91-48a0-9a5a-9a77751e7c8f_2752x1302.jpeg)

---

**Resources:**
- GitHub: [Agents CLI](https://fandf.co/44tlJqK)
- [ADK documentation](https://fandf.co/44yZhfO)
- [Agent Platform](https://fandf.co/4widxWp)
