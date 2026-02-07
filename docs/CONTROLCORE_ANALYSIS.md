# ControlCore Architecture Analysis

*Deep-dive into the original project to inform Forecaster's design*

## Executive Summary

After reviewing the [ControlCore repository](https://github.com/excessus1/controlcore_pub), the architecture is **more sophisticated than initially understood**. It's a custom-built Text-to-SQL pipeline with:

- **Two FastAPI services** (not a single app)
- **Sentence Transformers** for embeddings (not Ollama)
- **Custom prompt engineering** for SQL generation
- **Multi-model orchestration** (coding model + general model)

This document captures the exact architecture so we can make informed decisions about Forecaster.

---

## ControlCore Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           USER (Browser)                                     │
│                    Next.js Frontend + MQTT WebSocket                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SAURON (Data Machine)                                │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ Sauron API (FastAPI, port 8000)                                       │  │
│  │ - /chat endpoint receives user questions                              │  │
│  │ - Retrieves schema context via pgvector similarity search             │  │
│  │ - Orchestrates calls to Gandalf API                                   │  │
│  │ - Executes generated SQL against appropriate database                 │  │
│  │ - Returns formatted response                                          │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ PostgreSQL + pgvector                                                 │  │
│  │ - schema_embeddings table (768-dim vectors)                           │  │
│  │ - openweather_historical database                                     │  │
│  │ - openweather_forecast database                                       │  │
│  │ - controlcore database                                                │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         GANDALF (AI Machine)                                 │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ Gandalf API (FastAPI, port 9001)                                      │  │
│  │ - /rephrase: Cleans user questions (llama3)                           │  │
│  │ - /generate-sql: Creates SQL queries (deepseek-coder:6.7b)            │  │
│  │ - /analyze: Summarizes results (llama3)                               │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ Ollama (port 11434)                                                   │  │
│  │ - deepseek-coder:6.7b (SQL generation)                                │  │
│  │ - llama3 (rephrasing + summarization)                                 │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Chat Flow (Step by Step)

```
1. User types: "How much did it rain last Thursday in Fincastle?"
                                    │
                                    ▼
2. Sauron API (/chat) receives the question
   - Collects table schema metadata
   - Calls Gandalf /rephrase to clean the question
                                    │
                                    ▼
3. Sauron retrieves schema context from pgvector
   - Encodes question using all-mpnet-base-v2
   - SELECT ... FROM schema_embeddings ORDER BY embedding <-> vector LIMIT 5
   - Returns matching tables/columns/descriptions
                                    │
                                    ▼
4. Sauron calls Gandalf /generate-sql
   - Sends: question + schema + context hints
   - Gandalf uses deepseek-coder:6.7b to generate SQL
   - Returns: SQL query
                                    │
                                    ▼
5. Sauron validates and executes SQL
   - Strips fake schema prefixes
   - Validates table names exist
   - Picks correct database based on query content
   - Executes SQL, gets rows
                                    │
                                    ▼
6. Sauron calls Gandalf /analyze
   - Sends: original question + SQL + result rows
   - Gandalf uses llama3 to summarize
   - Returns: Markdown summary with chart recommendations
                                    │
                                    ▼
7. Response returned to frontend
```

---

## Key Components Deep Dive

### 1. Schema Embeddings Table

```sql
CREATE TABLE schema_embeddings (
  id serial PRIMARY KEY,
  table_name text,
  column_name text,
  content text,
  embedding vector(768),
  source_file text,
  entry_type text  -- alias | column | prompt | sql | description
);
```

**Entry types stored:**
- `description` - Table descriptions
- `column` - Column descriptions
- `alias` - Natural language aliases ("rain history" → daily_summary_data)
- `prompt` - Example questions users might ask
- `sql` - Example SQL queries (for few-shot learning)

### 2. Schema JSON Files

```json
{
  "database": "openweather_historical",
  "tables": [
    {
      "table": "daily_summary_data",
      "description": "Daily summaries including precipitation, humidity, temperature...",
      "columns": {
        "date": "UNIX timestamp representing the date.",
        "precipitation_total": "Total precipitation in mm for the day.",
        ...
      },
      "aliases": [
        "daily weather summary",
        "rain history",
        "yesterday rain"
      ],
      "example_prompts": [
        "How much did it rain last Thursday?",
        "What was the hottest day last month?"
      ]
    }
  ]
}
```

### 3. Embedding Model

**Model:** `all-mpnet-base-v2` (Sentence Transformers)
**Dimensions:** 768
**Library:** sentence-transformers (Python)

This is **NOT** an Ollama model - it's loaded directly via Python.

### 4. LLM Models

| Task | Model | Purpose |
|------|-------|---------|
| Question cleanup | llama3 | Simplify and clarify user questions |
| SQL generation | deepseek-coder:6.7b | Generate PostgreSQL queries |
| Result summarization | llama3 | Create natural language response |

### 5. Prompt Engineering

**SQL Generation Prompt:**
```
You are an AI assistant that generates Postgres SQL queries for weather and
environmental databases. The databases use PostgreSQL syntax.

Schema:
```json
{schema}
```

Hints:
```json
{context_from_vector_search}
```

Each listed database is separate. Use table names directly without
prefixing them with the database name.

User question:
{question} postgres

Only return a valid Postgres SQL query. Do not explain it.
Respond with the Postgres SQL statement only—no commentary, no Markdown fences.
```

---

## Technology Stack Comparison

| Component | ControlCore | Forecaster (Original Plan) | Match? |
|-----------|-------------|---------------------------|--------|
| Frontend | Next.js (custom) | AnythingLLM | ❌ Different |
| Backend | FastAPI (2 services) | AnythingLLM agents | ❌ Different |
| Vector DB | pgvector in PostgreSQL | pgvector or Qdrant | ✅ Partial |
| Embedding Model | all-mpnet-base-v2 (768d) | nomic-embed-text (768d) | ⚠️ Similar |
| SQL Model | deepseek-coder:6.7b | Mistral 7B | ⚠️ Different |
| Summary Model | llama3 | Mistral 7B | ⚠️ Different |
| Embedding Library | sentence-transformers | Ollama | ❌ Different |

---

## Critical Insights

### What Makes ControlCore Work

1. **Schema-aware embeddings** - They embed descriptions/aliases, not raw data
2. **Multi-model approach** - Coding model for SQL, general model for language
3. **Hint injection** - Vector search results guide the SQL model
4. **Validation layer** - SQL is parsed and validated before execution
5. **Database routing** - Question content determines which DB to query

### Why AnythingLLM May Not Be Ideal

1. **Document-focused** - Designed for PDFs/docs, not structured data
2. **No SQL execution** - Would need custom agent development
3. **Single model** - Doesn't easily support multi-model workflows
4. **Black box** - Less control over prompt engineering

### Better Options for Matching ControlCore

| Option | Effort | Fidelity | Notes |
|--------|--------|----------|-------|
| **A. Clone ControlCore** | Low | 100% | Fork and adapt his code directly |
| **B. Rebuild with FastAPI** | Medium | 95% | Same architecture, our infrastructure |
| **C. Dify workflows** | Medium | 70% | Visual builder, but less control |
| **D. AnythingLLM + custom agents** | High | 50% | Fighting against the tool |

---

## Recommendation

**Option A (Clone + Adapt)** is the fastest path to matching ControlCore:

1. Fork his repository
2. Update `.env` for our infrastructure (Jarvis for Ollama, Banner for Postgres)
3. Create our own schema JSON files for test data
4. Deploy the two FastAPI services
5. Point frontend to our endpoints

**This gives your friend a deployment guide that mirrors his exact setup.**

If you prefer a more turnkey solution (Option C/D), we should document clearly:
- What we're doing differently
- Why (simplicity, existing tools, etc.)
- Trade-offs in capability

---

## Questions to Resolve

1. **Do you want to match his architecture exactly** (fork/rebuild)?
2. **Or use pre-built tools** (AnythingLLM/Dify) with documented deviations?
3. **Which LLM models** does Jarvis currently have installed?
4. **Sample data** - create weather test data or use another domain?

---

*Generated: 2026-02-06*
*Source: https://github.com/excessus1/controlcore_pub*
