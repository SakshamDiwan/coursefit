# CourseFit Components

All components live in one Python package, `coursefit/`, with one folder per component (design decision AD-4). A single Lambda behind a **Lambda Function URL** serves MCP over Streamable HTTP (AD-1). Two small Lambdas reuse the same package for scheduled jobs.

## Design Decisions

| ID | Decision | Source |
|---|---|---|
| AD-1 | Requests go through a **Lambda Function URL**, not API Gateway, which avoids the roughly 29-second gateway limit. CourseFit enforces API-key auth and per-key rate limits in code. | Design Q1 |
| AD-2 | **Hybrid tool surface**: typed tools for structured actions, plus one `say` tool for the student's own words. CourseFit interprets the free text. | Design Q2 |
| AD-3 | **Code-driven LLM pipeline**: code runs small, structured model steps and validates each output. No agent loop. | Design Q3 |
| AD-4 | **One package, few Lambdas**. The scheduling engine is pure, with no database or network access. | Design Q4 |


> **Update (Units Generation):** AD-4 is implemented as a **uv workspace** with 6 packages and 3 apps (see [unit-of-work.md](unit-of-work.md)). It still deploys as one Lambda bundle. Component folders now live inside their unit's package.

---

## C1 — MCP Interface (`coursefit/mcp_server/`)
- **Purpose**: The only entry point for hosts.
- **Responsibilities**:
  - Define the MCP tools, their Pydantic input and output schemas, and descriptions, including the host guidance on busy blocks.
  - Run the tool chain for every request: access guard, then input validation, then the service call, then response shaping.
  - Run FastMCP in stateless Streamable HTTP mode inside the Lambda handler.
- **Interface**: the 12 MCP tools in [component-methods.md](component-methods.md#c1--mcp-interface).

## C2 — Access and Identity (`coursefit/access/`)
- **Purpose**: All security-critical identity logic in one place (SECURITY-08, SECURITY-11).
- **Responsibilities**:
  - Resolve the bearer API key to a client ID by hash lookup.
  - Reject missing, unknown, or revoked keys.
  - Enforce per-key rate limits.
  - Check plan ownership before any plan access.
  - Emit auth and authorization failure events for alerting.
  - Issue and revoke keys (used by the admin CLI).
- **Interface**: `authenticate`, `check_rate_limit`, `require_plan_owner`, `issue_key`, `revoke_key`.

## C3 — Plan Store (`coursefit/plans/`)
- **Purpose**: Persist planning sessions ("plans") in PostgreSQL.
- **Responsibilities**:
  - Plan create, read, update, and delete, using parameterized SQL and optimistic version checks.
  - Every query is scoped to the owning client.
  - Update the last-activity timestamp on each change.
  - Delete a plan with all its data.
  - Find plans idle for more than 30 days.
- **Interface**: a `PlanRepository` that reads and writes the `Plan` aggregate.

## C4 — Catalog Rules (`coursefit/catalog/`)
- **Purpose**: Verified requirement rules and requirement matching.
- **Responsibilities**:
  - Load the version-controlled rule file and expose only `verified` rules.
  - List supported requirements.
  - Decide whether a course matches a rule.
  - Produce assumptions for conditions CourseFit can't check, and re-evaluate when the student overrides one.
- **Interface**: `RuleCatalog`.
- **Offline part** (`coursefit/catalog/scraper/`): scrape the catalog pages into candidate rules with status `unverified`. This produces the same output for the same page.

## C5 — Roster Data (`coursefit/roster/`)
- **Purpose**: Spring 2027 offerings from the Cornell Class Roster API.
- **Responsibilities**:
  - An httpx client that sends at most 1 request per second.
  - Map SP27 fields to CourseFit's course, section-group, and meeting models.
  - Take snapshots and do the daily refresh into Postgres, keeping the previous snapshot if a refresh fails.
  - Look up a course, falling back to a live request when it isn't cached.
  - Resolve cross-listings.
- **Interface**: `RosterRepository`, `RosterClient`, `RosterRefresher`.

## C6 — Scheduling Engine (`coursefit/schedule/`) — pure
- **Purpose**: Deterministic fit and credit logic. No I/O.
- **Responsibilities**:
  - Expand weekly and dated busy blocks into dated intervals within the term (America/New_York).
  - Check interval overlap with exclusive end times.
  - Enumerate allowed lecture, discussion, and lab combinations.
  - Check feasibility against busy blocks, constraints, and selections.
  - Do credit arithmetic, including variable credits and the per-course limit.
  - Apply selections idempotently and remove them.
  - Explain which constraints block every option.
- **Interface**: the free functions listed in component-methods.md. These are the primary targets for property-based tests.

## C7 — Retrieval (`coursefit/retrieval/`)
- **Purpose**: Find candidate courses for the student's interests.
- **Responsibilities**:
  - Build a BM25 index over SP27 titles and descriptions, from the roster cache.
  - Run a search with the expanded terms and an optional candidate filter (the requirement's course list).
  - Sit behind a `Retriever` protocol so semantic search can be swapped in later.
- **Interface**: `Retriever.search`.

## C8 — Model Gateway (`coursefit/llm/`)
- **Purpose**: The only path to OpenRouter.
- **Responsibilities**:
  - Make structured (JSON-schema) completion calls.
  - Retry once, then switch to `COURSEFIT_FALLBACK_MODEL`. Fallback can be disabled for benchmarks.
  - Enforce per-call timeouts.
  - Validate each response against a Pydantic schema.
  - Record the model that answered, latency, and token cost.
  - Enforce an allow-list of what may go in a prompt: never busy blocks or client identity.
- **Interface**: `ModelGateway.complete_structured`.

## C9 — Interview Engine (`coursefit/interview/`)
- **Purpose**: Understand what the student said and decide what to ask next.
- **Responsibilities**:
  - **Interpret** a message into structured updates:
    - preferences (inferred) and must-haves (explicit)
    - binding constraints (new, edited, or removed)
    - assumption overrides
    - the answer to the pending question
    - a request for results
  - **Decide** the next step: ask a follow-up (only if it could change results) or stop. Apply the soft ceiling of about 4 follow-ups.
  - **Phrase** the follow-up question.
  - Record what each answer revealed, for evaluation.
- **Interface**: `InterviewEngine.interpret`, `InterviewEngine.decide_next`.

## C10 — Recommender (`coursefit/recommend/`)
- **Purpose**: Turn interests and the feasible set into five explained results.
- **Responsibilities**:
  - Expand search terms with the model.
  - Choose candidates: the requirement filter, then retrieval, then feasibility via C6.
  - Rerank and explain with the model.
  - Validate model output against the verified feasible IDs.
  - Pick the recommended section combination with a deterministic preference score.
  - Attach assumptions, prerequisite text, and trade-offs.
  - Detect when nothing is feasible and report the blocking constraints.
- **Interface**: `Recommender.recommend`, `Recommender.section_options`.

## C11 — Observability (`coursefit/observability/`)
- **Purpose**: Logging, errors, and metrics that are safe for student data (SECURITY-03, 14, 15).
- **Responsibilities**:
  - Structured JSON logging with request IDs and a redaction filter, so no keys, bodies, busy blocks, or free text are logged.
  - A global error handler that returns generic messages and fails closed.
  - Metrics for auth and authorization failures and for model latency and cost.
- **Interface**: `get_logger`, `error_boundary`, `emit_metric`.

## C12 — Scheduled Jobs (`coursefit/jobs/`)
- **Purpose**: Lambda handlers triggered by EventBridge.
- **Responsibilities**:
  - Daily roster refresh (through C5).
  - Daily deletion of plans idle for 30 days (through C3).
  - Log counts only.
- **Interface**: `roster_refresh_handler`, `plan_cleanup_handler`.

## C13 — Admin CLI (`coursefit/admin/`)
- **Purpose**: Operator commands (persona P4).
- **Responsibilities**:
  - `keys issue` and `keys revoke`
  - `catalog scrape` and `catalog check`, which validates that the rule file has only verified demo categories with source URLs
  - `roster inspect-fields` and `roster snapshot`
- **Interface**: a command-line entry point, `coursefit-admin`.

## C14 — Benchmark Harness (`coursefit/benchmark/`)
- **Purpose**: Compare models on fixed scenarios (FR-9).
- **Responsibilities**:
  - Load scenario definitions (student profile, plan setup, busy blocks).
  - Drive the services in process with the fallback disabled.
  - Answer follow-ups through a pluggable `ProfileResponder`. Whether it uses scripted answers or a simulated student is decided in Functional Design (risk R5).
  - Record metrics and write a results file per model.
  - Check every result against the hard constraints.
- **Interface**: `coursefit-bench run --models ... --scenarios ...`.
