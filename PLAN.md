# Implementation Plan — Agentic SDLC Orchestration System (URL Shortener)

**Stack: C# 14 / .NET 10** | Companion docs: `docs/ARCHITECTURE.md`, `docs/SPECIFICATIONS.md`

## 1. Context

Charles Schwab interview assignment (2–3 day effort): build a working prototype of an **Agentic Software Engineering System** that automates the Software Development Life Cycle for a **URL shortener service** with controlled autonomy — human oversight, policy guardrails, and explicit approval checkpoints.

The **critical differentiator** is the workflow orchestration layer: non-linear stateful execution over an explicit dependency graph with entry/exit gates, parallel paths with synchronization, cross-stage context + decision lineage, bounded retries/fallback/rollback/safe-stop, audit-grade observability, reliability metrics (success rate, retries, rollbacks, MTTR, latency), and dynamic re-planning when upstream outputs change.

**Deliverables:** runnable prototype, architecture overview, three demo scenarios (greenfield / brownfield / ambiguous), setup instructions, testing approach + limitations + trade-offs.

**Environment:** `C:\Bhupesh\CharlesS`, Windows 10. Verified installed: **.NET SDK 10.0.302** (current LTS) with ASP.NET Core runtime 10.0.10. Note: stale 1.1.x/2.1.x SDKs are also present on this machine, so `global.json` pinning is mandatory (see Risks).

## 2. Confirmed decisions

1. **Full .NET end-to-end** — orchestrator, agents, policy engine, CLI, dashboard, and the target URL shortener are all C#/.NET 10. One toolchain, one language, coherent story.
2. **Hybrid agent brains** — agents call the Claude API through `Microsoft.Extensions.AI.IChatClient` when `ANTHROPIC_API_KEY` is set and `--llm` is passed; every agent has a deterministic offline generator so the full demo runs with zero keys and zero network. **Offline is the default demo path.**
3. **Spectre.Console CLI approvals + read-only web dashboard** — a minimal-API dashboard serving one static page that polls run files (not Blazor), preserving the file-based decoupling so it works both live and post-run.
4. **Full shortener scope** — create/redirect/delete, custom aliases, TTL expiry, click analytics, rate limiting, health checks, SQLite persistence, unit + integration tests.

Where the platform provides a capability, we use it rather than hand-rolling: Polly for resilience, ASP.NET Core rate limiting and health checks, `TimeProvider` for clock injection, Roslyn for code analysis. Hand-rolled equivalents would be weaker and would read as not knowing the platform.

## 3. Solution layout

```
C:\Bhupesh\CharlesS\
├── AgenticSdlc.sln
├── global.json                      # pins SDK 10.0.302 (rollForward: latestFeature)
├── Directory.Build.props            # net10.0, nullable enable, TreatWarningsAsErrors, LangVersion latest
├── README.md  .env.example  .gitignore
├── src/
│   ├── AgenticSdlc.Core/            # domain: TaskState + transition table, workflow DSL records,
│   │                                # artifact models, gate abstractions, audit event contracts
│   ├── AgenticSdlc.Engine/          # state actor (Channel), scheduler, gate evaluator, Polly retry,
│   │                                # replanner, rollback, safe-stop, audit writer, metrics folder
│   ├── AgenticSdlc.Agents/          # AgentBase + 8 role agents; Roslyn impact analysis,
│   │                                # in-memory compilation review, dotnet-test runner
│   ├── AgenticSdlc.Policy/          # 6 rules: SecretScan, ForbiddenApis, ProtectedPaths,
│   │                                # ChangeBudget, TestGate, ReleasePolicy
│   ├── AgenticSdlc.Llm/             # IChatClient wiring + Anthropic Messages adapter (HttpClient)
│   │                                # + deterministic offline generators
│   ├── AgenticSdlc.Cli/             # Spectre.Console host + Scenarios/ (greenfield, brownfield, ambiguous)
│   ├── AgenticSdlc.Dashboard/       # minimal API + wwwroot/index.html (vanilla JS polling)
│   └── TargetApp.UrlShortener/      # baseline app = the brownfield "existing codebase"
├── templates/                       # generated-code templates: greenfield/ (v_bug + v_fixed),
│                                    # brownfield/, ambiguous/, common/ (prose artifacts)
├── tests/
│   ├── AgenticSdlc.Tests/           # engine, policy, lineage, replanner, offline agents, scenario smoke
│   └── TargetApp.UrlShortener.Tests/# xUnit + WebApplicationFactory (in-process, no live server)
├── workspace/runs/<runId>/          # gitignored: audit.jsonl, state.json, metrics.json,
│                                    # artifacts/<name>/v<N>/, output/ (generated app),
│                                    # snapshots/, summary/FINAL_SUMMARY.md
└── docs/                            # ARCHITECTURE.md, SPECIFICATIONS.md, SCENARIOS.md,
                                     # ENGINEERING_SUMMARY.md, TESTING.md
```

**Key principle:** greenfield **generates a real, compiling, runnable ASP.NET Core project** into the run's `output/` directory and its xUnit suite is genuinely executed via `dotnet test`. `TargetApp.UrlShortener` is a real, tested baseline that the brownfield scenario modifies. Nothing is simulated where it can be real.

**NuGet dependencies** (kept deliberately small; ASP.NET Core itself comes from the shared framework, not NuGet):

| Package | Used by | Purpose |
|---|---|---|
| `Spectre.Console` | Cli | Approval prompts, selection menus, tables, live DAG rendering |
| `Microsoft.Extensions.Resilience` (Polly v8) | Engine | Retry pipelines: exponential backoff + jitter, timeout |
| `Microsoft.CodeAnalysis.CSharp` (Roslyn) | Agents, Policy | Semantic impact analysis, in-memory compile, banned-API detection |
| `Microsoft.Data.Sqlite` | TargetApp + generated app | Persistence (ADO.NET, no ORM) |
| `Microsoft.Extensions.AI` | Llm | `IChatClient` abstraction + middleware pipeline |
| `Microsoft.Extensions.Hosting` / `.DependencyInjection` / `.Configuration` | all hosts | DI, keyed agent registry, options binding |
| `xunit.v3`, `Microsoft.AspNetCore.Mvc.Testing`, `Microsoft.Extensions.TimeProvider.Testing` | tests | Test framework, in-process HTTP, fake clock |

## 4. Orchestrator engine (the differentiator)

- **Workflow DSL = C# records**, not YAML — gates are predicates (`Func<TaskContext, GateResult>`) and fault injections are declared data with behavior; a serialized format would need a mini-interpreter. `required`-member records give compile-time completeness checking, and scenario files double as readable documentation.
  ```csharp
  new WorkflowTask {
      Id = "architecture", Agent = AgentKey.Architect,
      DependsOn = ["decomposition"],
      Consumes  = ["requirements_spec", "task_plan"],
      Produces  = ["architecture_doc", "api_contract"],
      EntryGates = [Gate.ArtifactsCurrent("requirements_spec", "task_plan"), Gate.NotStopping],
      ExitGates  = [Gate.SchemaValid("api_contract"), Gate.Policy(PolicyPhase.PostDesign),
                    Gate.Approval("design_approval")],
      Retry = RetryProfile.Standard,   // 3 attempts, exponential + jitter
      Fallback = FallbackMode.Offline,
  }
  ```
  The workflow is validated at load: acyclicity via topological sort, every consumed artifact produced upstream, every agent key registered in DI. Validation failure aborts before any task runs (exit code 4).

- **Gate semantics.** Entry gates: all dependencies DONE, required artifacts exist *and are current* (content hash matches lineage expectation), policy pre-checks pass, safe-stop not engaged. Exit gates run in fixed order: schema validation → policy post-checks → test gate → approval gate last. Only after all exit gates pass is the changeset **applied** to `output/`, and only after snapshotting the previous state.

  > **Stage → validate → apply** is the load-bearing decision. Work is staged in an attempt directory, judged there, and lands only once. This makes rollback a directory restore rather than an inverse-diff problem, and keeps `output/` consistent at every instant.

- **State machine.** `TaskState` enum plus an explicit `FrozenDictionary<TaskState, FrozenSet<TaskState>>` legal-transition table. States: `Pending, Ready, Running, Validating, AwaitingApproval, Done, Retrying, Blocked, Failed, Invalidated, RolledBack, Skipped, Cancelled`. Illegal transitions throw.

- **Concurrency — the one genuine architecture change from a single-threaded design.** .NET `Task`s run on the thread pool, so there is no free single-threaded guarantee. State mutation is serialized through an **actor-style single-consumer `Channel<StateCommand>`**: the state actor is the *only* writer to task states, the artifact manifest, and the audit log. "One transition → one audit event" therefore holds by construction, with no locks anywhere. Agent work runs concurrently on the pool under a `SemaphoreSlim` (default 3); DAG joins are `Task.WhenAll`. Every await path carries a `CancellationToken`.

- **Retries and fallback.** Failures are classified `Transient` (HTTP 429/timeout, subprocess timeout) → Polly `ResiliencePipeline` with exponential backoff + jitter; `Validation` (schema mismatch, failing tests) → immediate retry with the failure output injected as a feedback input to the next attempt; `PolicyBlock` → `Blocked`, no automatic retry; `Fatal` → `Failed`. When LLM attempts are exhausted and an offline generator exists, the engine makes one fallback attempt (`FALLBACK_ACTIVATED`).

- **Rollback.** Restore `output/` from the pre-apply snapshot, retract the artifact version (manifest pointer returns to v(N−1)), transition to `RolledBack`, and invalidate downstream consumers via lineage. The `ROLLBACK` audit event records both content hashes.

- **Dynamic re-planning.** Any artifact gaining a new current version changes its hash. The replanner walks `DecisionRecord`s to find completed tasks whose *recorded input hash* no longer matches the current version, flips them `Done → Invalidated → Pending` transitively, and lets the scheduler re-execute exactly that subgraph. `REPLAN_TRIGGERED` records cause and affected set. Branch selection uses the same machinery plus `Skipped` for unchosen branches.

- **Safe-stop, two paths.** In-process: `CancellationTokenSource` propagated through every await. Cross-process: a `STOP` sentinel file written by `stop <runId>` from another terminal, polled each scheduler cycle (also surfaced on the dashboard). On stop: no new dispatch, in-flight work cancelled at await points, non-terminal tasks → `Cancelled`, `SAFE_STOP` emitted, state flushed. Because apply happens only after gates, a stopped run never leaves `output/` torn.

- **Audit.** `audit.jsonl`, append-only, one JSON object per line, flushed per event, written only by the state actor: `{seq, ts, runId, correlationId, eventType, payload}`. Serialized with source-generated `System.Text.Json` contexts.

- **Metrics.** Derived by a single fold over the audit log (no parallel bookkeeping to drift): success rate, retry/fallback/rollback counts, policy blocks, approval latency, MTTR (first failure → subsequent Done), per-stage latency, end-to-end wall time. Written to `metrics.json` via atomic replace after each transition; rendered by the dashboard and the final CLI summary. `System.Diagnostics.Metrics.Meter` counters are emitted alongside so the same signals could be scraped by OpenTelemetry in a real deployment.

## 5. Agents

`AgentBase.RunAsync(TaskContext, CancellationToken)`: load inputs (hashes recorded) → policy pre-check → generate (LLM or offline) → materialize + validate artifacts → policy post-check → record a `DecisionRecord`. Agents are registered as **keyed DI services** (`AddKeyedSingleton<IAgent>(AgentKey.Architect, …)`) and resolved by the scheduler from the task's `Agent` key.

Agents write **only** into their attempt directory — they never touch `output/` directly. Only the engine applies changesets.

Offline mode is deliberately *not* all-fake. Roslyn and the real test runner do genuine work regardless of mode:

| Agent | Produces | Real computation (both modes) |
|---|---|---|
| RequirementsAnalyst | RequirementsSpec, AmbiguityReport | — (templated prose; ambiguity interpretations pre-authored) |
| Planner | TaskPlan | — |
| Architect | ArchitectureDoc, ApiContract, **ImpactReport** | **Roslyn semantic analysis**: load the existing project, walk symbols, resolve references and call sites → impacted types/endpoints/data flows/regression risks |
| CodeGenerator | CodeChangeSet | Staged file changesets only |
| TestEngineer | TestSuite, TestReport | **Runs `dotnet test`** as an async subprocess against the generated project; TestReport parsed from the TRX report (exit code + stdout tail as fallback) |
| Reviewer | ReviewReport | **In-memory `CSharpCompilation.Emit()`** for real compiler diagnostics with source locations, plus a banned-API semantic walk |
| DocWriter | README / API docs | Populated from real values (endpoints from ApiContract, counts from TestReport) |
| ReleaseManager | ReleaseChecklist | Computed from actual audit state: gates green, tests passed on latest applied code, approvals present |

This is the strongest advantage of the .NET stack: codebase reasoning is done with a full semantic model (symbol resolution, reference finding), not text or syntax matching — and generated code is verified by the actual compiler.

**LLM layer.** `Microsoft.Extensions.AI.IChatClient` gives a provider-agnostic abstraction with a middleware pipeline (logging, telemetry). Behind it sits a thin `HttpClient` adapter for the Anthropic Messages API (`POST https://api.anthropic.com/v1/messages`, headers `x-api-key` and `anthropic-version`) — deliberately no third-party SDK dependency, since there is no official first-party Anthropic .NET SDK and a ~150-line adapter carries less supply-chain risk than a community package. Default model `claude-fable-5`, configurable. Structured outputs use `JsonSchemaExporter.GetJsonSchemaAsNode(...)` to generate the JSON schema *from the same C# record* used for deserialization and validation — one source of truth for prompting and checking. On a validation failure the client attempts exactly one repair round-trip before the attempt is classified `Validation`.

## 6. Policy guardrails

Each rule returns `PolicyResult(RuleId, Verdict {Pass|Warn|Block}, Details, ComplianceTags)` and every evaluation is recorded as a `POLICY_CHECK` audit event.

1. **SecretScan** — regex battery over generated file contents (cloud access keys, PEM private-key headers, hardcoded `apikey/password/secret/token` assignments) → Block.
2. **ForbiddenApis** — Roslyn *semantic* walk against a banned-symbol list (`System.Diagnostics.Process.Start`, `System.Reflection.Emit.*`, `System.Runtime.InteropServices` P/Invoke, dynamic code loading) plus a NuGet package allowlist (generated app may reference only `Microsoft.Data.Sqlite` beyond the shared framework) → Block. Same enforcement model as `Microsoft.CodeAnalysis.BannedApiAnalyzers`, run in-process so verdicts flow into the audit log rather than only failing a build.
3. **ProtectedPaths** — changesets may write only inside the run's `output/` (verified after `Path.GetFullPath` normalization, so `..` traversal is caught); designated protected files require an explicit human override, itself audited with `overrideBy`.
4. **ChangeBudget** — Block above 15 files or 1200 changed LOC per changeset, with guidance to split the change.
5. **TestGate** — exit gate for code stages: `dotnet test` exit code 0 **and** collected test count ≥ the scenario minimum.
6. **ReleasePolicy** — release gate requires zero unresolved Blocks, TestGate green on the latest applied code version, and every mandatory approval present in the audit log.

## 7. Target application — URL shortener

ASP.NET Core minimal APIs, `Microsoft.Data.Sqlite` (ADO.NET, no ORM — three tables do not justify EF Core, and it keeps the generated project to a single NuGet reference).

| Endpoint | Behavior |
|---|---|
| `POST /api/links` | `{url, customAlias?, ttlSeconds?}` → 201 `{code, shortUrl, expiresAt}`; 409 alias taken; 422 invalid; 429 rate-limited |
| `GET /{code}` | 307 redirect (`TypedResults.Redirect(url, permanent: false, preserveMethod: true)`) + click recorded; 404 unknown/deleted; 410 expired |
| `GET /api/links/{code}` | Metadata |
| `DELETE /api/links/{code}` | 204 soft delete |
| `GET /api/links/{code}/stats` | `{totalClicks, lastClickedAt, clicksByDay}` |
| `GET /api/links` | Paged list |
| `GET /healthz` | Health check with DB probe |

Platform capabilities used instead of hand-rolled code:

| Concern | Implementation |
|---|---|
| Rate limiting | `AddRateLimiter` + `FixedWindowLimiter` (20/min per client IP on create), `RequireRateLimiting("create")`, 429 + `Retry-After` via `OnRejected` |
| Health checks | `AddHealthChecks().AddCheck<SqliteHealthCheck>()` + `MapHealthChecks("/healthz")` with a JSON response writer |
| Clock (TTL) | `TimeProvider` injected via DI; tests use `FakeTimeProvider` — no time-freezing library needed |
| API documentation | `Microsoft.AspNetCore.OpenApi` (`AddOpenApi` / `MapOpenApi`) |
| Configuration | `Shortener__DbPath` environment variable binds to `Shortener:DbPath` via `IOptions<ShortenerOptions>` |

Schema: `links(id, code UNIQUE, url, is_custom, created_at, expires_at, deleted)` and `clicks(id, link_id, clicked_at, referrer)` with an index on `link_id`. `expires_at` and the whole `clicks` table are what the brownfield scenario adds.

Codes are **base62 of `rowid + 100000`** — collision-free by construction, always ≥ 4 characters, one code path with no retry loop. Custom aliases match `^[A-Za-z0-9_-]{3,32}$` and exclude a reserved set (`api`, `healthz`, `openapi`, `admin`). TTL is enforced lazily at redirect time against the injected `TimeProvider`, with a `PurgeExpired()` maintenance method.

Tests: xUnit v3 + `WebApplicationFactory<Program>` (in-process, no live server; the app declares `public partial class Program;` to make the entry point addressable). Unit coverage for base62 round-tripping, alias validation, TTL boundaries against `FakeTimeProvider`, and rate-limiter window math; integration coverage for create → redirect → stats → delete, plus 409 / 410 / 429 / 404 / health.

## 8. Three scenarios

Fault injections are declared data in the scenario definition and labeled `injected: true` in every related audit event — deterministic demo choreography, honestly surfaced rather than hidden.

**Constraint specific to a compiled language:** every `v_bug` template must **compile cleanly and fail at test time**. A syntax error would fail at build, which is a different (and less interesting) failure class than the retry loop is meant to demonstrate.

Shared DAG core:
`requirements → decomposition → architecture[APPROVAL] → {codegen_core ∥ codegen_api} → test_authoring → test_run[TESTGATE] → review[APPROVAL] → docs → release[APPROVAL]`

1. **Greenfield — "Build the URL shortener from requirements."** Full pipeline with parallel codegen branches joined at `test_authoring`. *Fault 1:* `codegen_core` attempt 1 uses `v_bug` — an off-by-one in base62 decode that compiles fine and fails real tests → `Validation` retry with the test output as feedback → attempt 2 `v_fixed` → green. *Fault 2:* `codegen_api` attempt 1 contains a hardcoded `ApiKey = "sk-demo-…"` → SecretScan Block → regeneration.

2. **Brownfield — "Add TTL expiry + click analytics to the existing shortener."** `TargetApp.UrlShortener` is copied into `output/` at run start. An extra `impact_analysis` stage produces a Roslyn-derived ImpactReport. *Fault 1:* a scripted post-design requirement revision ("analytics must record referrer") creates RequirementsSpec v2 → `REPLAN_TRIGGERED`, architecture and downstream invalidated and re-run. *Fault 2:* changeset attempt 1 exceeds ChangeBudget → Block → targeted patch. *Fault 3:* the applied patch fails regression tests (a seeded **logic** bug: the INSERT omits the `expires_at` default, so TTL reads back null) → **rollback** from snapshot → corrected patch → green, including the original baseline tests.

3. **Ambiguous — "Make the service handle high traffic."** RequirementsAnalyst emits an AmbiguityReport: clarifying questions plus three ranked interpretations — **A** in-process caching of hot redirects (recommended, in scope), **B** horizontal scaling with an external store (out of prototype scope, recommend defer), **C** measure-first load testing. The requirements approval gate is a Spectre `SelectionPrompt` menu; the choice writes RequirementsSpec v2 with populated `assumptions[]`, activates branch A, and marks B/C `Skipped`. Path A implements an LRU cache over code→url lookups with invalidation on delete, plus a load smoke test (200 concurrent redirects asserting p95 latency and a non-zero cache-hit count).

Every run ends by generating `summary/FINAL_SUMMARY.md` from real run data — decisions, approvals, policy results, metrics, assumptions, limitations.

## 9. Dashboard

`dotnet run --project src/AgenticSdlc.Dashboard` on port 8600. A minimal API serving one static page, **read-only over the run directory** — the engine writes `state.json` and `metrics.json` via atomic replace and appends to `audit.jsonl`; the dashboard polls and tails by `seq`. Zero shared state means it cannot destabilize the engine, it survives an engine crash, and it replays finished runs for post-hoc inspection.

Four panels: DAG laid out in topological columns with per-task state chips, gates/approvals (pending approval highlighted), audit tail (last 50, incremental fetch), metrics cards. Vanilla JS, no npm and no build step — an evaluator needs only the .NET SDK they already installed.

## 10. Documentation deliverables

`README.md` (prerequisites, build, the three demo commands, flags, expected runtime), `docs/ARCHITECTURE.md`, `docs/SPECIFICATIONS.md`, `docs/SCENARIOS.md` (per-scenario walkthrough: input, where each fault fires, what to watch, expected audit events), `docs/ENGINEERING_SUMMARY.md` (rationale, risks, trade-offs, assumptions, limitations, production-hardening path), `docs/TESTING.md`.

## 11. Build order

Always demoable. Cut lines if time runs short, in order: SVG DAG edges → LLM mode (offline is the demo default anyway) → dashboard polish. Never cut: engine tests, scenario smoke tests, README.

| Phase | Scope | Exit criterion |
|---|---|---|
| 0 | `git init`; solution scaffold, `global.json`, `Directory.Build.props`; Core: `TaskState` + transition table + audit contracts | `dotnet test` green on StateMachineTests |
| 1 | Engine: state actor, scheduler, workflow DSL + validation, Polly retry, non-approval gates | Toy DAG with an injected failure retries and completes; parallel branches join correctly |
| 2 | Spectre approvals + `--auto-approve`, 6 policy rules, safe-stop, replanner | Replanner test: mutating an artifact invalidates exactly the right downstream set |
| 3 | `TargetApp.UrlShortener` baseline + its tests; greenfield templates (`v_bug` / `v_fixed`) | Baseline `dotnet test` green standalone; both templates compile |
| 4 | Offline agents + greenfield scenario end-to-end | **First full demo:** generated project compiles and its tests pass via `dotnet test` |
| 5 | Brownfield (Roslyn impact analysis, replan, rollback) + ambiguous (branch selection) | All three smoke tests green with `--auto-approve --offline` |
| 6 | Dashboard | Live run visible in the browser |
| 7 | LLM adapter + fallback path (tested with a fake failing `IChatClient`) | Runs with a key; falls back cleanly without one |
| 8 | Docs, final-summary generator, metrics table, dry run of all three scenarios | README quickstart works from a clean clone |

## 12. Verification

- `dotnet test` at the solution root — engine unit tests (state machine, toy DAGs with parallel/retry/block, lineage, replanner, all six policy rules, offline agents) plus `ScenarioSmokeTests` running all three scenarios end-to-end with `--auto-approve --offline`.
- `dotnet test tests/TargetApp.UrlShortener.Tests` — baseline green before brownfield touches it.
- Manual demo pass with the dashboard open: greenfield (interactive approvals — observe retry, policy block, approval, live metrics), brownfield (observe `REPLAN_TRIGGERED` and `ROLLBACK`), ambiguous (interpretation menu branches the DAG). Then Ctrl+C or `stop <runId>` → `Cancelled` states and a `SAFE_STOP` event.
- The generated greenfield app runs: `dotnet run` from `workspace/runs/<id>/output/`, then `POST /api/links` → `GET /{code}` redirect → stats.
- Clean-clone check of the README quickstart.

## 13. Key risks and mitigations

| Risk | Mitigation |
|---|---|
| **Build-cycle latency** — `dotnet build` + `dotnet test` per attempt is 5–15 s versus ~1 s for an interpreted stack, and retries multiply it | Keep the generated project minimal (one NuGet reference); warm `NUGET_PACKAGES` cache; `--no-restore` after the first restore; 180 s subprocess timeout classified `Transient`. Budget ~6–10 min per scenario, not 2–4. |
| **Stale SDKs on this machine** (1.1.x / 2.1.x alongside 10.0.302) | `global.json` pinning `10.0.302` with `rollForward: latestFeature`; README states the requirement explicitly |
| **NuGet restore needs network on first build** — undercuts the "zero-network demo" claim | Restore during setup, before the demo; document that the *demo* is offline but the *build* needs one prior restore; optionally vendor a local package folder |
| **Injected bugs must compile** — a syntax error fails at build, not at the test gate | Every `v_bug` template is a logic bug, and a build-check test asserts all templates compile |
| **No official Anthropic .NET SDK** | Thin `HttpClient` adapter behind `IChatClient` (~150 LOC), no third-party package; LLM mode is optional anyway |
| **Thread-pool concurrency loses the free single-thread guarantee** | Actor discipline: only the state actor writes state; enforced by keeping mutation methods internal to the actor and covered by a concurrency test that hammers the channel |
| **Roslyn adds meaningful package weight and first-load latency** | Accepted deliberately — it is the single biggest capability win; load the workspace once per run and reuse the compilation |
| **Scope creep** | Pre-agreed cut lines; no git-based changesets (directory snapshots instead), no run resume after stop, no SignalR, no auth, no coverage-percentage gate — all documented as trade-offs |
