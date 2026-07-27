# System Specifications — Agentic SDLC Orchestration System (URL Shortener)

**Version 1.1** | **Stack: C# 14 / .NET 10** | Status: Baseline for implementation
Companion documents: `../PLAN.md` (implementation plan), `ARCHITECTURE.md` (architecture overview)

> Change from v1.0: the implementation stack moved from Python to .NET. All requirement IDs are unchanged so existing traceability holds; requirement *content* is re-targeted to the platform, and the concurrency, codebase-reasoning, and policy-enforcement requirements changed substantively (FR-ORCH-3/14, FR-AGT-5/6/7, FR-POL-3).

---

## 1. Purpose and Scope

This document specifies the functional and non-functional requirements, external interfaces, data contracts, and acceptance criteria for:

1. **The Agentic SDLC Orchestration System** — a governed workflow engine that coordinates specialized AI agents through the software development lifecycle (requirements, decomposition, architecture, implementation, testing, documentation, release) under controlled autonomy.
2. **The Target Application** — a URL shortener service that the orchestration system builds (greenfield), enhances (brownfield), and hardens against an ambiguous requirement.

**Out of scope** (documented trade-offs, not deficiencies): distributed/multi-node execution, run resume after safe-stop, authentication/authorization on service endpoints, git-based change management inside the engine, code-coverage-percentage gates.

## 2. Definitions

| Term | Definition |
|---|---|
| Run | One end-to-end execution of a scenario workflow, identified by `runId` |
| Task | A node in the workflow DAG, executed by exactly one agent |
| Gate | A predicate that must pass to enter (entry gate) or leave (exit gate) a task |
| Artifact | An immutable, versioned, SHA-256-hashed output of a task attempt |
| Changeset | An artifact listing files and contents to be applied to the application directory |
| Apply | Copying a validated changeset into the run's `output/` directory after snapshotting |
| DecisionRecord | Lineage record linking consumed artifact versions (by hash) to produced ones |
| State actor | The single-consumer `Channel<StateCommand>` loop that is the sole writer of task state, the artifact manifest, and the audit log |
| Fault injection | A scenario-declared, audit-labelled deterministic failure used for demonstration |
| Offline mode | Deterministic agent generation with no model-API or network dependency |

---

## 3. Functional Requirements

Requirement IDs are stable and referenced by tests and the traceability matrix (Section 12).

### 3.1 Orchestration Engine (FR-ORCH)

| ID | Requirement |
|---|---|
| FR-ORCH-1 | Workflows SHALL be declared as an explicit dependency graph (DAG) of tasks using the C# record DSL, with `DependsOn`, `Consumes`, `Produces`, entry gates, exit gates, retry profile, and optional fallback mode. |
| FR-ORCH-2 | The engine SHALL validate the workflow at load time: acyclicity (topological sort), every consumed artifact produced upstream, every referenced agent key registered in the DI container. Validation failure SHALL abort the run before any task executes (exit code 4). |
| FR-ORCH-3 | The engine SHALL support sequential and parallel execution paths with synchronization at join nodes. Concurrent task execution SHALL be bounded by a configurable `SemaphoreSlim` limit (default 3), and joins SHALL be implemented as `Task.WhenAll` over dependency completion. |
| FR-ORCH-4 | Each task SHALL follow the state machine in Section 10. **All** state changes SHALL be enqueued as commands to the single-consumer state actor, which is the only component permitted to mutate task state, the artifact manifest, or the audit log. The actor SHALL validate every transition against the legal-transition table and SHALL throw on an illegal transition. No locks SHALL be required for state consistency. |
| FR-ORCH-5 | Entry gates SHALL verify: all dependencies Done, required artifacts exist and are current (content hash matches lineage expectation), policy pre-checks pass, safe-stop not engaged. |
| FR-ORCH-6 | Exit gates SHALL execute in fixed order: schema validation, policy post-checks, test gate (if declared), approval gate (if declared) last. |
| FR-ORCH-7 | Changesets SHALL be staged in an attempt directory and applied to `output/` only after all exit gates pass, and only after snapshotting the current `output/` state. Agents SHALL NOT write to `output/` directly (see FR-AGT-9). |
| FR-ORCH-8 | Failures SHALL be classified `Transient`, `Validation`, `PolicyBlock`, or `Fatal`, with class-specific handling per FR-ORCH-9..11. |
| FR-ORCH-9 | `Transient` failures SHALL be retried via a Polly `ResiliencePipeline` with bounded attempts (default 3), exponential backoff, and jitter. `Validation` failures SHALL be retried immediately with the failure output (compiler diagnostics, test output, or schema errors) injected as a feedback input to the next attempt. |
| FR-ORCH-10 | `PolicyBlock` SHALL transition the task to Blocked with no automatic retry; only an explicit, audited human override or safe-stop may release it. |
| FR-ORCH-11 | When model-backed attempts are exhausted and an offline generator exists for the agent, the engine SHALL make one fallback attempt and emit `FALLBACK_ACTIVATED`. |
| FR-ORCH-12 | Rollback SHALL restore `output/` from the pre-apply snapshot, retract the artifact version (the current pointer returns to the prior version), transition the task to RolledBack, and invalidate downstream consumers via lineage. |
| FR-ORCH-13 | When any artifact gains a new current version, the replanner SHALL invalidate exactly those completed tasks whose recorded input hashes no longer match, transitively, and re-queue them (Done → Invalidated → Pending). A `REPLAN_TRIGGERED` event SHALL record cause and affected tasks. |
| FR-ORCH-14 | Safe-stop SHALL be triggerable by (a) `Ctrl+C`, cancelling the root `CancellationTokenSource` that is propagated through every asynchronous call path, and (b) a `STOP` sentinel file created by the `stop <runId>` command from another process, polled each scheduler cycle. On stop: no new dispatch, in-flight tasks cancelled at await points, non-terminal tasks → Cancelled, `SAFE_STOP` emitted, state flushed. A stopped run SHALL never leave `output/` partially applied. |
| FR-ORCH-15 | Cross-stage context SHALL be preserved via the artifact store. Every completed attempt SHALL produce a DecisionRecord capturing consumed artifact versions (with SHA-256), produced artifacts, rationale, and generation mode. |

### 3.2 Agents (FR-AGT)

| ID | Requirement |
|---|---|
| FR-AGT-1 | The system SHALL provide eight role agents — RequirementsAnalyst, Planner, Architect, CodeGenerator, TestEngineer, Reviewer, DocWriter, ReleaseManager — sharing a common `AgentBase` contract (input load, policy pre-check, generate, materialize + validate, policy post-check, decision recording), registered as keyed DI services and resolved by agent key. |
| FR-AGT-2 | Every agent SHALL operate in LLM mode (when `ANTHROPIC_API_KEY` is set and `--llm` is passed) and offline mode (default). Offline mode SHALL require no network access. |
| FR-AGT-3 | Model outputs SHALL be validated against the target artifact's C# record schema, where the JSON schema supplied to the model is generated from that same record via `JsonSchemaExporter`. On validation failure the client SHALL attempt exactly one repair round-trip before the attempt is classified as a `Validation` failure. |
| FR-AGT-4 | RequirementsAnalyst SHALL, for ambiguous inputs, produce an AmbiguityReport containing clarifying questions and at least 3 ranked interpretations with scope, risks, and assumptions, exactly one of which is marked recommended. |
| FR-AGT-5 | Architect SHALL, in brownfield scenarios, produce an ImpactReport derived from **Roslyn semantic analysis** of the existing project: symbols resolved through a `Compilation`, references and call sites located via the semantic model, yielding impacted types/members, impacted endpoints, data-flow changes, and regression risks. Text or syntax-only matching SHALL NOT be the basis of the report. |
| FR-AGT-6 | TestEngineer SHALL execute the generated application's test suite by invoking `dotnet test` as an asynchronous subprocess with an isolated working directory and environment, and a 180-second timeout (timeout classified `Transient`). The TestReport SHALL be parsed from the emitted TRX report, falling back to exit code plus stdout tail if the TRX file is unavailable. |
| FR-AGT-7 | Reviewer SHALL perform real static verification of generated code: an in-memory `CSharpCompilation.Emit()` producing compiler diagnostics with severity and source locations, plus a semantic banned-API walk (see FR-POL-3). |
| FR-AGT-8 | ReleaseManager SHALL compute the ReleaseChecklist from actual run state (the audit log): gates green, tests passed on the latest applied code version, mandatory approvals present. |
| FR-AGT-9 | Agents SHALL write only within their attempt directory. Direct writes to `output/` are prohibited; changesets are applied exclusively by the engine per FR-ORCH-7. |

### 3.3 Policy and Governance (FR-POL)

| ID | Requirement |
|---|---|
| FR-POL-1 | Every policy evaluation SHALL produce a verdict `Pass`, `Warn`, or `Block` and SHALL be recorded as a `POLICY_CHECK` audit event with rule id, verdict, details, and compliance tags. |
| FR-POL-2 | **SecretScan**: generated file contents SHALL be scanned for credential patterns (cloud access keys, PEM private-key headers, hardcoded `apikey`/`password`/`secret`/`token` assignments). Any hit → Block. |
| FR-POL-3 | **ForbiddenApis**: generated application code SHALL be checked by a Roslyn **semantic** walk against a banned-symbol list, and its project file against a NuGet package allowlist. Use of a banned symbol or a non-allowlisted package reference → Block. Detection SHALL use resolved symbols, not identifier text, so aliasing or fully-qualified spelling cannot evade it. |
| FR-POL-4 | **ProtectedPaths**: changesets SHALL only write inside the run's `output/`, verified after `Path.GetFullPath` normalization so relative traversal is caught. Declared protected files within it SHALL require an explicit human override, which is itself audited with the overriding identity. |
| FR-POL-5 | **ChangeBudget**: changesets exceeding 15 files or 1200 changed LOC → Block, with guidance to split the change. |
| FR-POL-6 | **TestGate**: code-affecting stages SHALL not pass their exit gate unless `dotnet test` exits 0 and the collected test count is at least the scenario minimum. |
| FR-POL-7 | **ReleasePolicy**: the release gate SHALL require zero unresolved Blocks, TestGate green on the latest applied code version, and all mandatory approvals present in the audit log. |

### 3.4 Approvals and Controlled Autonomy (FR-APR)

| ID | Requirement |
|---|---|
| FR-APR-1 | Human approval checkpoints SHALL gate design, code review, and release in all scenarios, plus requirements sign-off (interpretation selection) in the ambiguous scenario. |
| FR-APR-2 | The approval prompt SHALL present: task id and attempt, artifact summaries, changeset diff statistics (files, LOC added/removed), a policy check results table, and a lineage note naming consumed artifact versions. |
| FR-APR-3 | Approver actions SHALL be: approve; reject with feedback (the feedback becomes a new artifact version and triggers re-planning per FR-ORCH-13); view the full artifact; safe-stop. Where an interpretation choice is required, the prompt SHALL be a selection menu over the AmbiguityReport interpretations. |
| FR-APR-4 | Every approval decision SHALL be audited with approver identity, decision, feedback text, timestamp, and the SHA-256 of the approved content. |
| FR-APR-5 | An `--auto-approve` mode SHALL exist for CI and smoke tests, recording approver `auto` and selecting the analyst-recommended interpretation where a choice is required. |

### 3.5 Observability (FR-OBS)

| ID | Requirement |
|---|---|
| FR-OBS-1 | The engine SHALL maintain an append-only `audit.jsonl` with one JSON object per line: `{seq, ts, runId, correlationId, eventType, payload}`. Every event SHALL be flushed on write, and writes SHALL originate only from the state actor, guaranteeing ordering and completeness. |
| FR-OBS-2 | Audited event types SHALL include: `RUN_STARTED`/`COMPLETED`/`FAILED`, `TASK_STATE_CHANGED`, `GATE_EVALUATED`, `POLICY_CHECK`, `APPROVAL_REQUESTED`/`GRANTED`/`REJECTED`, `ARTIFACT_CREATED`, `DECISION_RECORDED`, `RETRY_SCHEDULED`, `FALLBACK_ACTIVATED`, `ROLLBACK`, `REPLAN_TRIGGERED`, `SAFE_STOP`. |
| FR-OBS-3 | Injected demo faults SHALL be labelled `payload.injected: true` in every related audit event. |
| FR-OBS-4 | Reliability metrics SHALL be derived solely from the audit log: task success rate, retry count and rate, fallback count, rollback count, policy blocks, approval latency, MTTR (first failure → subsequent Done per recovered task), per-stage latency (Ready → Done), and end-to-end wall time. |
| FR-OBS-5 | Metrics SHALL be written to `metrics.json` via atomic replace after every state transition and included in the final CLI summary. Equivalent `System.Diagnostics.Metrics` instruments SHALL be published so the same signals are exportable via OpenTelemetry without engine changes. |
| FR-OBS-6 | Every run SHALL end by generating `summary/FINAL_SUMMARY.md` from real run data: plan and rationale, artifacts produced, decisions and approvals, policy results, metrics, assumptions, and limitations. |

### 3.6 CLI (FR-CLI)

| ID | Requirement |
|---|---|
| FR-CLI-1 | `dotnet run --project src/AgenticSdlc.Cli -- run <greenfield\|brownfield\|ambiguous> [--auto-approve] [--offline\|--llm] [--parallelism N]` SHALL execute the named scenario. `--offline` is the default mode. |
| FR-CLI-2 | `dotnet run --project src/AgenticSdlc.Cli -- stop <runId>` SHALL engage safe-stop for a running run via the sentinel file. |
| FR-CLI-3 | CLI output SHALL use Spectre.Console rendering and SHALL end with a summary containing per-task final states, metrics, artifact locations, and the path to `FINAL_SUMMARY.md`. |
| FR-CLI-4 | Process exit codes: 0 = run completed (all terminal tasks Done or Skipped); 2 = run failed; 3 = safe-stopped; 4 = workflow validation error. |

### 3.7 Dashboard (FR-DASH)

| ID | Requirement |
|---|---|
| FR-DASH-1 | The dashboard SHALL run as a separate read-only process (`dotnet run --project src/AgenticSdlc.Dashboard`, default port 8600) with zero shared state with the engine — file-based access only. |
| FR-DASH-2 | It SHALL display: the DAG with per-task state chips, a gates/approvals panel (pending approvals highlighted), an audit tail (last 50 events, incrementally fetched by `seq`), and metrics cards. |
| FR-DASH-3 | It SHALL function both during a live run (poll interval 1.5 s) and after run completion (post-run inspection and replay). |
| FR-DASH-4 | It SHALL tolerate concurrent engine writes: snapshot files are read after atomic replace, and a partially written final JSONL line SHALL be skipped rather than cause an error. |

### 3.8 URL Shortener Service (FR-URL)

| ID | Requirement |
|---|---|
| FR-URL-1 | `POST /api/links` SHALL create a short link from `{url, customAlias?, ttlSeconds?}`, returning 201 with `{code, shortUrl, expiresAt}`; 409 if the alias is taken; 422 for a semantically invalid URL, alias, or TTL. |
| FR-URL-2 | `GET /{code}` SHALL return a 307 redirect to the original URL (`TypedResults.Redirect(url, permanent: false, preserveMethod: true)`) and record a click (timestamp, referrer); 404 for unknown or soft-deleted codes; 410 for expired links. |
| FR-URL-3 | `GET /api/links/{code}` SHALL return link metadata; `DELETE /api/links/{code}` SHALL soft-delete (204); `GET /api/links` SHALL list links with paging. |
| FR-URL-4 | `GET /api/links/{code}/stats` SHALL return `{totalClicks, lastClickedAt, clicksByDay}`. |
| FR-URL-5 | Generated codes SHALL be base62 of `rowid + 100000` (≥ 4 characters, collision-free by construction). Custom aliases SHALL match `^[A-Za-z0-9_-]{3,32}$` and SHALL NOT be in the reserved set {`api`, `healthz`, `openapi`, `admin`}. |
| FR-URL-6 | TTL SHALL be enforced lazily at redirect time against an injected `TimeProvider`; a `PurgeExpired()` maintenance method SHALL exist. Tests SHALL control time via `FakeTimeProvider`. |
| FR-URL-7 | `POST /api/links` SHALL be rate-limited per client IP using the built-in `RateLimiter` middleware with a `FixedWindowLimiter` (default 20 per minute), returning 429 with a `Retry-After` header. |
| FR-URL-8 | `GET /healthz` SHALL be served by the built-in health-check middleware with a SQLite probe, returning service and database status as JSON. |
| FR-URL-9 | Persistence SHALL use `Microsoft.Data.Sqlite` (ADO.NET, no ORM). The database path SHALL be configurable via the `Shortener__DbPath` environment variable, bound to `Shortener:DbPath` through `IOptions<ShortenerOptions>`. |

---

## 4. Non-Functional Requirements (NFR)

| ID | Requirement |
|---|---|
| NFR-1 | **Portability**: runs on Windows 10 with .NET SDK 10.0.302. A `global.json` SHALL pin the SDK version (`rollForward: latestFeature`) because older SDKs are present on target machines. `Directory.Build.props` SHALL set `net10.0`, nullable reference types enabled, and warnings as errors. |
| NFR-2 | **Offline demo**: all three scenarios SHALL complete end-to-end with `--auto-approve --offline` and no network access, once NuGet packages are restored. Target wall time under 8 minutes per scenario, reflecting compile-and-test cycles. |
| NFR-3 | **Dependencies**: limited to `Spectre.Console`, `Microsoft.Extensions.Resilience` (Polly v8), `Microsoft.CodeAnalysis.CSharp`, `Microsoft.Data.Sqlite`, `Microsoft.Extensions.AI`, the `Microsoft.Extensions.*` hosting/DI/configuration packages, and for tests `xunit.v3`, `Microsoft.AspNetCore.Mvc.Testing`, `Microsoft.Extensions.TimeProvider.Testing`. ASP.NET Core itself comes from the shared framework. The **generated** application SHALL reference only `Microsoft.Data.Sqlite` beyond the shared framework. |
| NFR-4 | **Determinism**: offline runs are reproducible; fault injections are declared data, not random. |
| NFR-5 | **Auditability**: system behavior SHALL be fully reconstructible from `audit.jsonl` alone, and all metrics recomputable offline from that log. |
| NFR-6 | **Crash safety**: dashboard-visible files written via temp file + atomic replace; audit appends flushed per event; `output/` never torn because apply occurs only post-gate and snapshot-first. |
| NFR-7 | **Testability**: engine logic unit-testable with toy DAGs (no real agents, no model calls); the target application testable in-process via `WebApplicationFactory<Program>` with no live server; time controlled by `FakeTimeProvider`. |
| NFR-8 | **Security of generated code**: enforced mechanically by policy rules FR-POL-2/3/4 and by compilation, not by convention. |

---

## 5. API Specification — URL Shortener

Base URL: `http://localhost:5000` (Kestrel default; port configurable). Content type: `application/json` with camelCase property naming (`System.Text.Json` web defaults). Errors return **RFC 7807 ProblemDetails** (`AddProblemDetails()`): `{type, title, status, detail}`.

Status-code convention: **400** for a malformed request body (framework-level model binding failure); **422** for a well-formed body that fails semantic validation (our rules). This distinction is deliberate and asserted in tests.

### 5.1 POST /api/links — create short link

Request body:
```json
{
  "url": "https://example.com/some/long/path",
  "customAlias": "my-link",
  "ttlSeconds": 86400
}
```
`url` is required, http(s), max 2048 characters. `customAlias` is optional, `^[A-Za-z0-9_-]{3,32}$`, not reserved. `ttlSeconds` is optional and must be > 0; omitted means the link never expires.

Responses:

| Status | Condition | Body |
|---|---|---|
| 201 | Created | `{"code":"e9a1","shortUrl":"http://localhost:5000/e9a1","url":"...","expiresAt":"2026-07-28T12:00:00Z"\|null,"createdAt":"..."}` |
| 409 | Custom alias already in use | ProblemDetails, title `alias already exists` |
| 422 | Invalid URL, alias format, reserved alias, or non-positive TTL | ProblemDetails with validation errors |
| 429 | Rate limit exceeded | ProblemDetails + `Retry-After` header (seconds) |

### 5.2 GET /{code} — redirect

| Status | Condition | Behavior |
|---|---|---|
| 307 | Active link | `Location: <original url>`; click recorded (UTC timestamp, referrer header if present) |
| 404 | Unknown or soft-deleted code | ProblemDetails |
| 410 | Link expired (`expiresAt` ≤ now per injected `TimeProvider`) | ProblemDetails; no click recorded |

### 5.3 GET /api/links/{code} — metadata

200: `{"code","url","isCustom","createdAt","expiresAt","deleted","totalClicks"}` | 404 unknown.

### 5.4 DELETE /api/links/{code} — soft delete

204 on success (idempotent if already deleted) | 404 unknown code.

### 5.5 GET /api/links/{code}/stats — analytics

200:
```json
{
  "code": "e9a1",
  "totalClicks": 42,
  "lastClickedAt": "2026-07-27T15:04:05Z",
  "clicksByDay": [{"day": "2026-07-26", "clicks": 30}, {"day": "2026-07-27", "clicks": 12}]
}
```
404 unknown code. Stats remain readable for soft-deleted links.

### 5.6 GET /api/links — list

Query parameters: `limit` (default 50, max 200), `offset` (default 0), `includeDeleted` (default false).
200: `{"items":[<metadata>...],"total":N,"limit":L,"offset":O}`.

### 5.7 GET /healthz

200: `{"status":"Healthy","db":"ok"}` | 503: `{"status":"Unhealthy","db":"<error>"}`. Served by the health-check middleware with a custom JSON response writer.

### 5.8 Dashboard API (read-only, port 8600)

| Endpoint | Returns |
|---|---|
| `GET /` | Single-page dashboard HTML |
| `GET /api/runs` | Available run ids, newest first |
| `GET /api/state?run=<id>` | Contents of `state.json` (DAG nodes, states, gates, pending approvals) |
| `GET /api/metrics?run=<id>` | Contents of `metrics.json` |
| `GET /api/audit?run=<id>&afterSeq=N` | Audit events with `seq > N` (incremental tail) |

---

## 6. Data Specifications

### 6.1 URL shortener SQLite schema

Column names remain snake_case per SQL convention; the JSON API surface is camelCase.

```sql
CREATE TABLE links(
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  code TEXT UNIQUE NOT NULL,
  url TEXT NOT NULL,
  is_custom INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL,            -- UTC ISO-8601
  expires_at TEXT NULL,                -- added by the brownfield scenario
  deleted INTEGER NOT NULL DEFAULT 0
);
CREATE TABLE clicks(                   -- added by the brownfield scenario
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
  "mediaType": "application/json | text/markdown",
  "producedBy": {"taskId": "requirements", "attempt": 1, "mode": "llm | offline"},
  "createdAt": "<UTC ISO-8601>"
}
```

Stored immutably under `artifacts/<name>/v<N>/`. `manifest.json` tracks the current (highest non-retracted) version per name and is rewritten atomically via temp file + `File.Move(overwrite: true)`.

### 6.3 Structured artifact schemas

Defined as C# records with source-generated `System.Text.Json` contexts. The same records generate the JSON schema supplied to the model via `JsonSchemaExporter`, so the prompt contract and the validation contract cannot drift (FR-AGT-3).

| Record | Key members |
|---|---|
| `RequirementsSpec` | `ProblemStatement`, `FunctionalRequirements[] {Id, Text, Priority}`, `NonFunctional[]`, `Assumptions[]`, `OpenQuestions[]` |
| `AmbiguityReport` | `ClarifyingQuestions[]`, `Interpretations[] {Id, Title, Scope, Risks[], Assumptions[], Recommended}` |
| `TaskPlan` | `Tasks[] {Id, Title, Description, DependsOn[], EstimatedSize}` |
| `ArchitectureDoc` | `Overview`, `Components[] {Name, Responsibility}`, `Decisions[] {Decision, Rationale}` |
| `ApiContract` | `Endpoints[] {Method, Path, RequestSchema, Responses[]}` |
| `ImpactReport` | `ImpactedTypes[] {File, Symbols[], Reason}`, `ImpactedEndpoints[]`, `DataFlowChanges[]`, `RegressionRisks[]` |
| `CodeChangeSet` | `Files[] {Path, Content, Sha256, Action: Create\|Modify\|Delete, LocDelta}`, `Summary`, `TotalFiles`, `TotalLoc` |
| `TestReport` | `ExitCode`, `Collected`, `Passed`, `Failed`, `Errors`, `DurationSeconds`, `FailureTail` |
| `ReviewReport` | `CompileOk`, `Diagnostics[] {Id, Severity, File, Line, Message}`, `BannedApiViolations[]`, `Verdict: Approve\|RequestChanges` |
| `ReleaseChecklist` | `Items[] {Check, Status: Pass\|Fail, Evidence}`, `ReleaseReady` |

### 6.4 DecisionRecord

```json
{
  "taskId": "architecture",
  "attempt": 2,
  "inputs":  [{"artifact": "requirements_spec", "version": 2, "sha256": "..."}],
  "outputs": [{"artifact": "architecture_doc", "version": 1, "sha256": "..."}],
  "rationale": "<agent-provided reasoning summary>",
  "mode": "offline",
  "policyResults": [{"rule": "secret_scan", "verdict": "Pass"}],
  "ts": "<UTC ISO-8601>"
}
```

### 6.5 Audit event schema

```json
{
  "seq": 143,
  "ts": "2026-07-27T15:04:05.123Z",
  "runId": "greenfield-20260727-150000",
  "correlationId": "codegen_core:2",
  "eventType": "TASK_STATE_CHANGED",
  "payload": {"from": "Running", "to": "Validating", "reason": "...", "injected": false}
}
```
`seq` is strictly increasing per run, assigned by the state actor.

### 6.6 metrics.json

```json
{
  "runId": "...", "status": "running | completed | failed | stopped",
  "tasksTotal": 10, "tasksDone": 7, "tasksFailed": 0,
  "successRateFirstAttempt": 0.7,
  "retries": 2, "fallbacks": 0, "rollbacks": 1, "policyBlocks": 2,
  "approvalLatencySeconds": {"design_approval": 12.4},
  "mttrSeconds": 8.9,
  "stageLatencySeconds": {"requirements": 1.2, "codegen_core": 22.5},
  "endToEndSeconds": 145.0,
  "updatedAt": "<UTC ISO-8601>"
}
```

---

## 7. Policy Rules Specification

| Rule | Applies to | Verdict logic | Compliance tags |
|---|---|---|---|
| SecretScan | Every CodeChangeSet and generated text artifact | Block on: `AKIA[0-9A-Z]{16}`; `-----BEGIN (RSA \|EC )?PRIVATE KEY-----`; `(apikey\|api_key\|password\|secret\|token)\s*=\s*"[^"]{8,}"` (case-insensitive, covers C# property initializers and const assignments) | sec-scan |
| ForbiddenApis | CodeChangeSet files destined for the application, plus the generated `.csproj` | Block if the Roslyn semantic model resolves a call to a banned symbol — `System.Diagnostics.Process.Start`, `System.Reflection.Emit.*`, `System.Runtime.InteropServices.DllImportAttribute`, `Assembly.Load*` — or if a `PackageReference` is outside the allowlist (shared framework + `Microsoft.Data.Sqlite`). Symbol-based, so `using` aliases and fully-qualified names are both caught | sec-scan, supply-chain |
| ProtectedPaths | Every CodeChangeSet | Block if any file path escapes `output/` after `Path.GetFullPath` normalization, or matches the scenario's protected list. A human override converts Block to Warn and records `overrideBy` | change-control |
| ChangeBudget | Every CodeChangeSet | Block if `TotalFiles > 15` or `TotalLoc > 1200` | change-control |
| TestGate | Exit gate of `test_run` and regression runs | Fail if `dotnet test` exit code ≠ 0, or collected tests < the scenario minimum | quality-gate |
| ReleasePolicy | Exit gate of `release` | Fail if any unresolved Block exists, TestGate is not green on the latest applied version, or a mandatory approval is absent from the audit log | change-control, quality-gate |

---

## 8. Scenario Specifications

All scenarios share the DAG core: `requirements → decomposition → architecture[APPROVAL] → {codegen_core ∥ codegen_api} → test_authoring → test_run[TESTGATE] → review[APPROVAL] → docs → release[APPROVAL]`. Fault injections are declared in the scenario definition and labelled `injected: true` in audit events.

**Compiled-language constraint (applies to all scenarios):** every injected `v_bug` template MUST compile without errors and fail at the test gate. A template that fails to compile would exercise the wrong failure class. A build-check test asserts that all templates compile (see Section 11).

### 8.1 Greenfield — "Build the URL shortener from requirements"

- **Input**: a full prose requirement covering FR-URL-1..9.
- **Demonstrates**: the full SDLC pipeline; parallel codegen branches with a synchronization join; retry driven by a real test failure; policy block on a seeded secret.
- **Fault injections**: (1) `codegen_core` attempt 1 uses the `v_bug` template — an off-by-one in base62 decode that compiles cleanly and fails real tests → `Validation` retry with the test output as feedback → attempt 2 uses `v_fixed` → green. (2) `codegen_api` attempt 1 contains `const string ApiKey = "sk-demo-..."` → SecretScan Block → regeneration.
- **Acceptance**: run exits 0 with `--auto-approve --offline`; the generated project in `output/` compiles and its `dotnet test` suite passes; the app starts and serves the FR-URL-1/2/4 flows; the audit log contains `RETRY_SCHEDULED`, `POLICY_CHECK` with a Block verdict, three `APPROVAL_GRANTED` events, and `FINAL_SUMMARY.md` is generated.

### 8.2 Brownfield — "Add TTL expiry + click analytics to the existing shortener"

- **Input**: an enhancement request against `TargetApp.UrlShortener`, copied into `output/` at run start. The baseline lacks `expires_at`, the `clicks` table, and the stats endpoint.
- **Extra stage**: `impact_analysis` (Architect) producing a Roslyn-derived ImpactReport per FR-AGT-5.
- **Demonstrates**: codebase reasoning over a semantic model; dynamic re-planning; change-budget policy block; rollback with regression protection.
- **Fault injections**: (1) a scripted post-design requirement revision ("analytics must record referrer") produces RequirementsSpec v2 → `REPLAN_TRIGGERED`, with architecture and downstream tasks Invalidated and re-run. (2) changeset attempt 1 exceeds ChangeBudget → Block → a targeted patch on attempt 2. (3) the applied patch fails regression tests due to a seeded **logic** bug — the INSERT omits the `expires_at` value so TTL reads back null — triggering **rollback** from snapshot, then a corrected patch → green, including the original baseline tests.
- **Acceptance**: run exits 0 with `--auto-approve --offline`; the final `output/` passes baseline plus new tests; the ImpactReport names the data-access and endpoint types with reasons derived from symbol references; the audit log contains `REPLAN_TRIGGERED` and `ROLLBACK`; the stats endpoint returns recorded clicks including referrer.

### 8.3 Ambiguous — "Make the service handle high traffic"

- **Input**: the vague requirement above, with no quantitative targets.
- **Demonstrates**: ambiguity surfacing, human-directed interpretation, assumption documentation, DAG branching, and safe-stop (documented walkthrough).
- **Flow**: RequirementsAnalyst emits an AmbiguityReport with clarifying questions and three interpretations — **A** in-process caching of hot redirects (recommended); **B** horizontal scaling with an external store (defer, out of prototype scope); **C** measure-first load testing. The requirements approval gate presents a selection menu; the choice produces RequirementsSpec v2 with populated `Assumptions[]`, activates branch A, and marks the B and C branches Skipped.
- **Path A implementation**: an LRU cache over code→url lookups with invalidation on delete, plus a load smoke test issuing 200 concurrent redirects and asserting p95 latency under threshold and a non-zero cache-hit count.
- **Acceptance**: run exits 0 with `--auto-approve --offline` (auto-selects A); assumptions appear in RequirementsSpec v2, the generated docs, and `FINAL_SUMMARY.md`; the audit log shows Skipped branch tasks; the load smoke test passes.

---

## 9. CLI Specification

```
dotnet run --project src/AgenticSdlc.Cli -- run <scenario> [options]
  scenario:        greenfield | brownfield | ambiguous
  --auto-approve   approve all checkpoints as "auto"; select the recommended interpretation
  --offline        force offline agent mode (default)
  --llm            use the Claude API (requires ANTHROPIC_API_KEY); falls back offline on exhaustion
  --parallelism N  maximum concurrent tasks (default 3)

dotnet run --project src/AgenticSdlc.Cli -- stop <runId>
dotnet run --project src/AgenticSdlc.Dashboard -- [--port 8600] [--run <id>|latest]
```

Exit codes: 0 success | 2 failed | 3 safe-stopped | 4 workflow validation error.

---

## 10. State Machine Specification

States: `Pending, Ready, Running, Validating, AwaitingApproval, Done, Retrying, Blocked, Failed, Invalidated, RolledBack, Skipped, Cancelled`.

Legal transitions (complete table; anything absent is illegal and throws):

| From | To | Trigger |
|---|---|---|
| Pending | Ready | dependencies Done + entry gates pass |
| Pending | Skipped | branch not selected |
| Pending | Cancelled | safe-stop |
| Pending | Blocked | entry policy check Block |
| Ready | Running | scheduler dispatch |
| Ready | Cancelled | safe-stop |
| Running | Validating | agent returned artifacts |
| Running | Retrying | `Transient` failure |
| Running | Cancelled | safe-stop |
| Validating | AwaitingApproval | exit gates pass, approval declared |
| Validating | Done | exit gates pass, no approval declared |
| Validating | Retrying | `Validation` failure |
| Validating | Blocked | policy Block |
| AwaitingApproval | Done | approved |
| AwaitingApproval | Retrying | rejected with feedback |
| AwaitingApproval | Cancelled | safe-stop |
| Retrying | Running | backoff elapsed or fallback activated |
| Retrying | Failed | attempts and fallback exhausted |
| Blocked | Running | audited human override |
| Blocked | Cancelled | safe-stop |
| Done | Invalidated | replan (upstream hash mismatch) |
| Done | RolledBack | rollback |
| Invalidated | Pending | re-queued |
| RolledBack | Pending | re-queued |

Terminal states for run-completion purposes: Done, Skipped (success); Failed (failure); Cancelled (stopped).

---

## 11. Testing Requirements

All tests are xUnit v3. Engine tests use toy DAGs with stub agents — no model calls and no compilation — so they run in milliseconds.

| Test class | Scope | Key cases |
|---|---|---|
| `StateMachineTests` | Transition table | Every legal transition accepted; illegal transitions throw; exactly one audit event per accepted transition |
| `EngineTests` | Scheduler over toy DAGs | Parallel branches with join; bounded retry then success; retry exhaustion → Failed; a Block halts dependents; `SemaphoreSlim` limit respected under load |
| `StateActorConcurrencyTests` | Actor discipline | Concurrent command producers yield strictly increasing `seq`, no interleaved writes, and a consistent final state |
| `ContextLineageTests` | Artifacts and lineage | Version increment; hash stability; atomic manifest replace; DecisionRecord input/output hashes recorded correctly |
| `ReplannerTests` | Re-planning | A new artifact version invalidates the exact downstream set, transitively; unrelated tasks untouched; identical-content regeneration triggers nothing |
| `PolicyTests` | All six rules | Secret patterns hit and miss; banned API detected through a `using` alias and a fully-qualified call; non-allowlisted `PackageReference` blocked; path escape via `..` caught; budget boundaries (exactly 15 files / 1200 LOC); release policy composition |
| `OfflineAgentTests` | Offline agents | Schema-valid artifacts per scenario; Roslyn ImpactReport correctness against `TargetApp.UrlShortener` fixtures |
| `TemplateCompilationTests` | Fault-injection safety | Every template variant, including every `v_bug`, compiles without errors (enforces the §8 compiled-language constraint) |
| `ScenarioSmokeTests` | End-to-end | Each scenario exits 0 with `--auto-approve --offline`; scenario-specific audit events present (retry, block, replan, rollback, skipped) |
| `TargetApp.UrlShortener.Tests` | Baseline application | Unit: base62 round-trip and monotonic length, alias validation including reserved words, TTL boundaries via `FakeTimeProvider`, rate-limiter window math. Integration via `WebApplicationFactory`: create → redirect → stats → delete, 409, 410, 429, 404, 400-vs-422 distinction, health |

---

## 12. Traceability Matrix

| Assignment core requirement | Specification coverage |
|---|---|
| 1. Requirement understanding | FR-AGT-4 (ambiguity surfacing), `RequirementsSpec` / `AmbiguityReport` (§6.3), scenario §8.3 |
| 2. Task decomposition | Planner (FR-AGT-1), `TaskPlan` (§6.3), FR-ORCH-1 (dependencies and sequencing) |
| 3. Codebase reasoning (brownfield) | FR-AGT-5 (Roslyn semantic ImpactReport), scenario §8.2 |
| 4a. Explicit dependency graph, entry/exit gates | FR-ORCH-1, FR-ORCH-2, FR-ORCH-5, FR-ORCH-6 |
| 4b. Sequential + parallel with synchronization | FR-ORCH-3 |
| 4c. Cross-stage context + decision lineage | FR-ORCH-15, DecisionRecord (§6.4) |
| 4d. Human approval checkpoints | FR-APR-1..5 |
| 4e. Bounded retries, fallback, rollback, safe-stop | FR-ORCH-8..12, FR-ORCH-14 |
| 4f. Policy guardrails (security, compliance, change control) | FR-POL-1..7, §7 |
| 4g. Audit-grade observability and traceability | FR-OBS-1..3, NFR-5 |
| 4h. Reliability metrics (success, retries, rollback, MTTR, latency) | FR-OBS-4, FR-OBS-5, metrics schema (§6.6) |
| 4i. Dynamic re-planning under governance | FR-ORCH-13, scenarios §8.2 and §8.3 |
| 5. Engineering output generation | FR-AGT-1, FR-AGT-6, FR-AGT-7, FR-URL-1..9, §11 |
| 6. Validation and risk control | §7, §11; NFR-6, NFR-8; `ENGINEERING_SUMMARY.md` |
| 7. Controlled autonomy | FR-APR-1..5, FR-AGT-9, FR-POL-4 (audited override), `ARCHITECTURE.md` §6 |
| 8. Final engineering summary | FR-OBS-6 |
