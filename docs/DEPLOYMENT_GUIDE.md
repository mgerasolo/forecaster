# Middle Earth Forecaster - Deployment Guide

*A local RAG system for natural language queries against structured data*

## Overview

This guide documents how to deploy a ChatGPT-like interface that queries databases using natural language, powered entirely by local LLMs (zero cloud cost).

**Based on:** [ControlCore](https://github.com/excessus1/controlcore_pub) by excessus1

---

## Architecture Concept

```
┌──────────────────────────────────────────────────────────────────┐
│                        USER (Browser)                            │
│                   ChatGPT-like chat interface                    │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                     CHAT FRONTEND                                │
│  (Open WebUI, AnythingLLM, or custom Next.js)                   │
│  - Accepts natural language questions                            │
│  - Forwards to backend API                                       │
│  - Displays formatted responses                                  │
└──────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                     SAURON API (Orchestrator)                    │
│  FastAPI service that:                                           │
│  1. Receives user question                                       │
│  2. Searches vector DB for relevant schema hints                 │
│  3. Calls Gandalf to generate SQL                                │
│  4. Executes SQL against database                                │
│  5. Calls Gandalf to summarize results                           │
│  6. Returns natural language answer                              │
└──────────────────────────────────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
┌──────────────────┐  ┌──────────────┐  ┌──────────────────┐
│   GANDALF API    │  │  PostgreSQL  │  │   Vector Search  │
│  (LLM Wrapper)   │  │  + pgvector  │  │  (pgvector or    │
│                  │  │              │  │   separate DB)   │
│  /rephrase       │  │ Your actual  │  │                  │
│  /generate-sql   │  │ data tables  │  │ Schema embeddings│
│  /analyze        │  │              │  │                  │
└──────────────────┘  └──────────────┘  └──────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────┐
│                     LOCAL LLM INFERENCE                          │
│  (Ollama, LM Studio, or similar)                                 │
│                                                                  │
│  Models needed:                                                  │
│  - Code/SQL model (deepseek-coder, devstral, qwen-coder)        │
│  - General model (llama3, mistral, qwen)                        │
│  - Embedding model (nomic-embed-text, bge-large)                │
└──────────────────────────────────────────────────────────────────┘
```

---

## Key Concepts

### 1. Schema Embeddings (The Secret Sauce)

The system doesn't embed your actual data - it embeds **descriptions of your schema**:

```json
{
  "table": "daily_summary_data",
  "description": "Daily weather summaries including precipitation and temperature",
  "columns": {
    "precipitation_total": "Total precipitation in mm for the day",
    "temperature_max": "Maximum temperature observed"
  },
  "aliases": ["rain history", "daily weather", "yesterday rain"]
}
```

When a user asks "How much did it rain yesterday?", the system:
1. Embeds the question → vector
2. Searches schema_embeddings for similar vectors
3. Finds "rain history" alias → `daily_summary_data` table
4. Provides this context to the SQL generation model

### 2. Multi-Model Pipeline

Different models excel at different tasks:

| Task | Model Type | Why |
|------|------------|-----|
| Question cleanup | General LLM | Understands natural language nuance |
| SQL generation | Code-focused LLM | Trained on code syntax |
| Result summarization | General LLM | Good at explanations |
| Embeddings | Embedding model | Optimized for semantic similarity |

### 3. Gandalf = Prompt Engineering Layer

Gandalf isn't an LLM - it's a service that builds prompts and calls LLMs:

```python
# Simplified Gandalf logic
@app.post("/generate-sql")
async def generate_sql(question: str, schema: str, context: list):
    prompt = f"""You are an AI that generates PostgreSQL queries.

    Schema: {schema}
    Context hints: {context}

    Question: {question}

    Return only the SQL query, no explanation."""

    return await call_llm(prompt, model="code-model")
```

---

## Deployment Steps (Conceptual)

### Step 1: Set Up Local LLM Inference

**Options:**
- **Ollama** - Easy CLI-based model management
- **LM Studio** - GUI-based, good for Windows
- **LocalAI** - OpenAI-compatible API server

**Required models:**
```bash
# If using Ollama
ollama pull deepseek-coder:6.7b   # SQL generation
ollama pull llama3:8b             # General tasks
ollama pull nomic-embed-text      # Embeddings
```

**Or use an LLM proxy** (like LiteLLM) to unify access to multiple model sources.

### Step 2: Deploy PostgreSQL with pgvector

```yaml
# docker-compose.yml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_USER: forecaster
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: forecaster
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
```

Create the schema embeddings table:
```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE schema_embeddings (
  id SERIAL PRIMARY KEY,
  table_name TEXT,
  column_name TEXT,
  content TEXT,
  embedding VECTOR(768),  -- Match your embedding model dimensions
  source_file TEXT,
  entry_type TEXT  -- 'alias', 'column', 'description', 'prompt'
);

CREATE INDEX ON schema_embeddings
  USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

### Step 3: Create Schema Description Files

Document your database schema in JSON:

```json
{
  "database": "your_database",
  "tables": [
    {
      "table": "sensors",
      "description": "IoT sensor readings from field devices",
      "columns": {
        "sensor_id": "Unique identifier for the sensor",
        "temperature": "Temperature reading in Celsius",
        "humidity": "Relative humidity percentage",
        "timestamp": "When the reading was taken"
      },
      "aliases": [
        "sensor data",
        "field readings",
        "temperature history"
      ],
      "example_prompts": [
        "What was the temperature at noon?",
        "Show humidity readings for the last hour"
      ]
    }
  ]
}
```

### Step 4: Load Schema Embeddings

```python
from sentence_transformers import SentenceTransformer
import psycopg2

model = SentenceTransformer('all-mpnet-base-v2')  # or your chosen model

def embed_and_store(table_name, content, entry_type):
    vector = model.encode(content).tolist()
    cursor.execute("""
        INSERT INTO schema_embeddings (table_name, content, embedding, entry_type)
        VALUES (%s, %s, %s, %s)
    """, (table_name, content, vector, entry_type))
```

### Step 5: Deploy Gandalf API

FastAPI service with three endpoints:
- `POST /rephrase` - Clean up user questions
- `POST /generate-sql` - Generate SQL from question + context
- `POST /analyze` - Summarize results in natural language

Each endpoint builds a prompt and calls your LLM.

### Step 6: Deploy Sauron API

FastAPI service that orchestrates the flow:
1. Receive question from frontend
2. Embed question, search schema_embeddings
3. Call Gandalf /rephrase
4. Call Gandalf /generate-sql with schema context
5. Execute SQL against your database
6. Call Gandalf /analyze with results
7. Return response to frontend

### Step 7: Deploy Chat Frontend

**Options:**
- **Open WebUI** - Best for ChatGPT-like experience, supports pipelines
- **AnythingLLM** - Good for document-heavy use cases
- **Custom** - Build with Next.js/React if you need full control

Configure the frontend to send messages to your Sauron API endpoint.

---

## NLF-Specific Implementation

### Our Infrastructure Mapping

| Component | Host | Port | Service |
|-----------|------|------|---------|
| Open WebUI | Banner (10.0.0.33) | 3351 | Chat frontend |
| Sauron API | Banner | 3352 | Orchestrator |
| PostgreSQL + pgvector | Banner | 3353 | Data + vectors |
| Gandalf API | Banner | 3354 | LLM wrapper |
| LLM Inference | Jarvis via LiteLLM | 2764 | Model serving |

### Our Model Selection

| Task | Model | Via |
|------|-------|-----|
| SQL Generation | `jarvis-devstral` (24B code model) | LiteLLM |
| Rephrasing | `jarvis-llama31` (8B) | LiteLLM |
| Summarization | `jarvis-llama31` (8B) | LiteLLM |
| Embeddings | `jarvis-embed` (nomic-embed-text) | LiteLLM |

### Our Deviation from ControlCore

| Aspect | ControlCore | Middle Earth Forecaster | Reason |
|--------|-------------|------------------------|--------|
| Frontend | Custom Next.js | Open WebUI | Faster deployment, better UX |
| LLM Access | Direct Ollama | LiteLLM proxy | Unified API, model flexibility |
| Deployment | 2 machines | Single host (Banner) | Simpler for demo |

---

## Testing Your Deployment

Once deployed, test with queries like:

```
User: "What was the lowest temperature recorded last week?"

Expected flow:
1. Question embedded → finds "temperature" column context
2. Gandalf generates: SELECT MIN(temperature), timestamp FROM readings
                      WHERE timestamp > NOW() - INTERVAL '7 days'
3. SQL executed → returns row
4. Gandalf summarizes: "The lowest temperature was 12.3°C, recorded on
                        Tuesday at 3:47 AM."
```

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| "Unknown table" errors | SQL model hallucinating tables | Add more aliases to schema JSON |
| Poor SQL quality | Insufficient context | Add example_prompts and example_sql to schema |
| Slow responses | Large model, no GPU | Use smaller models or add GPU |
| Wrong database queried | Ambiguous question | Improve database routing logic in Sauron |

---

## Resources

- [ControlCore Source](https://github.com/excessus1/controlcore_pub)
- [pgvector Documentation](https://github.com/pgvector/pgvector)
- [Open WebUI Pipelines](https://docs.openwebui.com/pipelines/)
- [LiteLLM Docs](https://docs.litellm.ai/)

---

*Document version: 1.0*
*Last updated: 2026-02-06*
