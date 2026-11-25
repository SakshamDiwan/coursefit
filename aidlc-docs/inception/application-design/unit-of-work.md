# CourseFit Units of Work

**Deployment model**: modular monolith in a **uv workspace**. The 6 **packages** are reusable libraries and the 3 **apps** are runnable or deployable. Everything shares one `uv.lock`. The MCP app and its job handlers deploy as one Lambda bundle built from the workspace.

**Owners**: TBD, to be assigned at the team meeting. **Construction** runs in parallel: each owner runs the AI-DLC Construction stages for their unit on a branch and opens a pull request reviewed by at least one teammate. **U0 merges first.**

---

## Code Organization Strategy (greenfield)

```text
coursefit/                                # repo root (workspace root)
├── pyproject.toml                        # [tool.uv.workspace] members = ["packages/*", "apps/*"]; shared ruff/pytest config
├── uv.lock
├── packages/
│   ├── coursefit-contracts/              # U0
│   │   ├── pyproject.toml
│   │   ├── src/coursefit_contracts/
│   │   └── tests/
│   ├── coursefit-platform/               # U1   src/coursefit_platform/{access,plans,observability,db}/
│   ├── coursefit-cornell/                # U2   src/coursefit_cornell/{catalog,roster}/
│   ├── coursefit-schedule/               # U3   src/coursefit_schedule/
│   ├── coursefit-recommend/              # U4   src/coursefit_recommend/{retrieval,llm,interview,recommender}/
│   └── coursefit-planning/               # U5   src/coursefit_planning/{plan,conversation,recommendation,selection}.py
├── apps/
│   ├── mcp-server/                       # U6   src/coursefit_server/{tools,handler,jobs}/ ; tests/
│   ├── admin-cli/                        # U7   src/coursefit_admin/ ; tests/
│   └── benchmark/                        # U8   src/coursefit_bench/ ; tests/
├── infra/                                # U6   IaC (tool chosen in Infrastructure Design)
├── rules/                                # U2   2026-27.yaml (verified catalog rules, reviewed via PR)
├── benchmark/scenarios/                  # U8   S1–S10 scenario + student profile files
├── tests/integration/                    # cross-unit tests (owned jointly; run in Build and Test)
└── aidlc-docs/                           # documentation only
```

**Conventions**
- **Import names**: `coursefit_<name>`. **Distribution names**: `coursefit-<name>`.
- **Dependencies between workspace members** are declared with `[tool.uv.sources] <pkg> = { workspace = true }`. A unit may import only what it declares (see unit-of-work-dependency.md).
- **Tests** live in each package's own `tests/`, so an owner runs `uv run --package <pkg> pytest`. Cross-unit tests live in `tests/integration/`.
- **Shared assets** stay at the root: `infra/`, `rules/`, `benchmark/scenarios/`.
- **Mapping to the application design**: component folders inside each package keep the C-numbers from the design, so AD-4's "folder per component" still holds across workspace members.

---

## U0 — Contracts (`packages/coursefit-contracts`)
- **Purpose**: The shared data shapes every unit codes against. Written **jointly** in the first 1–2 days.
- **Contents**:
  - IDs: `ClientId`, `PlanId`, `CourseId`, `CombinationId`
  - Course models: `Course`, `SectionGroup`, `Section`, `Meeting`, `SectionCombination`
  - Busy time: `WeeklyBlock`, `DatedBlock`, `BusySet`
  - Planning state: `Constraint`, `Preferences`, `Assumption`, `Selection`, `Goal`, `InterviewState`, `Plan`
  - Tool I/O: `TurnResult` (question / results / conflict / ack), `Recommendation`, `PlanSummary`, `PlanDetail`, `ToolError` with error codes, and the MCP tool input models
- **Rules**: Pydantic v2. `extra="forbid"` on all inputs. Length and size limits live on the models (SECURITY-05). No I/O.
- **Done when**:
  - The models are merged and reviewed by all five teammates.
  - JSON round-trip property tests pass (PBT-02).
  - Example fixtures exist (a small fake SP27 course set and sample plans) that the other units use.

## U1 — Platform (`packages/coursefit-platform`)
- **Components**: C2 Access and Identity, C3 Plan Store, C11 Observability, plus shared DB plumbing (connection pool, migrations).
- **Responsibilities**:
  - API-key hashing, lookup, issue and revoke.
  - Per-key rate limits.
  - `require_plan_owner`.
  - The `PlanRepository` with client-scoped SQL and optimistic versioning.
  - `delete_idle`.
  - Redacted JSON logging, `error_boundary`, and metrics.
  - Database schema migrations.
- **Done when**:
  - Ownership isolation is tested.
  - Keys are stored only as hashes.
  - A redaction test proves no keys, busy blocks, or free text are logged.
  - Migrations apply to a disposable Postgres.

## U2 — Cornell (`packages/coursefit-cornell`)
- **Components**: C4 Catalog Rules (including the offline scraper) and C5 Roster Data (client, field mapping, repository, refresher).
- **Responsibilities**:
  - Load verified rules from `rules/2026-27.yaml`.
  - Requirement matching, plus assumptions and overrides.
  - A scraper that produces the same output for the same page.
  - The SP27 field mapping.
  - A roster client limited to 1 request per second.
  - Atomic snapshot swaps that keep the previous snapshot on failure.
  - Live lookups on cache misses.
  - Cross-listing resolution.
- **Done when**:
  - The demo categories (2–3 per program) are verified in a pull request.
  - The SP27 field mapping is recorded.
  - respx fixtures cover the roster API.
  - A refresh-failure test passes.

## U3 — Schedule (`packages/coursefit-schedule`)
- **Component**: C6 Scheduling Engine (**pure**, depends only on U0).
- **Responsibilities**:
  - Busy-block and meeting expansion over the term (America/New_York).
  - Overlap checks with exclusive end times.
  - Section-combination enumeration and feasibility.
  - Credit fit, including variable credits and the per-course limit.
  - Idempotent `apply_selection` and `remove_selection`.
  - `explain_infeasibility`.
- **Done when**:
  - Hypothesis property tests (PBT-03, 07, 08) and example tests pass: back-to-back meetings, a one-minute overlap, a lab conflict, an empty schedule, a repeated selection, and partial-term sections.

## U4 — Recommend (`packages/coursefit-recommend`)
- **Components**: C7 Retrieval (BM25), C8 Model Gateway, C9 Interview Engine, C10 Recommender.
- **Responsibilities**:
  - OpenRouter structured calls with retry, then fallback, and a flag to disable fallback.
  - Usage recording.
  - A prompt allow-list that admits no busy blocks and no identity.
  - Interpret / apply / decide_next, with the soft ceiling of about 4 enforced in code.
  - Term expansion.
  - Rerank and explain.
  - `validate_ranked`.
  - `choose_combination`.
- **Done when**:
  - All model steps are tested with a fake `ModelGateway`.
  - The fallback path and the "OpenRouter down" path are tested.
  - A test proves prompts never contain busy blocks.

## U5 — Planning (`packages/coursefit-planning`)
- **Components**: S1 PlanService, S2 ConversationService, S3 RecommendationService, S4 SelectionService.
- **Responsibilities**:
  - The per-turn and recommendation pipelines.
  - One transaction per request with version checks.
  - Fail closed: nothing is saved on a model failure.
  - Selection and refresh.
  - Conflict results.
- **Done when**:
  - In-process service tests pass for demo scenarios S1–S10 using a fake model, a fixture roster, and a disposable Postgres.

## U6 — MCP Server (`apps/mcp-server` + `infra/`)
- **Components**: C1 MCP Interface (12 tools with descriptions that include the host guidance on busy blocks), the Lambda handler (FastMCP stateless Streamable HTTP behind a Function URL), the C12 job handlers (roster refresh and plan cleanup as separate Lambda entry points in the same bundle), and the IaC.
- **Responsibilities**:
  - The tool request chain: auth, then rate limit, then validation, then ownership, then the service, then the error boundary.
  - The workspace-to-Lambda bundle build.
  - Infrastructure: Lambda, Function URL, RDS, Secrets Manager, VPC, EventBridge, CloudWatch alarms, retention.
  - **An early spike on R2**: does Claude Desktop accept the API-key header?
- **Done when**:
  - An ownership test exists for every tool.
  - The MCP in-process client tests pass.
  - The app is deployed to AWS.
  - It's verified in Claude Desktop and in Claude Code or Codex.

## U7 — Admin CLI (`apps/admin-cli`)
- **Component**: C13.
- **Commands**: `keys issue` and `keys revoke`; `catalog scrape` and `catalog check`; `roster inspect-fields` and `roster snapshot`.
- **Done when**: keys print only once, `catalog check` fails on any unverified demo rule or one missing a `source_url`, and each command has tests.

## U8 — Benchmark (`apps/benchmark` + `benchmark/scenarios/`)
- **Components**: C14 Benchmark Harness and S6 (calls U5 services directly, fallback disabled).
- **Responsibilities**:
  - Scenario and profile files for S1–S10.
  - `ProfileResponder`, whose design resolves risk R5.
  - Per-model run records: model, latency, cost, questions, what each answer revealed, the shortlist, and whether it met the 60-second target.
  - Hard-constraint checks.
  - A results report.
- **Done when**: a run against at least two models produces a comparable report with zero constraint violations.

---

## Construction Stage Applicability per Unit

| Unit | Functional Design | NFR Requirements | NFR Design | Infrastructure Design | Code Gen |
|---|---|---|---|---|---|
| U0 Contracts | EXECUTE (data models) | EXECUTE (light: validation limits, PBT-09 framework) | SKIP (no runtime behavior) | SKIP | EXECUTE |
| U1 Platform | EXECUTE (schema, auth, ownership) | EXECUTE | EXECUTE | SKIP (provides DB requirements to U6) | EXECUTE |
| U2 Cornell | EXECUTE (rule model, matching, mapping) | EXECUTE (rate limit, freshness) | EXECUTE | SKIP (job infrastructure in U6) | EXECUTE |
| U3 Schedule | EXECUTE (core rules, testable properties) | EXECUTE (light: PBT-09) | SKIP (pure library) | SKIP | EXECUTE |
| U4 Recommend | EXECUTE (interview policy, ranking, prompts) | EXECUTE (latency, prompt safety) | EXECUTE | SKIP | EXECUTE |
| U5 Planning | EXECUTE (orchestration, transactions) | EXECUTE (light) | EXECUTE (fail closed, concurrency) | SKIP | EXECUTE |
| U6 MCP Server | SKIP (thin, contract fixed in U0) | EXECUTE | EXECUTE | **EXECUTE** | EXECUTE |
| U7 Admin CLI | SKIP (thin wrapper) | EXECUTE (light: key handling) | SKIP | SKIP | EXECUTE |
| U8 Benchmark | EXECUTE (scenarios, ProfileResponder R5) | EXECUTE (light: cost, reproducibility) | SKIP | SKIP | EXECUTE |

## Suggested Grouping for Five Owners (team decides)

| Owner slot | Units | Why grouped |
|---|---|---|
| A | U1 Platform + U7 Admin CLI | Keys and the database live together |
| B | U2 Cornell | Scraping, verification, and roster work is a full load |
| C | U3 Schedule + U8 Benchmark | The benchmark checks hard constraints using the same scheduling knowledge |
| D | U4 Recommend | The LLM-heavy core |
| E | U5 Planning + U6 MCP Server/infra | End-to-end wiring and deployment |

Everyone co-writes U0.
