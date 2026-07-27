# Implementation Plan — Agentic SDLC Orchestration System (URL Shortener)

## 1. Context

Charles Schwab interview assignment (2–3 day effort): build a working prototype of an **Agentic Software Engineering System** that automates the Software Development Life Cycle for a **URL shortener service** with controlled autonomy — human oversight, policy guardrails, and explicit approval checkpoints.

The **critical differentiator** is the workflow orchestration layer: non-linear stateful execution over an explicit dependency graph with entry/exit gates, parallel paths with synchronization, cross-stage context + decision lineage, bounded retries/fallback/rollback/safe-stop, audit-grade observability, reliability metrics (success rate, retries, rollbacks, MTTR, latency), and dynamic re-planning when upstream outputs change.

**Deliverables:** runnable prototype, architecture overview, three demo scenarios (greenfield / brownfield / ambiguous), setup instructions, testing approach + limitations + trade-offs.

**Environment:** `C:\Bhupesh\CharlesS`, Windows 10, Python 3.14.0. Verified compatible: fastapi 0.140.7, pydantic 2.13.4, uvicorn, httpx, pytest 9.1.1.

## 2. Confirmed decisions

1. **Python end-to-end** — FastAPI shortener + Python orchestrator.
2. **Hybrid agent brains** — agents call the Claude API when `ANTHROPIC_API_KEY` is set; every agent has a deterministic offline fallback so the full demo runs with zero keys. Offline is the default demo path.
3. **CLI approvals + read-only web dashboard** (live DAG, gates, audit tail, metrics).
4. **Full shortener scope** — shorten/redirect/delete, custom aliases, TTL, click analytics, rate limiting, health checks, SQLite, unit + integration tests.

## 3. Repository structure

```
C:\Bhupesh\CharlesS\
├── README.md  requirements.txt  .env.example  .gitignore
├── run_scenario.py                  # python run_scenario.py greenfield [--auto-approve] [--offline|--llm] | stop <run_id>
├── agentic_sdlc/
│   ├── cli.py
│   ├── orchestrator/                # workflow.py (DSL), states.py, engine.py, context.py,
│   │                                # artifacts.py, gates.py, approvals.py, retry.py,
│   │                                # replanner.py, rollback.py, safestop.py, audit.py, metrics.py
│   ├── agents/                      # base.py + requirements_analyst, planner, architect,
│   │                                # code_generator, test_engineer, reviewer, doc_writer, release_manager
│   ├── llm/                         # client.py (anthropic wrapper), offline.py (template generators)
│   ├── policy/                      # engine.py, rules.py
│   ├── templates/                   # greenfield/ (v_bug + v_fixed variants), brownfield/, ambiguous/, common/
│   └── dashboard/                   # server.py + static/index.html (vanilla JS polling)
├── scenarios/                       # greenfield.py, brownfield.py, ambiguous.py (DAG + fault injections)
├── target_app/                      # hand-built baseline shortener = the brownfield "existing codebase" (+ its tests)
├── workspace/runs/<run_id>/         # gitignored: audit.jsonl, state.json, metrics.json,
│                                    # artifacts/<name>/v<N>/, output/ (generated app), snapshots/, summary/
├── docs/                            # ARCHITECTURE.md, SCENARIOS.md, ENGINEERING_SUMMARY.md, TESTING.md
└── tests/                           # engine/policy/lineage/replan unit tests + 3 scenario smoke tests
```

Key principle: greenfield **generates a real runnable app** into `workspace/runs/<id>/output/` and its pytest suite is actually executed in a subprocess. `target_app/` is a real, tested baseline that brownfield modifies. Nothing is simulated where it can be real.

## 4. Orchestrator engine (the differentiator)

- **DAG declaration = Python DSL** (frozen dataclasses, not YAML — gates are predicates, fault injections are callables). `Task(id, agent, depends_on, consumes, produces, entry_gates, exit_gates, retry, fallback, parallel_group)`. Workflow validates acyclicity + artifact coverage at load.
- **Gate semantics**: entry gates (deps DONE, artifacts exist + hashes current, policy pre-checks, no safe-stop) gate READY. Exit gates run in order: schema validation, policy post-checks, TestGate, ApprovalGate last. Changesets are **staged, validated, then applied** to `output/` (after snapshotting) — this makes rollback trivial.
- **State machine** (`states.py` has explicit `LEGAL_TRANSITIONS`; a single `transition()` function is the only mutation point, one audit event per transition):
  `PENDING -> READY -> RUNNING -> VALIDATING -> AWAITING_APPROVAL -> DONE`, plus `RETRYING`, `BLOCKED` (policy), `FAILED`, `INVALIDATED` (replan), `ROLLED_BACK`, `SKIPPED` (branch not chosen), `CANCELLED` (safe-stop).
- **Concurrency**: asyncio, single process (I/O-bound work; no locks needed; Windows Proactor loop supports async subprocess for pytest). Semaphore bounds parallelism (3). Synchronization = DAG join nodes. `input()` and copytree via `asyncio.to_thread`. No `add_signal_handler` on Windows — safe-stop via KeyboardInterrupt + `STOP` sentinel file checked each scheduler cycle (`run_scenario.py stop <run_id>` from a second terminal).
- **Context/lineage**: immutable versioned artifacts (`artifacts/<name>/v<N>/` + sha256 + `manifest.json` written atomically via temp + `os.replace`). Pydantic schemas for structured artifacts (RequirementsSpec, TaskPlan, ArchitectureDoc, ImpactReport, CodeChangeSet, TestReport, ReviewReport, AmbiguityReport, ReleaseChecklist) — double as LLM output validators. A `DecisionRecord` per attempt links consumed artifact-versions (with hashes) to produced ones + rationale + mode.
- **Failure classification drives retry**: TRANSIENT (LLM timeout/429, subprocess timeout) -> exponential backoff + jitter; VALIDATION (schema fail, test fail) -> immediate retry with failure output injected as feedback input; POLICY_BLOCK -> BLOCKED awaiting human; FATAL -> FAILED. After LLM attempts are exhausted -> one offline-fallback attempt (`FALLBACK_ACTIVATED` audit event).
- **Rollback**: restore `output/` from pre-apply snapshot + retract artifact version (manifest points back to v(N-1)) + invalidate downstream consumers via lineage.
- **Re-planning** (`replanner.py`): any artifact gaining a new current version -> walk DecisionRecords, find tasks whose recorded input hash no longer matches -> `DONE -> INVALIDATED -> PENDING` transitively -> scheduler re-executes the subgraph. Emits `REPLAN_TRIGGERED`. Branch selection (ambiguous scenario) = same machinery + `SKIPPED` for unchosen branches.
- **Approvals**: gated at requirements sign-off (ambiguous), design, code review, release. CLI shows artifact summaries, diff stats, policy results table, lineage note. Options: `[a]pprove [r]eject+feedback [v]iew [s]afe-stop`; rejection feedback becomes a new artifact version -> triggers replan. `--auto-approve` for CI/smoke tests (approver recorded as "auto"). ASCII-only console output.
- **Audit**: `audit.jsonl` append-only, one JSON object per line: `{seq, ts, run_id, correlation_id, event_type, payload}` — every transition, gate evaluation, policy check, approval, artifact, retry, fallback, rollback, replan, safe-stop. Flushed per event.
- **Metrics** (`metrics.py`, single fold over audit.jsonl): success rate, retry/fallback/rollback counts, policy blocks, approval latency, **MTTR** (first failure -> subsequent DONE), per-stage + end-to-end latency. Written to `metrics.json`, shown in dashboard + final CLI summary.

## 5. Agents

`AgentBase.run(ctx)`: load inputs (hash-recorded) -> policy precheck -> `generate()` (LLM if enabled, else/fallback offline) -> materialize + schema-validate artifacts -> policy postcheck -> record DecisionRecord. LLM client: anthropic SDK, JSON-mode prompt -> pydantic validate -> one repair round-trip on validation error -> exceptions classified TRANSIENT/FATAL.

Offline mode is **not** all-fake — honest computation where possible:

| Agent | Real computation in offline mode |
|---|---|
| Architect | Real AST scan of the codebase for brownfield `ImpactReport` (routes, functions, imports, data flows) |
| TestEngineer | Actually runs `sys.executable -m pytest` (async subprocess, cwd=`output/`, 120s timeout), parses a real TestReport |
| Reviewer | Runs `py_compile` + AST import checks on generated code |
| ReleaseManager | Computes the release checklist from actual audit state |

Creative artifacts (requirements/design/docs/code) come from parametrized templates keyed by scenario, with `v_bug`/`v_fixed` variants for fault injection. Framed openly in ENGINEERING_SUMMARY.md.

## 6. Policy guardrails (`policy/rules.py`)

Each rule -> `PolicyResult(rule_id, PASS|WARN|BLOCK, details)` + `POLICY_CHECK` audit event with compliance tags:

1. **SecretScan** — regex battery (AWS keys, PEM headers, hardcoded key/password assignments) over generated files -> BLOCK.
2. **ForbiddenImports** — AST allowlist (stdlib + fastapi/pydantic/uvicorn/httpx/pytest/starlette); `eval`/`exec`/`os.system` in generated app code -> BLOCK.
3. **ProtectedPaths** — writes only inside run `output/`; protected files require explicit human override (audited).
4. **ChangeBudget** — max 15 files / 1200 LOC per changeset -> BLOCK with "split the change" guidance.
5. **TestGateRule** — pytest exit 0 AND collected tests >= scenario minimum.
6. **ReleasePolicy** — no unresolved BLOCKs + TestGate green on latest code + all mandatory approvals present.

## 7. URL shortener (target app)

FastAPI + stdlib `sqlite3` (no ORM). Endpoints: `POST /api/links` (url, custom_alias?, ttl_seconds? -> 201/409/422), `GET /{code}` (307 + click recorded / 404 / 410 expired), `GET/DELETE /api/links/{code}`, `GET /api/links/{code}/stats` (total, last_clicked, by-day), `GET /api/links` (paged), `GET /healthz`.

Schema: `links(id, code UNIQUE, url, is_custom, created_at, expires_at, deleted)` + `clicks(id, link_id, clicked_at, referrer)` (clicks + expires_at are what brownfield adds).

- **Base62 of `rowid + 100_000`** — no collision handling needed; custom alias `^[A-Za-z0-9_-]{3,32}$` + reserved set {api, healthz, docs, admin}.
- **TTL** = lazy expiry with injectable clock (testable, no freezegun).
- **Rate limiting** = in-memory fixed window per IP, 20/min on create, 429 + Retry-After (documented single-process trade-off).
- DB path from env; sync endpoints + per-request connection.
- **Tests**: pytest + httpx `ASGITransport` — unit (base62, alias, TTL clock, limiter math) + integration (create->redirect->stats->delete, 409, 410, 429, healthz, 404).

Baseline `target_app/` = everything except TTL/analytics (those arrive via brownfield). Greenfield templates generate the full set.

## 8. Three scenarios

Scripted fault injections are declared data (`payload.injected: true` in audit) — honest, deterministic demo choreography.

Shared DAG core:
`requirements -> decomposition -> architecture(APPROVAL) -> {codegen_core || codegen_api} -> test_authoring -> test_run(TestGate) -> review(APPROVAL) -> docs -> release(APPROVAL)`

1. **Greenfield — "Build the URL shortener from requirements"**: full pipeline, parallel codegen branches joined at test_authoring. Fault 1: `v_bug` template (off-by-one in base62 decode) -> real pytest failure -> retry with feedback -> `v_fixed` -> green. Fault 2: seeded `API_KEY="sk-demo..."` in attempt 1 -> SecretScan BLOCK -> regenerate clean.
2. **Brownfield — "Add TTL + click analytics to target_app"**: extra `impact_analysis` stage — real AST scan -> ImpactReport (impacted modules/endpoints/data flows/regression risks). Fault 1: scripted mid-run requirement revision ("record referrer") -> RequirementsSpec v2 -> **REPLAN** invalidates architecture + downstream, dashboard shows DONE->INVALIDATED. Fault 2: oversized changeset -> ChangeBudget BLOCK -> targeted patch; then seeded migration bug fails regression tests -> **ROLLBACK** from snapshot -> fixed patch -> green including original baseline tests.
3. **Ambiguous — "Make the service handle high traffic"**: RequirementsAnalyst emits AmbiguityReport (clarifying questions + 3 interpretations: A cache hot redirects, B horizontal scaling [defer], C measure-first). **Approval gate = numbered interpretation menu** (auto-approve picks recommended A) -> branch A activates, B/C `SKIPPED`; assumptions written into RequirementsSpec v2 and flow to docs. Path A: LRU cache + invalidation on delete + load smoke test (200 concurrent redirects, p95 + cache-hit assertions). Safe-stop demo documented here.

Every run ends with a generated `FINAL_SUMMARY.md` from real run data (decisions, approvals, policy results, metrics, assumptions, limitations).

## 9. Dashboard

Standalone `python -m agentic_sdlc.dashboard` (port 8600), **read-only over run files** (engine writes state.json/metrics.json atomically; dashboard tails audit.jsonl by seq — works live and post-run). Single `index.html`, vanilla JS polling 1.5s, 4 panels: DAG as topological columns with state chips (+ simple SVG edges if time allows), gates/approvals, audit tail, metrics cards. No React/npm — zero-install for evaluators, no risk to the engine.

## 10. Docs

README (setup/quickstart/flags/demo commands), ARCHITECTURE.md (component + DAG + state diagrams, gate semantics, lineage model, policy catalog, concurrency + Windows notes), SCENARIOS.md (per-scenario walkthrough: input, faults, what to watch, approval script), ENGINEERING_SUMMARY.md (rationale, risks/trade-offs, assumptions, limitations, production-hardening path), TESTING.md.

## 11. Build order

Always demoable; cut lines if time runs short (in order): SVG DAG edges -> LLM mode -> dashboard niceties. Never cut: engine tests, scenario smoke tests, README.

| Phase | Scope | Exit criterion |
|---|---|---|
| 0 | git init, venv, pinned deps, skeleton, states.py + audit.py + tests | test_states green |
| 1 | DSL, engine scheduler, context/artifacts/lineage, retry, non-approval gates | toy DAG retries through injected failure |
| 2 | approvals CLI + auto-approve, policy engine + 6 rules, safe-stop, replanner | replan test: artifact mutation invalidates downstream |
| 3 | target_app baseline + tests; greenfield templates (v_bug/v_fixed) | baseline pytest green standalone |
| 4 | agents offline + greenfield end-to-end | generated app's tests pass in subprocess |
| 5 | brownfield + ambiguous scenarios | all 3 smoke tests green `--auto-approve --offline` |
| 6 | dashboard | live run visible in browser |
| 7 | LLM mode + fallback test (fake failing client) | runs with key; falls back cleanly without |
| 8 | docs + final-summary polish + demo dry-runs | README quickstart works clean |

## 12. Verification

- `pytest tests/` — engine unit tests (states, toy DAGs with parallel+retry+block, lineage, replanner, policy) + `test_scenarios_smoke.py` running all three scenarios end-to-end with `--auto-approve --offline`.
- `pytest target_app/tests/` — baseline green before brownfield touches it.
- Manual demo pass: `python run_scenario.py greenfield` (interactive approvals) with dashboard open — verify retry, policy block, approval, metrics render live; brownfield -> observe REPLAN + ROLLBACK events; ambiguous -> interpretation menu branches the DAG; Ctrl+C / `stop` command -> CANCELLED states + SAFE_STOP event.
- Generated greenfield app boots: `uvicorn` from `workspace/runs/<id>/output/`, hit `POST /api/links` -> `GET /{code}` redirect -> stats.
- Fresh-clone check of README quickstart.

## 13. Key risks and mitigations

| Risk | Mitigation |
|---|---|
| Python 3.14 wheel availability | Verified installed/importable already (fastapi, pydantic, uvicorn, httpx, pytest) |
| Windows console cp1252 encoding | ASCII-only CLI, `PYTHONIOENCODING=utf-8` bootstrap, explicit `encoding="utf-8"` on all file I/O |
| asyncio on Windows | Keep Proactor loop, no signal handlers, sentinel-file safe-stop, `input()` in `to_thread` |
| Dashboard/engine file contention | Atomic replace writes, tolerate partial JSONL last line |
| pytest subprocess flakiness | `sys.executable -m pytest`, explicit cwd/env, 120s timeout classified TRANSIENT |
| Scope creep | Pre-agreed cut lines; no git-based changesets, no run-resume, no websockets, no auth, no coverage% (all documented trade-offs) |
