# System Specifications — Agentic SDLC Orchestration System (URL Shortener)

Version 1.0 | Status: Baseline for implementation
Companion documents: `PLAN.md` (implementation plan), `docs/ARCHITECTURE.md` (architecture overview)

---

## 1. Purpose and Scope

This document specifies the functional and non-functional requirements, external interfaces, data contracts, and acceptance criteria for:

1. **The Agentic SDLC Orchestration System** — a governed workflow engine that coordinates specialized AI agents through the software development lifecycle (requirements, decomposition, architecture, implementation, testing, documentation, release) under controlled autonomy.
2. **The Target Application** — a URL shortener service that the orchestration system builds (greenfield), enhances (brownfield), and hardens against an ambiguous requirement.

**Out of scope** (documented trade-offs, not deficiencies): distributed/multi-node execution, run resume after safe-stop, authentication/authorization on service endpoints, git-based change management inside the engine, code-coverage-percentage gates.

## 2. Definitions

| Term | Definition |
|---|---|
| Run | One end-to-end execution of a scenario workflow, identified by `run_id` |
| Task | A node in the workflow DAG, executed by exactly one agent |
| Gate | A predicate that must pass to enter (entry gate) or leave (exit gate) a task |
| Artifact | An immutable, versioned, sha256-hashed output of a task attempt |
| Changeset | An artifact listing files + contents to be applied to the application directory |
| Apply | Copying a validated changeset into the run's `output/` directory after snapshotting |
| DecisionRecord | Lineage record linking consumed artifact versions (by hash) to produced ones |
| Fault injection | A scenario-declared, audit-labeled deterministic failure used for demonstration |
| Offline mode | Deterministic agent generation with no LLM/network dependency |

---

## 3. Functional Requirements

Requirement IDs are stable and referenced by tests and the traceability matrix (Section 12).

### 3.1 Orchestration Engine (FR-ORCH)

| ID | Requirement |
|---|---|
| FR-ORCH-1 | Workflows SHALL be declared as an explicit dependency graph (DAG) of Tasks with `depends_on`, `consumes`, `produces`, entry gates, exit gates, retry policy, and optional fallback. |
| FR-ORCH-2 | The engine SHALL validate the workflow at load time: acyclicity, every consumed artifact produced upstream, every referenced agent registered. Validation failure SHALL abort the run before any task executes. |
| FR-ORCH-3 | The engine SHALL support sequential and parallel execution paths with synchronization at join nodes; concurrent task execution SHALL be bounded by a configurable parallelism limit (default 3). |
| FR-ORCH-4 | Each task SHALL follow the state machine in Section 10; all state changes SHALL pass through a single transition function enforcing a legal-transition table; illegal transitions SHALL raise an error. |
| FR-ORCH-5 | Entry gates SHALL verify: all dependencies DONE, required artifacts exist and are current (hash match), policy pre-checks pass, safe-stop not engaged. |
| FR-ORCH-6 | Exit gates SHALL execute in fixed order: schema validation, policy post-checks, test gate (if declared), approval gate (if declared) last. |
| FR-ORCH-7 | Changesets SHALL be staged in an attempt directory and applied to `output/` only after all exit gates pass, and only after snapshotting the current `output/` state. |
| FR-ORCH-8 | Failures SHALL be classified TRANSIENT, VALIDATION, POLICY_BLOCK, or FATAL, with class-specific handling per FR-ORCH-9..11. |
| FR-ORCH-9 | TRANSIENT failures SHALL be retried with bounded attempts (default 3) and exponential backoff with jitter. VALIDATION failures SHALL be retried immediately with the failure output injected as a feedback input to the next attempt. |
| FR-ORCH-10 | POLICY_BLOCK SHALL transition the task to BLOCKED with no automatic retry; only an explicit, audited human override or safe-stop may release it. |
| FR-ORCH-11 | When LLM attempts are exhausted and an offline fallback exists, the engine SHALL make one fallback attempt and emit `FALLBACK_ACTIVATED`. |
| FR-ORCH-12 | Rollback SHALL restore `output/` from the pre-apply snapshot, retract the artifact version (current pointer returns to the prior version), transition the task to ROLLED_BACK, and invalidate downstream consumers via lineage. |
| FR-ORCH-13 | When any artifact gains a new current version, the re-planner SHALL invalidate exactly those completed tasks whose recorded input hashes no longer match, transitively, and re-queue them (DONE -> INVALIDATED -> PENDING). A `REPLAN_TRIGGERED` event SHALL record cause and affected tasks. |
| FR-ORCH-14 | Safe-stop SHALL be triggerable by (a) Ctrl+C and (b) a `STOP` sentinel file via `run_scenario.py stop <run_id>`. On stop: no new dispatch, in-flight tasks cancelled at await points, non-terminal tasks -> CANCELLED, `SAFE_STOP` emitted, state flushed. A stopped run SHALL never leave `output/` partially applied. |
| FR-ORCH-15 | Cross-stage context SHALL be preserved via the artifact store; every completed attempt SHALL produce a DecisionRecord capturing consumed artifact versions (with sha256), produced artifacts, rationale, and generation mode. |

### 3.2 Agents (FR-AGT)

| ID | Requirement |
|---|---|
| FR-AGT-1 | The system SHALL provide eight role agents: RequirementsAnalyst, Planner, Architect, CodeGenerator, TestEngineer, Reviewer, DocWriter, ReleaseManager, sharing a common base contract (input load, policy pre-check, generate, materialize + validate, policy post-check, decision recording). |
| FR-AGT-2 | Every agent SHALL operate in LLM mode (when `ANTHROPIC_API_KEY` is set and `--llm` selected) and offline mode (default); offline mode SHALL require no network access. |
| FR-AGT-3 | LLM outputs SHALL be validated against the target artifact's schema; on validation failure the client SHALL attempt one repair round-trip before classifying the attempt as a VALIDATION failure. |
| FR-AGT-4 | RequirementsAnalyst SHALL, for ambiguous inputs, produce an AmbiguityReport containing clarifying questions and at least 3 ranked interpretations with scope, risk, and assumptions. |
| FR-AGT-5 | Architect SHALL, in brownfield scenarios, produce an ImpactReport derived from real AST analysis of the existing codebase: impacted modules, endpoints, data flows, and regression risks. |
| FR-AGT-6 | TestEngineer SHALL execute the generated application's test suite via a real pytest subprocess (isolated cwd/env, 120 s timeout) and produce a TestReport from actual results. |
| FR-AGT-7 | Reviewer SHALL perform real static checks on generated code: `py_compile` per file and AST-based import verification. |
| FR-AGT-8 | ReleaseManager SHALL compute the ReleaseChecklist from actual run state (audit log): gates green, tests passed on latest code, mandatory approvals present. |
| FR-AGT-9 | Agents SHALL write only within the run directory; direct writes to `output/` are prohibited (changesets only, applied by the engine per FR-ORCH-7). |

### 3.3 Policy and Governance (FR-POL)

| ID | Requirement |
|---|---|
| FR-POL-1 | Every policy evaluation SHALL produce a verdict PASS, WARN, or BLOCK and SHALL be recorded as a `POLICY_CHECK` audit event with rule id, verdict, details, and compliance tags. |
| FR-POL-2 | **SecretScan**: generated file contents SHALL be scanned for credential patterns (cloud access keys, PEM private-key headers, hardcoded `api_key/password/secret/token` assignments). Any hit -> BLOCK. |
| FR-POL-3 | **ForbiddenImports**: generated application code SHALL be AST-checked against an import allowlist (stdlib + fastapi, pydantic, uvicorn, httpx, pytest, starlette); use of `eval`, `exec`, or `os.system` -> BLOCK. |
| FR-POL-4 | **ProtectedPaths**: changesets SHALL only write inside the run's `output/`; declared protected files within it SHALL require an explicit human override, which is itself audited. |
| FR-POL-5 | **ChangeBudget**: changesets exceeding 15 files or 1200 changed LOC -> BLOCK with guidance to split the change. |
| FR-POL-6 | **TestGate**: code-affecting stages SHALL not pass their exit gate unless pytest exits 0 and collected test count >= the scenario minimum. |
| FR-POL-7 | **ReleasePolicy**: the release gate SHALL require zero unresolved BLOCKs, TestGate green on the latest applied code version, and all mandatory approvals present in the audit log. |

### 3.4 Approvals and Controlled Autonomy (FR-APR)

| ID | Requirement |
|---|---|
| FR-APR-1 | Human approval checkpoints SHALL gate: design, code review, and release in all scenarios; plus requirements sign-off (interpretation selection) in the ambiguous scenario. |
| FR-APR-2 | The approval prompt SHALL present: task id and attempt, artifact summaries, code diff statistics (files/LOC), policy check results table, and lineage note (consumed artifact versions). |
| FR-APR-3 | Approver actions SHALL be: approve; reject with feedback (feedback becomes a new artifact version and triggers re-planning per FR-ORCH-13); view full artifact; safe-stop. |
| FR-APR-4 | Every approval decision SHALL be audited with approver identity, decision, feedback text, timestamp, and the sha256 of the approved content. |
| FR-APR-5 | An `--auto-approve` mode SHALL exist for CI/smoke tests, recording approver `"auto"` and selecting the analyst-recommended interpretation where a choice is required. |

### 3.5 Observability (FR-OBS)

| ID | Requirement |
|---|---|
| FR-OBS-1 | The engine SHALL maintain an append-only `audit.jsonl` with one JSON object per line: `{seq, ts, run_id, correlation_id, event_type, payload}`; every event SHALL be flushed on write. |
| FR-OBS-2 | Audited event types SHALL include: RUN_STARTED/COMPLETED/FAILED, TASK_STATE_CHANGED, GATE_EVALUATED, POLICY_CHECK, APPROVAL_REQUESTED/GRANTED/REJECTED, ARTIFACT_CREATED, DECISION_RECORDED, RETRY_SCHEDULED, FALLBACK_ACTIVATED, ROLLBACK, REPLAN_TRIGGERED, SAFE_STOP. |
| FR-OBS-3 | Injected demo faults SHALL be labeled `payload.injected: true` in every related audit event. |
| FR-OBS-4 | Reliability metrics SHALL be derived solely from the audit log: task success rate, retry count/rate, fallback count, rollback count, policy blocks, approval latency, MTTR (first failure -> subsequent DONE per recovered task), per-stage latency (READY -> DONE), end-to-end wall time. |
| FR-OBS-5 | Metrics SHALL be written to `metrics.json` after every state transition and included in the final CLI summary. |
| FR-OBS-6 | Every run SHALL end by generating `summary/FINAL_SUMMARY.md` from real run data: plan/rationale, artifacts produced, decisions and approvals, policy results, metrics, assumptions, limitations. |

### 3.6 CLI (FR-CLI)

| ID | Requirement |
|---|---|
| FR-CLI-1 | `python run_scenario.py <greenfield|brownfield|ambiguous> [--auto-approve] [--offline|--llm] [--parallelism N]` SHALL execute the named scenario. `--offline` is the default mode. |
| FR-CLI-2 | `python run_scenario.py stop <run_id>` SHALL engage safe-stop for a running run via the sentinel file. |
| FR-CLI-3 | CLI output SHALL be ASCII-only (Windows cp1252-safe) and SHALL end with a summary table: per-task final states, metrics, artifact locations, and the path to FINAL_SUMMARY.md. |
| FR-CLI-4 | Exit codes: 0 = run completed (all terminal tasks DONE or SKIPPED); 2 = run failed; 3 = safe-stopped; 4 = workflow validation error. |

### 3.7 Dashboard (FR-DASH)

| ID | Requirement |
|---|---|
| FR-DASH-1 | The dashboard SHALL run as a separate read-only process (`python -m agentic_sdlc.dashboard`, default port 8600) with zero shared state with the engine (file-based only). |
| FR-DASH-2 | It SHALL display: the DAG with per-task state chips, gates/approvals panel (pending approvals highlighted), audit tail (last 50 events, incremental fetch by `seq`), and metrics cards. |
| FR-DASH-3 | It SHALL function both during a live run (poll interval 1.5 s) and after run completion (post-run inspection/replay). |
| FR-DASH-4 | It SHALL tolerate concurrent engine writes: atomic-replace snapshot reads and a partially written final JSONL line SHALL not cause errors. |

### 3.8 URL Shortener Service (FR-URL)

| ID | Requirement |
|---|---|
| FR-URL-1 | `POST /api/links` SHALL create a short link from `{url, custom_alias?, ttl_seconds?}` returning 201 with `{code, short_url, expires_at}`; 409 if alias taken; 422 for invalid URL or alias. |
| FR-URL-2 | `GET /{code}` SHALL 307-redirect to the original URL and record a click (timestamp, referrer); 404 for unknown/deleted; 410 for expired. |
| FR-URL-3 | `GET /api/links/{code}` SHALL return link metadata; `DELETE /api/links/{code}` SHALL soft-delete (204); `GET /api/links` SHALL list links with paging. |
| FR-URL-4 | `GET /api/links/{code}/stats` SHALL return `{total_clicks, last_clicked_at, clicks_by_day}`. |
| FR-URL-5 | Generated codes SHALL be base62 of `rowid + 100000` (>= 4 chars, collision-free). Custom aliases SHALL match `^[A-Za-z0-9_-]{3,32}$` and not be in the reserved set {api, healthz, docs, admin}. |
| FR-URL-6 | TTL SHALL be enforced lazily at redirect time against an injectable clock; a `purge_expired()` maintenance function SHALL exist. |
| FR-URL-7 | `POST /api/links` SHALL be rate-limited per client IP (fixed window, default 20/min) returning 429 with `Retry-After`. |
| FR-URL-8 | `GET /healthz` SHALL return service and database status. |
| FR-URL-9 | Persistence SHALL be SQLite (stdlib sqlite3), DB path configurable via `SHORTENER_DB` env var. |

---

## 4. Non-Functional Requirements (NFR)

| ID | Requirement |
|---|---|
| NFR-1 | **Portability**: runs on Windows 10 with Python 3.14; no OS-specific APIs beyond documented asyncio Proactor constraints; all file I/O explicit UTF-8. |
| NFR-2 | **Zero-network demo**: all three scenarios complete end-to-end offline with `--auto-approve --offline` (target < 5 minutes each on developer hardware). |
| NFR-3 | **Dependencies**: limited to fastapi, uvicorn, pydantic, httpx, pytest (+ optional anthropic, pytest-asyncio); everything else stdlib. |
| NFR-4 | **Determinism**: offline runs are reproducible; fault injections are declared data, not random. |
| NFR-5 | **Auditability**: system behavior is fully reconstructible from `audit.jsonl` alone; metrics recomputable offline from the log. |
| NFR-6 | **Crash safety**: dashboard-visible files written via temp + atomic replace; audit appends flushed per event; `output/` never torn (apply is post-gate + snapshot-first). |
| NFR-7 | **Testability**: engine logic unit-testable with toy DAGs (no agents, no LLM); target app testable in-process via httpx ASGITransport (no live server). |
| NFR-8 | **Security of generated code**: enforced mechanically by policy rules FR-POL-2/3/4, not by convention. |

---

## 5. API Specification — URL Shortener

Base URL: `http://localhost:8000` (uvicorn default). Content type: `application/json`. Errors follow FastAPI convention: `{"detail": <string | validation errors>}`.

### 5.1 POST /api/links — create short link

Request body:
```json
{
  "url": "https://example.com/some/long/path",   // required, http(s), max 2048 chars
  "custom_alias": "my-link",                      // optional, ^[A-Za-z0-9_-]{3,32}$, not reserved
  "ttl_seconds": 86400                            // optional, integer > 0; omitted = never expires
}
```

Responses:

| Status | Condition | Body |
|---|---|---|
| 201 | Created | `{"code": "e9a1", "short_url": "http://localhost:8000/e9a1", "url": "...", "expires_at": "2026-07-28T12:00:00Z" \| null, "created_at": "..."}` |
| 409 | Custom alias already in use | `{"detail": "alias already exists"}` |
| 422 | Invalid URL / alias format / reserved alias / bad ttl | validation detail |
| 429 | Rate limit exceeded | `{"detail": "rate limit exceeded"}` + `Retry-After` header (seconds) |

### 5.2 GET /{code} — redirect

| Status | Condition | Behavior |
|---|---|---|
| 307 | Active link | `Location: <original url>`; click recorded (clicked_at UTC, referrer from header if present) |
| 404 | Unknown or soft-deleted code | error detail |
| 410 | Link expired (`expires_at` <= now) | error detail; no click recorded |

### 5.3 GET /api/links/{code} — metadata

200: `{"code", "url", "is_custom", "created_at", "expires_at", "deleted", "total_clicks"}` | 404 unknown.

### 5.4 DELETE /api/links/{code} — soft delete

204 on success (idempotent on already-deleted) | 404 unknown code.

### 5.5 GET /api/links/{code}/stats — analytics

200:
```json
{
  "code": "e9a1",
  "total_clicks": 42,
  "last_clicked_at": "2026-07-27T15:04:05Z",
  "clicks_by_day": [{"day": "2026-07-26", "clicks": 30}, {"day": "2026-07-27", "clicks": 12}]
}
```
404 unknown code. Stats remain readable for deleted links.

### 5.6 GET /api/links — list

Query params: `limit` (default 50, max 200), `offset` (default 0), `include_deleted` (default false).
200: `{"items": [<metadata>...], "total": N, "limit": L, "offset": O}`.

### 5.7 GET /healthz

200: `{"status": "ok", "db": "ok"}`; 503 with `{"status": "degraded", "db": "<error>"}` if the DB probe fails.

### 5.8 Dashboard API (read-only, port 8600)

| Endpoint | Returns |
|---|---|
| `GET /` | Single-page dashboard HTML |
| `GET /api/runs` | Available run ids, newest first |
| `GET /api/state?run=<id>` | Contents of `state.json` (DAG nodes, states, gates, pending approvals) |
| `GET /api/metrics?run=<id>` | Contents of `metrics.json` |
| `GET /api/audit?run=<id>&after_seq=N` | Audit events with `seq > N` (incremental tail) |

---

## 6. Data Specifications

### 6.1 URL shortener SQLite schema

```sql
CREATE TABLE links(
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  code TEXT UNIQUE NOT NULL,
  url TEXT NOT NULL,
  is_custom INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL,            -- UTC ISO-8601
  expires_at TEXT NULL,                -- added by brownfield scenario
  deleted INTEGER NOT NULL DEFAULT 0
);
CREATE TABLE clicks(                   -- added by brownfield scenario
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  link_id INTEGER NOT NULL REFERENCES links(id),
  clicked_at TEXT NOT NULL,            -- UTC ISO-8601
  referrer TEXT NULL
);
CREATE INDEX idx_clicks_link ON clicks(link_id);
```

### 6.2 Artifact envelope (all artifacts)

```json
{
  "name": "requirements_spec",
  "version": 2,
  "sha256": "<hex digest of content>",
  "path": "artifacts/requirements_spec/v2/",
  "media_type": "application/json | text/markdown",
  "produced_by": {"task_id": "requirements", "attempt": 1, "mode": "llm | offline"},
  "created_at": "<UTC ISO-8601>"
}
```
Stored immutably under `artifacts/<name>/v<N>/`; `manifest.json` (atomic rewrite) tracks the current (highest non-retracted) version per name.

### 6.3 Structured artifact schemas (pydantic; also LLM output validators)

| Schema | Key fields |
|---|---|
| RequirementsSpec | `problem_statement`, `functional_requirements[] {id, text, priority}`, `non_functional[]`, `assumptions[]`, `open_questions[]` |
| AmbiguityReport | `clarifying_questions[]`, `interpretations[] {id, title, scope, risks[], assumptions[], recommended: bool}` |
| TaskPlan | `tasks[] {id, title, description, depends_on[], estimated_size}` |
| ArchitectureDoc | `overview`, `components[] {name, responsibility}`, `decisions[] {decision, rationale}` |
| ApiContract | `endpoints[] {method, path, request_schema, responses[]}` |
| ImpactReport | `impacted_modules[] {path, symbols[], reason}`, `impacted_endpoints[]`, `data_flow_changes[]`, `regression_risks[]` |
| CodeChangeSet | `files[] {path, content, sha256, action: create|modify|delete, loc_delta}`, `summary`, `total_files`, `total_loc` |
| TestReport | `exit_code`, `collected`, `passed`, `failed`, `errors`, `duration_s`, `failure_tail` (last N lines of pytest output) |
| ReviewReport | `compile_ok: bool`, `import_violations[]`, `comments[] {file, severity, text}`, `verdict: approve|request_changes` |
| ReleaseChecklist | `items[] {check, status: pass|fail, evidence}`, `release_ready: bool` |

### 6.4 DecisionRecord

```json
{
  "task_id": "architecture",
  "attempt": 2,
  "inputs":  [{"artifact": "requirements_spec", "version": 2, "sha256": "..."}],
  "outputs": [{"artifact": "architecture_doc", "version": 1, "sha256": "..."}],
  "rationale": "<agent-provided reasoning summary>",
  "mode": "offline",
  "policy_results": [{"rule": "secret_scan", "verdict": "PASS"}],
  "ts": "<UTC ISO-8601>"
}
```

### 6.5 Audit event schema

```json
{
  "seq": 143,                          // strictly increasing per run
  "ts": "2026-07-27T15:04:05.123Z",
  "run_id": "greenfield-20260727-150000",
  "correlation_id": "codegen_core:2",  // task_id:attempt, or "run" for run-level events
  "event_type": "TASK_STATE_CHANGED",
  "payload": {"from": "RUNNING", "to": "VALIDATING", "reason": "...", "injected": false}
}
```

### 6.6 metrics.json

```json
{
  "run_id": "...", "status": "running | completed | failed | stopped",
  "tasks_total": 10, "tasks_done": 7, "tasks_failed": 0,
  "success_rate_first_attempt": 0.7,
  "retries": 2, "fallbacks": 0, "rollbacks": 1, "policy_blocks": 2,
  "approval_latency_s": {"design_approval": 12.4},
  "mttr_s": 8.9,
  "stage_latency_s": {"requirements": 1.2, "codegen_core": 22.5},
  "end_to_end_s": 145.0,
  "updated_at": "<UTC ISO-8601>"
}
```

---

## 7. Policy Rules Specification

| Rule | Applies to | Verdict logic | Compliance tags |
|---|---|---|---|
| SecretScan | Every CodeChangeSet, every generated text artifact | BLOCK on: `AKIA[0-9A-Z]{16}`; `-----BEGIN (RSA \|EC )?PRIVATE KEY-----`; `(api_key\|password\|secret\|token)\s*=\s*["'][^"']{8,}["']` (case-insensitive) | sec-scan |
| ForbiddenImports | CodeChangeSet files destined for the application | BLOCK if import not in allowlist (stdlib + fastapi, pydantic, uvicorn, httpx, pytest, starlette) or AST contains `eval`/`exec` calls or `os.system` | sec-scan, supply-chain |
| ProtectedPaths | Every CodeChangeSet | BLOCK if any file path escapes `output/` (after normalization) or matches the scenario's protected list; human override converts to WARN with `override_by` recorded | change-control |
| ChangeBudget | Every CodeChangeSet | BLOCK if `total_files > 15` or `total_loc > 1200` | change-control |
| TestGate | Exit gate of test_run (and regression runs) | FAIL if pytest exit != 0 or `collected < scenario_minimum` | quality-gate |
| ReleasePolicy | Exit gate of release | FAIL if any unresolved BLOCK, or TestGate not green on latest applied version, or a mandatory approval absent from audit log | change-control, quality-gate |

---

## 8. Scenario Specifications

All scenarios share the DAG core (`requirements -> decomposition -> architecture[APPROVAL] -> {codegen_core || codegen_api} -> test_authoring -> test_run[TESTGATE] -> review[APPROVAL] -> docs -> release[APPROVAL]`). Fault injections are declared in the scenario spec and labeled `injected: true` in audit events.

### 8.1 Greenfield — "Build the URL shortener from requirements"

- **Input**: full prose requirement covering FR-URL-1..9.
- **Demonstrates**: full SDLC pipeline; parallel codegen branches with a synchronization join; retry on real test failure; policy block on seeded secret.
- **Fault injections**: (1) codegen_core attempt 1 uses `v_bug` template (off-by-one in base62 decode) -> test_run genuinely fails -> VALIDATION retry with pytest tail as feedback -> attempt 2 `v_fixed` -> green. (2) codegen_api attempt 1 contains `API_KEY = "sk-demo-..."` -> SecretScan BLOCK -> regeneration.
- **Acceptance**: run exits 0 with `--auto-approve --offline`; generated app in `output/` passes its own pytest suite; app boots under uvicorn and serves FR-URL-1/2/4 flows; audit contains RETRY_SCHEDULED, POLICY_CHECK(BLOCK), APPROVAL_GRANTED x3, FINAL_SUMMARY generated.

### 8.2 Brownfield — "Add TTL expiry + click analytics to the existing shortener"

- **Input**: enhancement request against `target_app/` (copied to `output/` at run start); baseline lacks `expires_at`, `clicks`, stats endpoint.
- **Extra stage**: `impact_analysis` (Architect) — real AST scan producing ImpactReport per FR-AGT-5.
- **Demonstrates**: codebase reasoning; dynamic re-planning; change-budget policy block; rollback with regression protection.
- **Fault injections**: (1) post-design scripted requirement revision ("analytics must record referrer") -> RequirementsSpec v2 -> REPLAN_TRIGGERED, architecture + downstream INVALIDATED and re-run. (2) changeset attempt 1 exceeds ChangeBudget -> BLOCK -> targeted patch attempt 2. (3) applied patch attempt fails regression tests (seeded migration bug) -> ROLLBACK from snapshot -> fixed patch -> green including original baseline tests.
- **Acceptance**: run exits 0 with `--auto-approve --offline`; final `output/` passes baseline + new tests; ImpactReport lists `db.py` and `main.py` with reasons; audit contains REPLAN_TRIGGERED and ROLLBACK events; stats endpoint serves recorded clicks with referrer.

### 8.3 Ambiguous — "Make the service handle high traffic"

- **Input**: the vague requirement string above, no quantitative targets.
- **Demonstrates**: ambiguity surfacing, human-directed interpretation, assumption documentation, DAG branching, safe-stop (documented walkthrough).
- **Flow**: RequirementsAnalyst emits AmbiguityReport (clarifying questions + interpretations A: in-process caching of hot redirects [recommended]; B: horizontal scaling with external store [defer — out of prototype scope]; C: measure-first load test). Requirements approval gate presents a numbered menu; chosen interpretation -> RequirementsSpec v2 with `assumptions[]` populated; branch A tasks activate; B/C branches -> SKIPPED.
- **Path A implementation**: LRU cache for code->url lookups, invalidation on delete, load smoke test (200 concurrent redirects; asserts p95 latency under threshold and cache hit count > 0).
- **Acceptance**: run exits 0 with `--auto-approve --offline` (auto selects A); assumptions appear in RequirementsSpec v2, generated docs, and FINAL_SUMMARY; audit shows SKIPPED branch tasks; load smoke test passes.

---

## 9. CLI Specification

```
python run_scenario.py <scenario> [options]
  scenario:        greenfield | brownfield | ambiguous
  --auto-approve   approve all checkpoints as "auto"; pick recommended interpretation
  --offline        force offline agent mode (default)
  --llm            use Claude API (requires ANTHROPIC_API_KEY); falls back offline on exhaustion
  --parallelism N  max concurrent tasks (default 3)

python run_scenario.py stop <run_id>     engage safe-stop via sentinel file
python -m agentic_sdlc.dashboard [--port 8600] [--run <id>|latest]
```

Exit codes: 0 success | 2 failed | 3 safe-stopped | 4 workflow validation error.

---

## 10. State Machine Specification

States: `PENDING, READY, RUNNING, VALIDATING, AWAITING_APPROVAL, DONE, RETRYING, BLOCKED, FAILED, INVALIDATED, ROLLED_BACK, SKIPPED, CANCELLED`.

Legal transitions (complete table; anything absent is illegal and raises):

| From | To | Trigger |
|---|---|---|
| PENDING | READY | deps DONE + entry gates pass |
| PENDING | SKIPPED | branch not selected |
| PENDING | CANCELLED | safe-stop |
| PENDING | BLOCKED | entry policy check BLOCK |
| READY | RUNNING | scheduler dispatch |
| READY | CANCELLED | safe-stop |
| RUNNING | VALIDATING | agent returned artifacts |
| RUNNING | RETRYING | TRANSIENT failure |
| RUNNING | CANCELLED | safe-stop |
| VALIDATING | AWAITING_APPROVAL | exit gates pass, approval declared |
| VALIDATING | DONE | exit gates pass, no approval |
| VALIDATING | RETRYING | VALIDATION failure |
| VALIDATING | BLOCKED | policy BLOCK |
| AWAITING_APPROVAL | DONE | approved |
| AWAITING_APPROVAL | RETRYING | rejected with feedback |
| AWAITING_APPROVAL | CANCELLED | safe-stop |
| RETRYING | RUNNING | backoff elapsed or fallback activated |
| RETRYING | FAILED | attempts + fallback exhausted |
| BLOCKED | RUNNING | audited human override |
| BLOCKED | CANCELLED | safe-stop |
| DONE | INVALIDATED | replan (upstream hash mismatch) |
| DONE | ROLLED_BACK | rollback |
| INVALIDATED | PENDING | re-queue |
| ROLLED_BACK | PENDING | re-queue |

Terminal states for run-completion purposes: DONE, SKIPPED (success); FAILED (failure); CANCELLED (stopped).

---

## 11. Testing Requirements

| Suite | Scope | Key cases |
|---|---|---|
| `tests/test_states.py` | State machine | every legal transition accepted; illegal transitions raise; one audit event per transition |
| `tests/test_engine.py` | Scheduler on toy DAGs (no real agents) | parallel branches + join; bounded retry then success; retry exhaustion -> FAILED; BLOCK halts dependents; semaphore respected |
| `tests/test_context_lineage.py` | Artifacts + lineage | version increment, hash stability, manifest atomicity, DecisionRecord input/output hashes |
| `tests/test_replanner.py` | Re-planning | new artifact version invalidates exact downstream set, transitively; unrelated tasks untouched |
| `tests/test_policy.py` | All 6 rules | secret patterns hit/miss; forbidden import + eval detection; path escape (incl. `..` normalization); budget boundary (15 files/1200 LOC exactly); release policy composition |
| `tests/test_agents_offline.py` | Offline agents | schema-valid artifacts per scenario; AST ImpactReport correctness against target_app fixtures |
| `tests/test_scenarios_smoke.py` | End-to-end | each scenario exits 0 with `--auto-approve --offline`; expected audit events present (RETRY, BLOCK, REPLAN, ROLLBACK, SKIPPED per scenario) |
| `target_app/tests/` | Baseline app | unit: base62 roundtrip, alias validation + reserved, TTL vs fake clock, limiter window math; integration: create->redirect->stats->delete, 409, 410, 429, 404, healthz |

---

## 12. Traceability Matrix (assignment requirement -> specification)

| Assignment core requirement | Specification coverage |
|---|---|
| 1. Requirement understanding | FR-AGT-4 (ambiguity surfacing), RequirementsSpec schema (6.3), scenario 8.3 |
| 2. Task decomposition | Planner (FR-AGT-1), TaskPlan schema, FR-ORCH-1 (dependencies/sequencing) |
| 3. Codebase reasoning (brownfield) | FR-AGT-5 (AST ImpactReport), scenario 8.2 |
| 4a. Explicit dependency graph, entry/exit gates | FR-ORCH-1/2/5/6 |
| 4b. Sequential + parallel with synchronization | FR-ORCH-3 |
| 4c. Cross-stage context + decision lineage | FR-ORCH-15, DecisionRecord (6.4) |
| 4d. Human approval checkpoints | FR-APR-1..5 |
| 4e. Bounded retries, fallback, rollback, safe-stop | FR-ORCH-8..12, FR-ORCH-14 |
| 4f. Policy guardrails (security/compliance/change control) | FR-POL-1..7, Section 7 |
| 4g. Audit-grade observability/traceability | FR-OBS-1..3, NFR-5 |
| 4h. Reliability metrics (success, retries, rollback, MTTR, latency) | FR-OBS-4/5, metrics schema (6.6) |
| 4i. Dynamic re-planning under governance | FR-ORCH-13, scenarios 8.2/8.3 |
| 5. Engineering output generation | FR-AGT-1/6/7, FR-URL-1..9, Section 11 |
| 6. Validation and risk control | Sections 7, 11; NFR-6/8; ENGINEERING_SUMMARY.md |
| 7. Controlled autonomy | FR-APR-1..5, FR-AGT-9, FR-POL-4 (override), ARCHITECTURE.md Section 6 |
| 8. Final engineering summary | FR-OBS-6 |
