# Healthcare Analytics Agent

Structured analyst assistant over synthetic healthcare claims in DuckDB: plans before querying, guarded SQL, audit exports, and optional trust layers (peer review, HITL approval, RBAC).

**Quick start**

```bash
uv sync
cp env.template .env   # add OPENAI_API_KEY
make all             # or see Windows setup below
uv run streamlit run streamlit_app.py
```

**Architecture details:** [`ARCHITECTURE.md`](ARCHITECTURE.md)

---

## What I Built

Healthcare analysts spend hours writing SQL to answer questions they ask every week. This submission extends the starter chat into a structured analyst assistant that plans before acting, catches its own mistakes, and leaves a complete audit trail of every query it ran.

The core insight: in production healthcare analytics, **reproducibility and trust** matter as much as the answer itself. An analyst cannot rely on a result they cannot explain or reproduce.

### What it does differently from the starter

- **Plans before querying** — Data Analyst planner (JSON plan + viz hints) before database tools; legacy planner fallback if needed.
- **Self-corrects SQL errors** — retries capped at **2** failed `query_database` calls per turn in code (`retry_limit_reached`).
- **Audit trail** — session markdown export: questions, SQL, tool results, peer review.
- **Peer review** — second LLM pass vs tool evidence; optional separate `PEER_REVIEW_MODEL`; non-`pass` verdicts surface a notice in the answer.
- **Cohort continuity** — session state injects last cohort-defining SQL so “same cohort” follow-ups resolve without a separate tool.
- **Streamlit UI** — streaming answers, chart display, follow-up buttons, role selector, HITL approval panel.
- **Guarded SQL** — read-only statements, required `LIMIT`, server-side cap, timeouts.
- **RBAC** — `analyst` / `viewer` / `admin` roles filter tools and tables.
- **HITL** — optional pause before large-`LIMIT` SQL for human approve/reject.
- **Observability** — structured logs and per-turn traces (`observability.py`).

---

## Per-turn pipeline

```text
Planner (Data Analyst) → Executor (tool loop) → [HITL pause?]
→ Chart narrative → Report Writer → Follow-up ranking (top 3) → Peer review
```

Optional phases are disabled via env flags (see [Configuration](#configuration)).

---

## Entry points vs agent core

| Module | Role |
|--------|------|
| [`chat.py`](chat.py) | Terminal REPL; calls **`run_user_turn_async`**; `export` / exit; terminal HITL y/N when enabled. |
| [`streamlit_app.py`](streamlit_app.py) | Web UI; same async orchestrator; streaming, follow-up buttons, audit download, role + HITL settings. |
| [`agent_orchestrator.py`](agent_orchestrator.py) | Planner, executor, finalize passes, peer review, HITL checkpoint/resume. |
| [`tools/tool_registry.py`](tools/tool_registry.py) | Tool schemas + `dispatch_tool` (only execution path). |
| [`observability.py`](observability.py) | Logging, `trace_id` / `turn_id`, spans. |

---

## Features

- **Models:** `CHAT_MODEL` (default `gpt-4o`); optional `PEER_REVIEW_MODEL` for peer review.
- **Tools:** `list_tables`, `query_database`, `summarize_sql_stats`, `describe_table`, `table_info`, `profile_table`, `create_chart`, `data_quality_check`, `analyze_care_gap`
- **Query cache** — in-memory TTL ~5 min for identical SQL; charts not cached.
- **Ranked follow-ups** — ~6 candidates generated; top **3** kept by relevance/novelty heuristics.
- **Sub-agents (finalize path):** Visualization gate (when analyst prioritizes charts), Report Writer, chart narrative with plotted `sample_rows`.
- **Session export** — `outputs/reports/session_<timestamp>.md` with executive summary, reproducibility block, per-turn log, `error_kind` / `next_step` on failures.
- **SQL guardrails** — [`tools/sql_guard.py`](tools/sql_guard.py): read-only, `LIMIT` required, `SQL_SERVER_MAX_LIMIT`, wall-clock timeout.
- **Context pruning** — long threads compressed; cohort/session digest preserved in summary ([`tools/context_manager.py`](tools/context_manager.py)).

---

## Database tools

| Tool | Purpose |
|------|---------|
| `list_tables` | List tables (filtered by role) |
| `query_database` | Guarded SQL → JSON rows (cached when identical) |
| `summarize_sql_stats` | Numeric summaries on guarded SQL |
| `describe_table` / `table_info` | Schema and samples |
| `profile_table` | DuckDB `SUMMARIZE` on capped sample |
| `create_chart` | Two-column SQL → PNG under `outputs/visualization/` |
| `data_quality_check` | Null/duplicate checks on a table |
| `analyze_care_gap` | E11 + LAB utilization proxy (analyst/admin) |

**Tables:** demographics, geography, mortality, mx_events, rx_events; lookups: icd10_codes, procedure_codes, ndc_products. Join on `PATIENT_NUMBER`.

**Example workflows:** [`examples/README.md`](examples/README.md) · **Demo:** [`examples/demo_cohort_and_care_gap_workflow.md`](examples/demo_cohort_and_care_gap_workflow.md)

---

## Configuration

Copy [`env.template`](env.template) to `.env`. Do **not** commit `.env`.

| Variable | Purpose |
|----------|---------|
| `OPENAI_API_KEY` | Required |
| `APP_USER_ROLE` | `analyst` (default), `viewer`, or `admin` |
| `CHAT_MODEL` | Executor / planner model (default `gpt-4o`) |
| `PEER_REVIEW_MODEL` | Reviewer model (defaults to `CHAT_MODEL` if unset) |
| `DISABLE_PLANNER` | Skip planner (`1` / `true`) |
| `DISABLE_REPORT_WRITER` | Skip final rewrite pass |
| `DISABLE_VIZ_AGENT` | Skip visualization approval LLM |
| `DISABLE_CHART_NARRATIVE` | Skip chart interpretation sentence |
| `DISABLE_PEER_REVIEW` | Skip peer review |
| `DISABLE_FOLLOWUP_RANKING` | Keep all follow-up bullets |
| `SQL_SERVER_MAX_LIMIT` | Cap enforced LIMIT (default `10000`) |
| `ENABLE_QUERY_APPROVAL` | HITL before large LIMIT SQL |
| `QUERY_APPROVAL_MIN_LIMIT` | HITL threshold (default `1000`) |
| `LOG_LEVEL` | `DEBUG`, `INFO`, … |
| `LOG_FORMAT` | `text` or `json` |
| `LOG_TO_FILE` | Append to `outputs/logs/agent.log` |

Streamlit sidebar mirrors planner-off, HITL, and role settings.

---

## Observability and debugging

Logging is configured at startup in `chat.py` and `streamlit_app.py` via [`observability.py`](observability.py).

```bash
# .env
LOG_LEVEL=DEBUG
LOG_TO_FILE=1
```

Each user turn gets a **`turn_id`**; the process has a **`trace_id`**. Search logs for:

| `event` | Meaning |
|---------|---------|
| `turn.start` / `turn.end` | Turn lifecycle (`status`: ok, error, approval_pending) |
| `turn.exception` | Unhandled failure (stack trace included) |
| `executor.tool_round` | Tool round N |
| `tool.result` | Tool ok or failed (`error_kind` when failed) |
| `sql.execute` | DuckDB run or policy block |
| `hitl.pause` / `hitl.resume` | Approval gate |
| `peer_review.done` | Verdict logged |

---

## Prerequisites

- Python 3.13+
- [uv](https://github.com/astral-sh/uv)
- OpenAI API key

## Setup

### Quick (macOS/Linux with `make`)

```bash
make all
```

### Windows (PowerShell)

```powershell
uv sync
Expand-Archive -Path "data\input.zip" -DestinationPath "data" -Force
uv run python data\generate_data.py
uv run python scripts\load_archives_to_duckdb.py
```

### Manual + API key

```bash
uv sync
make generate-data
make setup-db
cp env.template .env   # edit OPENAI_API_KEY
```

**Makefile:** `install`, `generate-data`, `setup-db`, `clean-db`, `test-db`, `all`, `help`

## Usage

```bash
uv run chat.py
```

```bash
uv run streamlit run streamlit_app.py
```

**Terminal:** `export` saves audit markdown; `exit` / `quit` / Ctrl+C exits (exports if there is content).

---

## Tests

```bash
uv sync --extra dev
uv run pytest
```

- **Unit / policy (no DB):** `test_sql_guard`, `test_sql_error_hints`, `test_query_cache`, `test_session_*`, `test_peer_review`, `test_followup_ranking`, `test_permissions`, `test_observability`, `test_approval_policy`, `test_orchestrator_config`, `test_orchestrator`
- **Integration (needs `healthcare.duckdb`):** `test_tool_registry` — skipped if DB missing

---

## Architecture decisions (summary)

| Decision | Rationale |
|----------|-----------|
| Planner + executor | Fewer wasted SQL attempts on multi-table questions |
| Tool registry | Add tools in one place; entry points unchanged |
| Retry cap in code | Model cannot loop failed SQL past 2 attempts |
| Session state in system prompt | Cheap cohort memory without a vector DB |
| Markdown export | Diffable audit for reviewers |
| Peer review | Lightweight trust check vs tool evidence |
| Custom orchestration (no LangGraph) | Full control over guards, HITL, and cost flags |

---

## Design notes

- **Orchestration:** Custom Python loop on OpenAI tool calling — not LangChain/LangGraph.
- **Streaming:** Streamlit streams executor completion via `run_user_turn_async(..., stream_delta=...)`.
- **Charts:** matplotlib `Agg` backend; two SQL columns → PNG + optional narrative from `sample_rows`.
- **Export:** `db_query.py` unchanged per assessment constraints.

---

## Example workflow (diabetes + metformin cohort)

**User query**

> How many patients have at least one E11% admit diagnosis on medical claims and at least one metformin fill?

Screenshot (optional): `screenshots/diabetes_cohort1.png`

**Planner (illustrative)**

```text
1. mx_events: distinct PATIENT_NUMBER with ADMIT_DIAGNOSIS_CODE LIKE 'E11%'.
2. rx_events: distinct PATIENT_NUMBER with metformin in GENERIC_NAME.
3. Inner join; COUNT DISTINCT; LIMIT 1.
```

**SQL (illustrative)**

```sql
WITH dm_patients AS (
  SELECT DISTINCT "PATIENT_NUMBER"
  FROM mx_events
  WHERE "ADMIT_DIAGNOSIS_CODE" LIKE 'E11%'
),
metf AS (
  SELECT DISTINCT "PATIENT_NUMBER"
  FROM rx_events
  WHERE LOWER(COALESCE("GENERIC_NAME", '')) LIKE '%metformin%'
)
SELECT COUNT(DISTINCT m."PATIENT_NUMBER") AS patients_dm_and_metformin
FROM metf AS m
INNER JOIN dm_patients AS d ON m."PATIENT_NUMBER" = d."PATIENT_NUMBER"
LIMIT 1;
```

**Peer review (illustrative)**

```markdown
### Verdict
**Pass** — The stated patient count matches the single-row result from `query_database`.
```

More examples: [`examples/diabetes_workflow_sql_examples.md`](examples/diabetes_workflow_sql_examples.md), [`examples/cost_and_chart_sql_examples.md`](examples/cost_and_chart_sql_examples.md).

Session audit files under `outputs/reports/` and chart PNGs under `outputs/visualization/` are gitignored; generate locally.

---

## Implemented extensions (beyond starter)

- Ranked follow-up suggestions (`rank_suggested_followups`)
- Role-based permissions (`tools/permissions.py`)
- Partial multi-agent finalize: Data Analyst planner, Visualization gate, Report Writer, chart narrative
- HITL SQL approval (`tools/approval_policy.py`)
- Structured logging and tracing (`observability.py`)
- SQL read-only validation + server LIMIT cap
- Actionable peer review verdicts in the UI and answer text

## Possible next steps

1. **Different reviewer model in production** — `PEER_REVIEW_MODEL` is supported; document evals comparing models on verdict quality.
2. **Non-blocking long SQL** — async DB job queue with progress messages while queries run.
3. **Golden-turn regression tests** — mocked OpenAI end-to-end tests for `run_user_turn_async`.
4. **Persistent session state** — survive Streamlit refresh or support multi-day cohort work.
5. **Stronger SQL parsing** — AST-based allowlist instead of regex for table/RBAC checks.
6. **Full orchestrator state machine** — explicit delegate-to-specialist graph vs sequential finalize passes.
