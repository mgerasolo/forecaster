# Forecaster - Local RAG System Project Plan

## Project Overview

**Project Name:** Forecaster
**Challenge:** Build a local RAG system that matches/exceeds ControlCore's capabilities with zero cloud API costs
**Inspiration:** [ControlCore](https://github.com/excessus1/controlcore_pub) - friend's weather/irrigation RAG system

### The Challenge

Build a system where:
- End users ask questions in plain English (ChatGPT-like interface)
- System queries structured data (weather, sensors, etc.)
- Local LLM provides natural language answers
- **Zero cloud costs** - no API tokens, everything runs locally

Example queries:
- "What was the lowest data point across all series? Give me the location, date, time, and temperature."
- "How many days did it rain in September in Roanoke, Virginia?"

---

## ControlCore Analysis

### What They Built

| Component | Implementation |
|-----------|----------------|
| Vector DB | PostgreSQL + pgvector |
| Local LLM | Ollama (Llama3, Deepseek) |
| Frontend | Next.js with AI chat interface |
| Backend | FastAPI (Python) |
| Architecture | Two machines: Sauron (data) + Gandalf (AI) |

### How It Works

```
User Query → Vectorize → Find Similar Schema Elements → Generate SQL → Execute → Natural Language Response
```

Key insight: They embed **database schema** into vectors, not raw data. When user asks about temperature, system finds relevant columns/tables via similarity search, then generates appropriate SQL.

### Their Tech Stack
- TypeScript (54.3%) - Next.js frontend with MQTT WebSocket
- Python (30.8%) - FastAPI servers
- PostgreSQL with PL/pgSQL (11.5%) - pgvector for embeddings
- Arduino/ESP32 via MQTT for IoT sensors

---

## NLF Infrastructure Mapping

| ControlCore | NLF Equivalent | IP | Role |
|-------------|----------------|-----|------|
| Gandalf (AI) | Jarvis | 10.0.0.40 | Windows 11, Ollama inference |
| Sauron (Data) | Banner | 10.0.0.33 | Dev containers, PostgreSQL |
| - | Hulk | 10.0.0.32 | Production containers |

---

## Research Findings (2026)

### Best Embedding Models for Local RAG

| Model | Parameters | Accuracy | Latency | Recommendation |
|-------|------------|----------|---------|----------------|
| **e5-small** | 118M | 100% Top-5 | 16ms | **Best overall** - fastest + most accurate |
| e5-base-instruct | 33.5M | 100% Top-5 | ~20ms | Good alternative |
| nomic-embed-text | 274MB | Good | Fast | Available in Ollama |
| all-MiniLM-L6-v2 | - | 56% Top-5 | 28ms | **Avoid** - outdated |

**Key insight:** Smaller models beat larger ones. e5-small (118M params) outperforms 8B+ models.

### Best LLMs for RAG

| Model | Context | Why It Works | Ollama Command |
|-------|---------|--------------|----------------|
| **Mistral 7B** | 8K-32K | Best instruction-following, doesn't hallucinate | `ollama pull mistral` |
| Llama 3 8B | 8K | Strong reasoning | `ollama pull llama3` |
| Gemma 3 4B | - | Fast on CPU | `ollama pull gemma3:4b` |

**Requirements for RAG:**
- Minimum 8K context window (RAG consumes 2K-8K tokens for retrieved context)
- Strong instruction following ("answer only from provided documents")
- Low hallucination rate

### Vector Database Options

| Database | Best For | Setup Complexity | Notes |
|----------|----------|------------------|-------|
| **Chroma DB** | Beginners, <1M vectors | Easy (Docker) | Used by Open WebUI |
| FAISS | Maximum control | Medium (Python) | No server needed |
| **Qdrant** | Production, hybrid search | Medium (Docker) | Best features |
| pgvector | PostgreSQL integration | Medium | ControlCore uses this |
| Milvus | Enterprise scale | Complex | Overkill for this |

---

## Tool Selection

### Comparison Matrix

| Tool | What It Is | User-Facing? | RAG Score | Text-to-SQL | Best For |
|------|------------|--------------|-----------|-------------|----------|
| **AnythingLLM** | Document RAG specialist | ✅ Yes | 10/10 | Via Agents | **This challenge** |
| **Open WebUI** | ChatGPT clone | ✅ Yes | 9/10 | Via Functions | General chat |
| **Dify** | AI workflow builder | ❌ Backend | 9/10 | ✅ Visual | Complex pipelines |
| LobeChat | ChatGPT clone | ✅ Yes | 6/10 | Via MCP | Already deployed |
| PrivateGPT | Document Q&A | ✅ Yes | 9/10 | Limited | Privacy-first |

### Why AnythingLLM for Forecaster

1. **Purpose-built for RAG** - literally "chat with your data"
2. **Agents** - can chain actions, call external APIs, execute code
3. **Workspaces** - organize knowledge by topic (weather, sensors, etc.)
4. **Desktop app** - can demo without server setup
5. **10/10 RAG score** - highest in our audit

### Why Also Deploy Dify

1. **Visual workflow builder** - show friend how to construct Text-to-SQL logic
2. **REST API** - can be called from AnythingLLM agents
3. **n8n for AI** - familiar paradigm for workflow automation
4. **Hybrid approach** - Dify handles logic, AnythingLLM handles UI

---

## Architecture

### Option A: AnythingLLM Only (Simpler)

```
┌─────────────────────────────────────────────────────────────────┐
│                     USER (Browser)                              │
│              http://forecaster.nextlevelguild.com               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   BANNER (10.0.0.33)                            │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ AnythingLLM (port 3351)                                   │ │
│  │ - Chat interface                                          │ │
│  │ - Workspace: "Weather Data"                               │ │
│  │ - Agent: Text-to-SQL                                      │ │
│  │ - Vector DB: Chroma (built-in) or Qdrant                  │ │
│  └───────────────────────────────────────────────────────────┘ │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ PostgreSQL + pgvector (port 5432)                         │ │
│  │ - Weather data tables                                     │ │
│  │ - Schema embeddings                                       │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   JARVIS (10.0.0.40)                            │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Ollama (port 11434)                                       │ │
│  │ - mistral:7b (LLM for generation)                         │ │
│  │ - nomic-embed-text (embeddings)                           │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Option B: AnythingLLM + Dify (Full Demo)

```
┌─────────────────────────────────────────────────────────────────┐
│                     USER (Browser)                              │
└─────────────────────────────────────────────────────────────────┘
          │                                    │
          ▼                                    ▼
┌──────────────────────┐          ┌──────────────────────┐
│ AnythingLLM (:3351)  │          │ Dify (:3352)         │
│ - User chat UI       │          │ - Workflow builder   │
│ - Calls Dify API     │          │ - Text-to-SQL logic  │
│   for complex queries│          │ - Exposes REST API   │
└──────────────────────┘          └──────────────────────┘
          │                                    │
          └──────────────┬─────────────────────┘
                         ▼
              ┌─────────────────────┐
              │ PostgreSQL+pgvector │
              │ (Banner :5432)      │
              └─────────────────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │ Ollama (Jarvis)     │
              │ :11434              │
              └─────────────────────┘
```

---

## Deployment Plan

### Phase 1: Infrastructure Setup

| Task | Host | Port | Notes |
|------|------|------|-------|
| Deploy AnythingLLM | Banner | 3351 | Main user interface |
| Deploy Dify | Banner | 3352 | Workflow builder |
| Deploy PostgreSQL + pgvector | Banner | 3353 | Data storage |
| Deploy Qdrant (optional) | Banner | 3354 | Vector DB alternative |
| Configure Ollama models | Jarvis | 11434 | Already running |

### Phase 2: Traefik Routing

| Domain | Target | Middleware |
|--------|--------|------------|
| `forecaster.nextlevelguild.com` | Banner:3351 | internal-only |
| `forecaster-dify.nextlevelguild.com` | Banner:3352 | internal-only |

### Phase 3: Data & Configuration

1. Pull required Ollama models on Jarvis:
   ```bash
   ollama pull mistral
   ollama pull nomic-embed-text
   ```

2. Create sample weather dataset (or use ControlCore's schema)

3. Configure AnythingLLM:
   - LLM Provider: Ollama @ http://10.0.0.40:11434
   - Embedding Model: nomic-embed-text
   - Vector DB: Built-in Chroma or external Qdrant

4. Configure Dify:
   - LLM Provider: Ollama @ http://10.0.0.40:11434
   - Create Text-to-SQL workflow

### Phase 4: Text-to-SQL Agent

Create an agent in AnythingLLM that:
1. Receives natural language query
2. Retrieves relevant schema embeddings
3. Generates SQL query
4. Executes against PostgreSQL
5. Formats results as natural language

---

## Docker Compose Stacks

### Stack 1: AnythingLLM

```yaml
# ~/Infrastructure/stacks/ai-tools/anythingllm/docker-compose.yml
services:
  anythingllm:
    image: mintplexlabs/anythingllm
    container_name: anythingllm
    ports:
      - "3351:3001"
    volumes:
      - anythingllm-data:/app/server/storage
    environment:
      - STORAGE_DIR=/app/server/storage
      - LLM_PROVIDER=ollama
      - OLLAMA_BASE_PATH=http://10.0.0.40:11434
      - EMBEDDING_ENGINE=ollama
      - EMBEDDING_MODEL_PREF=nomic-embed-text
    restart: unless-stopped

volumes:
  anythingllm-data:
```

### Stack 2: Dify

```yaml
# ~/Infrastructure/stacks/ai-tools/dify/docker-compose.yml
# Note: Dify requires multiple services - use their official compose
# https://github.com/langgenius/dify/blob/main/docker/docker-compose.yaml
```

### Stack 3: PostgreSQL + pgvector

```yaml
# ~/Infrastructure/stacks/ai-tools/forecaster-db/docker-compose.yml
services:
  postgres:
    image: pgvector/pgvector:pg16
    container_name: forecaster-db
    ports:
      - "3353:5432"
    environment:
      - POSTGRES_USER=forecaster
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=forecaster
    volumes:
      - forecaster-pgdata:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  forecaster-pgdata:
```

---

## Success Criteria

### Minimum Viable Demo

- [ ] User can type natural language question in AnythingLLM
- [ ] System retrieves relevant data from PostgreSQL
- [ ] Local LLM (Mistral on Jarvis) generates answer
- [ ] Zero external API calls (verify with network monitor)

### Full Demo

- [ ] All minimum criteria met
- [ ] Dify workflow visualizes the Text-to-SQL pipeline
- [ ] Qdrant provides fast vector similarity search
- [ ] Multiple workspaces for different data types
- [ ] Traefik routing with proper domains

### Comparison Points vs ControlCore

| Feature | ControlCore | Forecaster |
|---------|-------------|------------|
| User interface | Custom Next.js | AnythingLLM |
| Vector DB | pgvector | pgvector + Qdrant |
| LLM | Llama3/Deepseek | Mistral 7B |
| Workflow visibility | Code only | Dify visual builder |
| Setup time | Hours (custom) | ~1 hour (pre-built) |
| Customization | Full control | Agent configuration |

---

## Resources

### Documentation
- [AnythingLLM Docs](https://docs.useanything.com/)
- [Dify Docs](https://docs.dify.ai/)
- [pgvector GitHub](https://github.com/pgvector/pgvector)
- [Qdrant Docs](https://qdrant.tech/documentation/)
- [Ollama API](https://github.com/ollama/ollama/blob/main/docs/api.md)

### NLF Resources
- AI Tools Comparison: https://dashboard.nextlevelfoundry.com/ai-tools-comparison.html
- Ollama on Jarvis: http://10.0.0.40:11434

### ControlCore Reference
- GitHub: https://github.com/excessus1/controlcore_pub
- Architecture: PostgreSQL + pgvector + Ollama + FastAPI + Next.js

---

## Next Steps

1. **Create Forecaster app directory** in Infrastructure repo
2. **Deploy AnythingLLM** on Banner (port 3351)
3. **Deploy Dify** on Banner (port 3352)
4. **Deploy PostgreSQL + pgvector** on Banner (port 3353)
5. **Configure Traefik** routing for both tools
6. **Pull Ollama models** on Jarvis (mistral, nomic-embed-text)
7. **Create sample weather dataset** for testing
8. **Build Text-to-SQL agent** in AnythingLLM
9. **Create visual workflow** in Dify
10. **Demo to friend** - show both approaches

---

*Generated: 2026-02-06*
*Conversation: conv-20260204-173002*
*Challenge: Local RAG with zero cloud costs*
