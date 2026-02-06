# CLAUDE.md

This file provides guidance to Claude Code when working with the forecaster codebase.

## Overview

**Forecaster** is a local RAG (Retrieval-Augmented Generation) system for querying structured data using natural language. Built to demonstrate zero-cloud-cost AI with local LLMs.

**Challenge:** Match/exceed ControlCore's capabilities (https://github.com/excessus1/controlcore_pub)

**Target Environment:** Banner (10.0.0.33)
**Port Block:** 3351-3354 (AnythingLLM=3351, Dify=3352, PostgreSQL=3353, Qdrant=3354)
**Domain:** forecaster.nextlevelguild.com

## Critical Rules

**Container Deployment:**
- **NEVER deploy containers to Stark or localhost** - Stark is a coding workstation only
- **Development containers → Banner (10.0.0.33)**
- **Production containers → Hulk (10.0.0.32)**
- This project targets: **Banner**

**URLs - Never localhost:**
- **NEVER use localhost or 127.0.0.1** - containers run on remote VMs
- **Preferred:** Fully qualified domain (requires Traefik setup)
- **Acceptable:** VM IP with port (e.g., `http://10.0.0.33:3351`)

```bash
# BEST - domain via Traefik
https://forecaster.nextlevelguild.com

# OK - direct IP to container host
http://10.0.0.33:3351

# WRONG - will not work (container not on your machine)
http://localhost:3351
http://127.0.0.1:3351
```

**Domain/Traefik Setup:**
- Traefik config: `~/Infrastructure/stacks/traefik/`
- Domain registry: `~/Infrastructure/DEPLOYMENTS.md`
- Standards: `/mnt/foundry_project/AppServices/Standards-v2/infrastructure/`

**SSH Access:**
- **ALWAYS use hostname, NEVER use IP address**
- SSH config handles port/user/key automatically

```bash
# CORRECT
ssh banner

# WRONG - will fail
ssh 10.0.0.33
```

## Technology Stack

| Purpose | Tool | Location |
|---------|------|----------|
| Chat Interface | AnythingLLM | Banner:3351 |
| Workflow Builder | Dify | Banner:3352 |
| Vector DB | PostgreSQL + pgvector | Banner:3353 |
| Vector DB (alt) | Qdrant | Banner:3354 |
| LLM Inference | Ollama (Mistral 7B) | Jarvis:11434 |
| Embeddings | nomic-embed-text | Jarvis (via Ollama) |

## Architecture

```
User → AnythingLLM (chat) → Ollama (Jarvis) → Text-to-SQL Agent
                                    ↓
                          PostgreSQL + pgvector (Banner)
                                    ↓
                          Natural language answer
```

## Project Plan

Full project plan: [docs/PROJECT_PLAN.md](docs/PROJECT_PLAN.md)

## Deployment Components

| Component | Image | Port | Purpose |
|-----------|-------|------|---------|
| AnythingLLM | mintplexlabs/anythingllm | 3351 | User-facing chat |
| Dify | langgenius/dify | 3352 | Visual workflow builder |
| PostgreSQL | pgvector/pgvector:pg16 | 3353 | Data + vector storage |
| Qdrant | qdrant/qdrant | 3354 | Fast vector search |

## Available Infrastructure Services

| Service | Endpoint / Location | Purpose |
|---------|---------------------|---------|
| Traefik | `~/Infrastructure/stacks/traefik/` | Reverse proxy, SSL, domain routing |
| Authentik | `auth.nextlevelfoundry.com` | SSO, OAuth, user management |
| LiteLLM | `http://10.0.0.27:2764` | AI model proxy (OpenAI, Anthropic, Ollama, etc.) |
| Loki + Grafana | Coulson (10.0.0.28) | Structured logging |
| Prometheus + Grafana | Coulson (10.0.0.28) | Metrics and alerting |
| PostgreSQL | Per-service (bundled in stack) | Relational database |
| Redis | Per-service (bundled in stack) | Caching, queues |
| MinIO | Helicarrier | S3-compatible object storage |
| n8n | `http://10.0.0.27:2750` | Workflow automation, webhooks |
| Image Generation | `http://10.0.0.27:2750/webhook/generate-image` | AI image generation (Gemini, DALL-E, Flux) |
| Infisical / .env | `/mnt/foundry_project/AppServices/env/` | Secrets management |
| SMTP | Via AppServices config | Transactional email |

## Key Directories

```
forecaster/
├── .claude/                    # Baton context management
│   ├── CONVERSATION_HISTORY.md # All conversations TLDR
│   ├── BUGS.md                 # Discovered bugs (tagged by conv-id)
│   ├── DECISIONS.md            # Architecture decisions (tagged by conv-id)
│   └── conversations/          # Per-conversation summaries
├── .github/
│   └── ISSUE_TEMPLATE/         # GitHub issue templates
├── src/                        # Application source code
├── docs/                       # Documentation
└── scripts/                    # Utility scripts
```

## Common Commands

```bash
# Secrets (from Infrastructure)
source ~/Infrastructure/scripts/secrets.sh
appservices_get SECRET_KEY       # Shared platform secrets
ai_apps_get OPENAI_API_KEY      # AI service API keys

# Deployment (when ready)
# Use /deployment banner forecaster
```

## Claude Code Telemetry

Usage metrics are tracked via OpenTelemetry to Coulson's monitoring stack.

```bash
# Set project name for this session (auto-detects from git)
claude-telemetry-project

# Or set explicitly
claude-telemetry-project forecaster
```

Dashboard: https://grafana.ucontrolnetwork.com/d/claude-code-working

## Context Management (Baton Protocol)

This project uses structured context management for multi-conversation workflows.

### On Session Start
1. Check `.claude/CURRENT_CONVERSATION_ID`
2. Read `.claude/CONVERSATION_HISTORY.md` for overview
3. Read `.claude/conversations/{conv-id}/SUMMARY.md` for current work

### During Work
- Update SUMMARY.md after significant actions
- Append to BUGS.md when discovering bugs (tag with conv-id)
- Append to DECISIONS.md for architecture decisions (tag with conv-id)

### After Compaction
- IMMEDIATELY read CONVERSATION_HISTORY.md
- Read your conversation's SUMMARY.md
- Resume work with context restored

## Standardized Response Format

**MANDATORY:** All responses must use this format:

```markdown
**Title:**
- [Conversation title, max 60 chars]

**Request:**
- [Up to 120 char summary of request]

**Tasks:**
- ✅ [Owner] [Details...] Completed task
- ⬜ [Owner] [Status] [Details...] Pending task

**Summary:**
- Portfolio manager perspective: features, branding, cost, big picture
- Avoid deep technical specifics

**Next:**
- [Next immediate action or "None"]

**USER ACTION NEEDED:**
- [Actions requiring human decision]

**Context:**
- XX% used, YY% remaining
```

**Emoji Legend:**
- **Owner:** 🤖 Claude | 👨‍🔧 Human | 👤 Other
- **Status:** ⏳ Waiting | 🛑 Blocked | 🏳️ Ready | 💬 Discuss
- **Details:** 🔸 Required | 🔹 Optional | ⚠️ Concern | ∥ Parallel

## GitHub Integration

**Repository:** https://github.com/mgerasolo/forecaster
**Project Board:** https://github.com/users/mgerasolo/projects/X

**Label Taxonomy:**
| Category | Labels |
|----------|--------|
| Type | `type:bug`, `type:feature`, `type:enhancement`, `type:docs` |
| Priority | `priority:critical`, `priority:high`, `priority:medium`, `priority:low` |
| Area | `area:ui`, `area:api`, `area:database`, `area:auth` |
| Status | `status:active`, `status:soon`, `status:blocked`, `status:pending-approval`, `status:ai-ready` |

**At session start:** Check for `status:ai-ready` issues (pre-approved for autonomous work)

## Cross-Project Coordination

**Dependencies:**
- Infrastructure (nlf-infrastructure) - Deployment, secrets, monitoring

**Before breaking changes:**
1. Check dependent projects
2. Create issue with `breaking:next-release` label
3. Notify Infrastructure project

## Security Notes

- Never commit secrets or API keys
- Use Infisical or .env files from shared location
- All external API calls must go through authenticated endpoints

## Related Documentation

- Infrastructure: `~/Infrastructure/CLAUDE.md`
- AppServices Standards: `/mnt/foundry_project/AppServices/`
- Deployment Docs: `/mnt/foundry_project/Forge/deployments/banner/forecaster/`
