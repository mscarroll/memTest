# DevMemory Agent — Local Filesystem Edition (OpenAI + Mem0)
_A zero‑server demo of an Agentic LLM developer assistant that showcases Mem0’s memory using only your local filesystem._

---

## Assumptions
- `OPENAI_API_KEY` is provided via the repo secret in CI and as an environment variable locally.
- No Docker, no long‑running servers. Everything runs as Python CLI scripts.
- Vector DB is **Chroma** in **in‑process**/local‑disk mode (no separate service).
- Traces/metrics/logs use **OpenTelemetry’s Console exporter** (stdout) and **JSON logs** on disk — no collector needed.

---

## Goals
- Show **how Mem0 remembers** (factual, episodic, working, semantic) in a developer workflow.
- Keep setup simple: `pip install … && python -m devmemory …`.
- Strong **observability** without external infra: spans to console, structured logs to `.telemetry/` (Windows: `.\\.telemetry\\`).

---

## Minimal Architecture (no servers)

```
repo/
├─ devmemory/                    # Python package (CLI tools)
│  ├─ cli.py                     # Typer-based CLI entrypoint
│  ├─ agent.py                   # Orchestrator (planner/coder/critic)
│  ├─ mem0_client.py             # Mem0 wrapper (OpenAI providers)
│  ├─ store_chroma.py            # Chroma init (local persistence)
│  ├─ repo_tools.py              # Python file walker/AST, git ops, patch, tests
│  ├─ observability.py           # OTel setup (console exporter) + JSON logger
│  └─ prompts/                   # prompt templates
├─ tests/                        # pytest (unit/integration/e2e)
├─ .telemetry/                   # local JSON logs & span dumps (gitignored)
├─ requirements.txt
└─ README.md / DEVELOPMENT_PLAN.md
```

### Components
- **CLI (Typer)**: commands like `plan`, `apply`, `run-tests`, `chat`, `memory:*`.
- **Mem0 wrapper**: `add/search/get/update/history/delete` with scoped identifiers & metadata.
- **Vector store: Chroma (in‑process)** with disk persistence in `.chroma/` (Windows: `.\\.chroma`).
- **OpenAI**: chat model (e.g., `gpt-4o` / `o4-mini`) + `text-embedding-3-large` (3072‑d) via Mem0 config.
- **Observability**: OpenTelemetry **ConsoleSpanExporter** for traces; JSON logs (one line per event) to `.telemetry/`.

---

## Data Flow (single command example)
1. User runs `python -m devmemory chat "Refactor api/errors.py to our logging style" --project X`.
2. Agent loads repo context (Python file walker/AST) and **Mem0.search** with filters.
3. Top‑K memories are ranked (factual first, then episodic recency) and injected into the prompt.
4. Model proposes edits; CLI writes patch; `python -m pytest -q` is run; results summarized.
5. New durable facts are added with `category=factual`; task notes → `episodic`; transient notes → `working`.
6. Every step emits spans to console and JSON events into `.telemetry/` for later inspection.

---

## Configuration

### Mem0 (Python)
```python
# devmemory/mem0_client.py
from mem0 import Memory

CONFIG = {
  "llm": {"provider": "openai", "config": {"model": "o4-mini"}},
  "embeddings": {"provider": "openai", "config": {"model": "text-embedding-3-large"}},
  "vector_store": {"provider": "chroma", "config": {"path": ".\\.chroma", "collection_name": "devmemory"}}
}
mem = Memory.from_config(CONFIG)
```
- **Chroma** runs inside the process with on‑disk persistence.
- Ensure your vector store supports **3072‑dim** embeddings.

### Observability (no collector)
```python
# devmemory/observability.py
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor, ConsoleSpanExporter
from opentelemetry import trace

provider = TracerProvider()
processor = BatchSpanProcessor(ConsoleSpanExporter())
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("devmemory")
```
- Console exporter prints spans to stdout.

---

## Memory Model
- **Identifiers**: `user_id`, `agent_id="devmemory-agent"`, `run_id`.
- **Metadata** example:
```json
{ "project":"repo_name", "language":["python"], "category":"factual|episodic|working|semantic", "topic":["logging","tests"], "source":"tool|user|llm", "confidence":0.0, "tags":["style","pytest"] }
```
- **Policies**: durable→`factual`, session→`working`, events→`episodic`, concepts→`semantic`; use `history` on updates.

---

## Observability Details
- **Trace spans**: `cli.request → plan → retrieve_memories → code_edit → run_tests → write_patch`.
- **Span attrs** on Mem0 calls: `user_id`, `agent_id`, `run_id`, `query`, `filters`, `num_hits`, `latency_ms`.
- **JSON events** (per line) in `.telemetry/`:
  - `MEMORY_INJECTED {mem_ids:[...], types:[...], topics:[...], project:"X"}`
  - `MEMORY_ADDED {mem_id, category, metadata}`
  - `MEMORY_UPDATED {mem_id, before, after}`
  - `TEST_RESULTS {passed, failed, duration}`

---

## Test Plan (Pytest)

### Unit
- Mem0 wrapper: add/get/search/update/history/delete.
- Ranking & injection: deterministic ordering.
- Prompt builders: provenance footer includes `(mem_id, type, topic)`.

### Integration
- Chroma persistence across runs.
- Search semantics: `user_id` isolation; `run_id` scoping; metadata filters.
- Observability: spans to console; JSON events written.

### End‑to‑End
- Style adherence from memory.
- Memory evolution with `history` tracking.
- Isolation gates (no leakage across projects).

---

## CLI Commands (Windows examples)
```powershell
python -m devmemory memory:add --project X --category factual "Use Black line-length 100; structlog+contextvars"
python -m devmemory memory:search --project X --topic logging
python -m devmemory plan "Adopt repo logging style" --project X
python -m devmemory chat "Refactor api/errors.py to our logging style" --project X
python -m devmemory run-tests
```
Each command prints a Trace Summary and writes a JSON artifact under `.\\.telemetry\\`.

---

## Milestones

**M0 — Bootstrap (≤1 day)**
- Skeleton package, Typer CLI, OTel console exporter, JSON logger.
- Implement Mem0 config. Smoke test add/search.
- Unit tests.

**M1 — Agent Loop (1–2 days)**
- Prompts, injection policy, repo tools.
- Integration tests + observability checks.

**M2 — Demo Scenarios (1 day)**
- Logging style adoption, flaky test fix.
- End‑to‑end tests.

---

## CI (GitHub Actions)
```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: windows-latest
    env:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - run: pip install -r requirements.txt
      - run: python -m pytest -q
```

---

## Demo Playbook
1. Add memory: `python -m devmemory memory:add --project X --category factual "Black 100; structlog+contextvars"`
2. Run chat: `python -m devmemory chat "Refactor api/errors.py to our logging style" --project X`
3. Inspect console spans and `.\\.telemetry\\*.jsonl` for injected memories, test results, and history updates.

---

## Risks & Mitigations
- Embedding dimension mismatch (3072) → assert at startup.
- Local file bloat → rotate logs, quotas.
- LLM nondeterminism → `temperature=0`; assert on behavior not exact strings.

