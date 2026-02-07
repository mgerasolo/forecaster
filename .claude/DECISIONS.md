# Architecture Decisions

Track architectural and technical decisions, tagged with conversation ID.

## Format

**Conv:** conv-YYYYMMDD-HHMMSS
**Decision:** Description
**Rationale:** Why this decision was made
**Date:** YYYY-MM-DD

---

## Decisions

### LLM Integration

**Conv:** conv-20260206
**Decision:** Use LiteLLM proxy instead of direct Ollama API
**Rationale:** LiteLLM provides OpenAI-compatible API that works with SQLCoder's expected prompt format. Allows switching models without code changes. Already deployed on Jarvis (10.0.0.27:2764).
**Date:** 2026-02-06

---

**Conv:** conv-20260206
**Decision:** Use SQLCoder-specific prompt format with `[QUESTION]...[/QUESTION]` and `[SQL]` markers
**Rationale:** SQLCoder was trained on this specific format. Using the correct prompt structure significantly improves SQL generation quality.
**Date:** 2026-02-06

---

### Embedding Model

**Conv:** conv-20260206
**Decision:** Use `all-mpnet-base-v2` (768 dimensions) instead of `nomic-embed-text`
**Rationale:** all-mpnet-base-v2 is available locally via sentence-transformers without API calls. Provides high-quality embeddings for schema similarity search. 768 dimensions fits pgvector well.
**Date:** 2026-02-06

---

**Conv:** conv-20260206
**Decision:** Load embedding model locally in Sauron container rather than via API
**Rationale:** Eliminates network latency for every embedding request. Model is ~400MB, acceptable for container size. Lazy-loaded on first request.
**Date:** 2026-02-06

---

### API Architecture

**Conv:** conv-20260206
**Decision:** Add OpenAI-compatible endpoints (`/v1/models`, `/v1/chat/completions`) to Sauron API
**Rationale:** Enables integration with Open WebUI and any OpenAI-compatible client. Wraps existing chat logic without duplication.
**Date:** 2026-02-06

---

**Conv:** conv-20260206
**Decision:** Create standalone `main_standalone.py` for Sauron rather than modifying original
**Rationale:** Preserves original ControlCore code. Simplifies deployment by removing MQTT and sensor dependencies. Single-file deployment is easier to debug.
**Date:** 2026-02-06

---

### Data Transparency

**Conv:** conv-20260206
**Decision:** Add query metadata to responses (location, date range requested, data available)
**Rationale:** Users need to understand when they're getting partial data. "Asked for Year 2026, only have Jan 15-20" prevents misinterpretation of incomplete results.
**Date:** 2026-02-06

---

**Conv:** conv-20260206
**Decision:** Extract metadata from SQL using regex patterns rather than LLM parsing
**Rationale:** Deterministic, fast, no additional API calls. Handles common SQL patterns (BETWEEN, EXTRACT YEAR, ILIKE). Falls back gracefully if pattern not matched.
**Date:** 2026-02-06

---

### Missing Data Handling

**Conv:** conv-20260206
**Decision:** Detect missing/NULL results and provide actionable suggestions
**Rationale:** Empty results are confusing. System now acknowledges missing data, offers to fetch from OpenWeather API, and suggests alternative queries (different dates or locations).
**Date:** 2026-02-06

---

### Deployment

**Conv:** conv-20260206
**Decision:** Use Docker Compose with separate containers for Gandalf, Sauron, and PostgreSQL
**Rationale:** Isolation, independent scaling, easier debugging. Gandalf and Sauron can be rebuilt without touching the database.
**Date:** 2026-02-06

---

**Conv:** conv-20260206
**Decision:** Use Cloudflare Tunnel for external access instead of direct port exposure
**Rationale:** No need to open firewall ports. SSL handled automatically. Integrates with existing Cloudflare Zero Trust setup.
**Date:** 2026-02-06

---

**Conv:** conv-20260206
**Decision:** Deploy to Banner (10.0.0.33) ports 3351-3354
**Rationale:** Per project requirements, development containers go to Banner. Port block reserved for Forecaster stack.
**Date:** 2026-02-06

---

### Database

**Conv:** conv-20260206
**Decision:** Use pgvector extension with 768-dimension vectors
**Rationale:** Matches all-mpnet-base-v2 output dimensions. IVFFlat index provides fast approximate nearest neighbor search. PostgreSQL 16 has improved vector performance.
**Date:** 2026-02-06

---

**Conv:** conv-20260206
**Decision:** Store schema embeddings in `schema_embeddings` table with content, table_name, column_name
**Rationale:** Enables vector similarity search to find relevant schema context for SQL generation. Content field contains natural language descriptions.
**Date:** 2026-02-06

---

### Known Limitations / Technical Debt

**Conv:** conv-20260206
**Decision:** Hardcode location ID map in `extract_location_from_sql()`
**Rationale:** Quick implementation. Should be replaced with database lookup for production. Current map: {1: Fincastle, 2: Rome, 3: Roanoke}.
**Date:** 2026-02-06

---

**Conv:** conv-20260206
**Decision:** Download embedding model on container start
**Rationale:** Simpler Dockerfile. Adds ~30s to cold start. Could be baked into image for faster startup if needed.
**Date:** 2026-02-06

---

### Phase 2: IoT Control Layer

**Conv:** conv-20260206
**Decision:** Use Mosquitto as MQTT broker, Sauron as MQTT client
**Rationale:** Mosquitto is battle-tested, lightweight, supports WebSockets for web clients. Sauron acts as the central orchestrator that sends commands to IoT nodes, not a broker itself. Nodes connect directly to Mosquitto.
**Date:** 2026-02-06

---

**Conv:** conv-20260206
**Decision:** Topic structure: `controlcore/{node_uuid}/{type}/{capability}`
**Rationale:** Hierarchical structure enables wildcard subscriptions. Node UUID provides unique identification. Type (status/sensor/command/response) separates message purposes. Capability allows per-sensor subscriptions.
**Date:** 2026-02-06

---

**Conv:** conv-20260206
**Decision:** Hybrid command parsing: pattern-based first, LLM fallback
**Rationale:** Pattern matching handles 80% of commands instantly without API calls. LLM fallback catches complex/ambiguous commands. Pattern confidence threshold (0.7) triggers LLM fallback.
**Date:** 2026-02-06

---

**Conv:** conv-20260206
**Decision:** All AI-initiated actions must pass safety engine
**Rationale:** Critical for unattended operation. Safety rules prevent dangerous conditions (watering during freeze, extreme temperatures). Built-in rules cannot be disabled. Database rules can be customized.
**Date:** 2026-02-06

---

**Conv:** conv-20260206
**Decision:** Safety decisions: APPROVED, DENIED, MODIFIED, REQUIRES_CONFIRMATION
**Rationale:** MODIFIED allows automatic parameter adjustment (e.g., 120min watering → 60min max). REQUIRES_CONFIRMATION enables human-in-the-loop for high-risk actions. All decisions are logged for audit.
**Date:** 2026-02-06

---

**Conv:** conv-20260206
**Decision:** Built-in critical safety rules hardcoded in safety engine
**Rationale:** These rules cannot be disabled via database: no watering during freeze, max single watering 60min, max daily watering 180min, temperature limits 40-90°F, no irrigation during high wind.
**Date:** 2026-02-06

---

**Conv:** conv-20260206
**Decision:** Action log stores: initiator, action, parameters, original_request, ai_reasoning, safety_check_result, safety_notes
**Rationale:** Complete audit trail for AI-initiated actions. Enables post-incident analysis. Supports "show me what the AI did yesterday" queries.
**Date:** 2026-02-06

---

**Conv:** conv-20260206
**Decision:** Lazy-load command parser, MQTT service, and safety engine
**Rationale:** Faster API startup. Services only initialized when first needed. MQTT disabled gracefully if MQTT_HOST not set (allows query-only mode).
**Date:** 2026-02-06
