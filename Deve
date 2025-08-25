# DevMemory Agent — Development Plan (OpenAI Edition)
_A demo Agentic LLM-based developer system with Mem0 memory and OpenAI models_

> **Assumption:** `OPENAI_API_KEY` is provided via a **repo secret** (e.g., GitHub Actions → Settings → Secrets and variables → `OPENAI_API_KEY`) and exposed to the app process as an environment variable. Local dev should also export `OPENAI_API_KEY` in the shell or a secrets manager.

---

## Overview
We will build **DevMemory Agent**, a self-hostable developer assistant that:
- Reads a repo, answers questions, drafts patches, and runs tests.
- **Remembers** project conventions, user preferences, and episodic facts via **Mem0**.
- Uses **OpenAI** for generation **and** embeddings; Mem0 is configured with `provider: "openai"`.
- Vector store: **Qdrant** (or **PgVector**); observability via **OpenTelemetry** (traces/metrics/logs) + **Langfuse** (LLM spans & analytics).

---

## Architecture

### Components
- **API & Orchestrator**: FastAPI service.
- **Tools**: repo reader (ripgrep + lightweight AST), git ops, test runner, patch applier.
- **Memory Service**: Mem0 client wrapper exposing `add`, `search`, `get_all`, `update`, `delete`, `history` with consistent metadata.
- **LLM Runtime**: OpenAI chat models (e.g., `gpt-4o`, `gpt-4.1`, `o4-mini`) and embeddings (`text-embedding-3-large`).
- **Vector DB**: Qdrant (default) or PgVector; configure for **3072-dim** embeddings.
- **Observability**: OpenTelemetry instrumentation + Langfuse (self-hosted or cloud) for LLM spans.

### Data Flow
1. User requests a task (e.g., “Refactor `api/errors.py` to follow our logging style”).
2. Orchestrator queries **Mem0.search** with `user_id`, `agent_id`, `run_id`, and metadata (e.g., `project`, `topic`).
3. Retrieved memories are ranked and **injected into prompts** sent to OpenAI.
4. Agent plans → edits code → runs tests → commits.
5. New learnings are written back via `Mem0.add(...)` with categories and metadata.

---

## Memory Model

**Identifiers**
- `user_id`: developer handle/email
- `agent_id`: `"devmemory-agent"`
- `run_id`: session/request UUID (rotated per task)

**Metadata Schema Example**
```json
{
  "project": "repo_name",
  "language": ["python"],
  "category": "factual|episodic|working|semantic",
  "topic": ["logging", "tests", "errors.py"],
  "source": "tool|user|llm",
  "confidence": 0.0,
  "tags": ["style", "pytest"]
}
```

**Policies**
- Durable facts → `factual`.
- Session notes → `working` (scoped to `run_id`).
- Episodic events → `episodic`.
- Conceptual knowledge → `semantic`.
- All updates produce `history` entries; deletions are logged.

---

## Prompts

- **Planner**: user request + top-K memories + repo context → ordered step plan.
- **Coder**: focused memories (factual + recent episodic only) → code changes/diff.
- **Critic**: checks diffs against remembered conventions; may propose memory updates.

Each injected memory chunk is annotated with `(mem_id, type, score, metadata)`; the app emits **MEMORY_INJECTED** events and OTel spans.

---

## Deployment (Local & CI)

### Configuration
- **Secrets**: `OPENAI_API_KEY` provided via repo secret → exported to the runtime environment.
- **Vector DB**: Qdrant (`localhost:6333`) or PgVector; ensure collection/table supports **3072** dimensions (for `text-embedding-3-large`).
- **Mem0**: configured to use OpenAI as both LLM and embedding providers.
- **Observability**: OTel exporter (OTLP) → collector/Tempo/Grafana; Langfuse for LLM spans.

### Minimal Mem0 Config (Python)
```python
from mem0 import Memory
import os

config = {
  "llm": {
    "provider": "openai",
    "config": {
      "model": "gpt-4o",        # or gpt-4.1, o4-mini, etc.
      # api_key omitted to read from env: OPENAI_API_KEY
    }
  },
  "embeddings": {
    "provider": "openai",
    "config": {
      "model": "text-embedding-3-large"  # 3072-dim vectors
    }
  },
  "vector_store": {
    "provider": "qdrant",
    "config": {"host": "localhost", "port": 6333, "collection_name": "devmemory"}
  }
}

mem = Memory.from_config(config)
```

### GitHub Actions (snippet)
```yaml
# .github/workflows/ci.yml (excerpt)
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    env:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install -r requirements.txt
      - run: pytest -q
```

---

## Observability

### Traces (OpenTelemetry)
- Span graph: `chat.request → plan → retrieve_memories → code_edit → run_tests → commit`.
- Each Mem0 call (`add/search/get/update/delete/history`) is a span with attributes: `user_id`, `agent_id`, `run_id`, `query`, `filters`, `num_hits`, `latency_ms`.

### LLM Spans (Langfuse)
- Capture prompts, outputs, token counts, cost, model.
- Tag spans with `memory_ids_used`, `project`, `topics`.

### Metrics
- **Recall@K**: rate of tasks where the correct convention memory was injected.
- **Memory Drift Rate**: frequency of contradictions/updates.
- **Latency breakdown**: retrieval vs. generation vs. tools.

### Logs (structured JSON)
- `MEMORY_INJECTED`, `MEMORY_ADDED`, `MEMORY_UPDATED`, `MEMORY_DELETED` with `(mem_id, category, metadata, diff)`.
- Apply redaction for secrets in prompts/outputs.

### Dashboards
- **Memory Timeline** per user/project (episodic & factual events, updates via `history`).
- **Recall & Precision** panels derived from the test harness.
- **Latency & Cost** by model and workflow.

---

## Milestones

### M0 — Skeleton & Local Stack (1–2 days)
- Bring up Qdrant (Docker), Mem0 (OpenAI providers), FastAPI skeleton.
- Endpoints: `/health`, `/memory/smoke` (writes & reads a test memory).
- Integrate OpenTelemetry + Langfuse (OpenAI client wrapper) and verify spans.

**Unit Tests**
- `test_mem0_add_get_search_basic`
- `test_mem0_scoping_user_vs_session`
- `test_metadata_filters`
- `test_history_update_delete`

---

### M1 — Orchestrator & Tools (2–3 days)
- Repo reader (ripgrep + minimal AST summaries);
- Tool adapters: git ops, patch applier, test runner;
- Planner/Coder/Critic prompts; memory injection policy (top-K by type/recency/confidence, temp=0 for determinism);
- Post-task extraction → `Mem0.add(messages, user_id, agent_id, run_id, metadata={...})`.

**Integration Tests**
- `test_injection_uses_factual_conventions`
- `test_session_working_memory_not_persisted`
- `test_update_convention_and_history`
- `test_metadata_scoping_by_project`

**Obs Checks**
- `retrieve_memories` span has `num_hits>0`; Langfuse trace links to `memory_ids_used`.

---

### M2 — Demo Workflows (2 days)
- **Workflow A: Adopt repo logging style**
  1) Seed memory: "use `structlog` with `contextvars`" → ask agent to instrument file;
  2) Verify tests pass and style adhered;
  3) Verify memory injection was present (trace/log).
- **Workflow B: Fix flaky test & remember root cause**
  - After fix, `Mem0.add` episodic entry with root cause + resolution tags.

**End-to-End Tests**
- `e2e_style_adherence_from_memory`: memory retrieved & injected; diff follows style; episodic memory created summarizing action.
- `e2e_memory_evolution`: conflicting info triggers critic to propose **update**; `history` shows prior→new value.

---

### M3 — Advanced Features (optional)
- Graph memory / entity linking (modules ↔ owners) and visualize it in the Memory Timeline.
- v2 filtering to assemble topic “memory packs” (e.g., `topic in ["logging","errors"] AND category="factual"`).

---

## API Sketch (FastAPI)

- `POST /chat` → runs the end-to-end loop; returns answer + memory hits + trace IDs.
- `POST /memory/search` → maps to Mem0 `search`/`get` with filters.
- `POST /memory/add` → add memories with metadata; default `category=factual` unless specified.
- `POST /memory/update|delete|history` → passthrough to Mem0 maintenance APIs.

All responses include a `telemetry` block: `trace_id`, `span_ids`, `memory_ids_used`.

---

## Test Strategy (Full)

### Unit
1. **Mem0 Wrapper**: add/get/search/update/history/delete/reset happy + edge cases; metadata round-trip.
2. **Ranking & Injection**: deterministic ordering by type/recency/confidence; ensure at least one factual + one episodic memory is preserved after truncation.
3. **Prompt Builders**: provenance footer lists `(mem_id, type, topic)` for every injected memory.

### Integration
1. **Local Stack**: Qdrant + Mem0 (OpenAI) up; embeddings dimension == 3072; collection exists.
2. **Observability**: OTel spans present; Langfuse traces include prompts/tokens and `memory_ids_used`.
3. **Search Semantics**: `user_id` isolation; `run_id` scoping; complex metadata filters.

### End-to-End
- **Recall Gate**: Recall@K ≥ 0.8 on curated prompts.
- **Latency Gate**: P95 ≤ (configurable target) across workflows.
- **Isolation Gate**: No memory leakage across projects/users.

> Use `temperature=0` for stable assertions; where text varies, assert on **behavioral signals** (tests pass, diffs compile, memory used) rather than exact tokens.

---

## Demo Playbook

1. Start stack: Qdrant, FastAPI, Langfuse.
2. Ensure `OPENAI_API_KEY` is available via environment (from repo secret in CI, local export in dev).
3. Seed factual memory: "Repo **X** uses **Black line-length 100**, logging via **structlog**." (`category=factual`, `project=X`, `topic=["style","logging"]`).
4. Ask: "Refactor `api/errors.py` to follow our logging style."
5. Observe:
   - Memory hits (IDs & snippets) included in the prompt payload.
   - OTel span `retrieve_memories` with `num_hits` and filters displayed.
   - Langfuse trace for the OpenAI call shows prompt segments and `memory_ids_used`.
6. Update convention to "line-length 120" → rerun → verify `history` entry and updated behavior.

---

## Risks & Mitigations
- **Token/cost sensitivity** → monitor in Langfuse; prefer lighter models (e.g., `o4-mini`) for routine tasks; summarize memories before injection.
- **Embedding dimension mismatch** → enforce 3072-dim schema; add migration checks in startup.
- **API rate limits** → exponential backoff/retry; instrument 429/timeout counters; fallbacks for read paths.
- **Memory bloat** → category quotas + decay; periodic audits surfaced on dashboard.
- **Cross-project leakage** → require `project` metadata on all add/search operations; add guardrails in the wrapper.

