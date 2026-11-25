# CourseFit Application Design (Consolidated)

This document summarizes the design. The details are in [components.md](components.md), [component-methods.md](component-methods.md), [services.md](services.md), and [component-dependency.md](component-dependency.md).

## 1. Architecture at a Glance

CourseFit is **one Python package**. It is deployed as:
- one **MCP Lambda behind a Lambda Function URL** (stateless Streamable HTTP, requests up to 15 minutes)
- two **scheduled job Lambdas**: the daily roster refresh and the 30-day plan cleanup

**PostgreSQL on RDS** stores API-key hashes, plans, and the Spring 2027 roster cache. Verified requirement rules ship as a reviewed YAML file. OpenRouter and the Cornell Class Roster API are the only outside services.

| Decision | Choice | Why |
|---|---|---|
| AD-1 Request path | Lambda Function URL, with auth and rate limiting in code | Avoids the roughly 29-second API Gateway limit against the 60-second shortlist target |
| AD-2 Tool surface | Hybrid: 11 typed tools plus 1 `say` tool for the student's own words | Exact actions stay validated and secure, while CourseFit interprets free text the same way in every host |
| AD-3 LLM use | Code-driven pipeline of 2–4 small structured model steps per turn | Predictable, testable with a fake model, and fair across benchmarked models |
| AD-4 Packaging | One package with a folder per component. The scheduling engine is pure. | Fastest to build in three weeks. Teammates own folders, and the core can be property-tested |


> **Update (Units Generation):** AD-4 is implemented as a **uv workspace** with 6 packages and 3 apps (see [unit-of-work.md](unit-of-work.md)). It still deploys as one Lambda bundle. Component folders now live inside their unit's package.

## 2. Components

| ID | Component | Role |
|---|---|---|
| C1 | MCP Interface | 12 tools, schemas, host guidance, request chain |
| C2 | Access and Identity | API-key auth, rate limits, ownership guard, key issue and revoke |
| C3 | Plan Store | Plan persistence scoped to each client, idle deletion |
| C4 | Catalog Rules | Verified rules, requirement match, assumptions, offline scraper |
| C5 | Roster Data | Cornell client (≤1 request/s), field mapping, snapshot and refresh, lookups |
| C6 | Scheduling Engine | **Pure**: intervals, section combinations, feasibility, credits, selections, infeasibility reasons |
| C7 | Retrieval | BM25 behind a `Retriever` protocol |
| C8 | Model Gateway | OpenRouter structured calls, retry then fallback, usage recording, prompt allow-list |
| C9 | Interview Engine | Interpret messages, decide the next question (soft ceiling of about 4), phrase the question |
| C10 | Recommender | Expand terms, check feasibility, rerank and explain, validate IDs, choose combinations |
| C11 | Observability | Redacted JSON logs, error boundary, metrics |
| C12 | Scheduled Jobs | Roster refresh, plan cleanup |
| C13 | Admin CLI | Keys, catalog scrape and check, roster inspect and snapshot |
| C14 | Benchmark Harness | Scenario runner, `ProfileResponder`, metrics, constraint checks |

Services: S1 Plan, S2 Conversation, S3 Recommendation, S4 Selection, S5 DataOps, S6 Benchmark.

## 3. MCP Tool Contract

| Tool | Purpose |
|---|---|
| `list_requirements` | List verified requirement categories |
| `start_plan` / `update_goal` | Set the term (SP27), credits, per-course limit, and optional requirement |
| `set_busy_blocks` | Replace busy time with weekly or dated blocks (times only) |
| `say` | The student's own words: answers, constraints, corrections, overrides, "show me results" |
| `get_recommendations` | Get results now (shortlist or conflict) |
| `list_section_options` | Show every feasible section combination for a course |
| `select_course` / `remove_course` | Build the plan. Re-selecting is idempotent and the shortlist refreshes |
| `list_plans` / `get_plan` / `end_plan` | Saved plans: list, resume, delete |

`say` and `get_recommendations` return a `TurnResult` whose `kind` is `question`, `results` (5 or fewer), `conflict`, or `ack`. Every error uses the same generic `ToolError`. A plan the caller doesn't own returns `not_found`.

## 4. Key Flows

- **Interview turn** (`say`):
  1. auth
  2. load the plan
  3. interpret [LLM]
  4. apply [pure]
  5. decide the next step [LLM, plus the ceiling in code]
  6. ask a question, or recommend
  7. save with a version check
- **Recommendation**:
  1. compute blocked time [pure]
  2. build the candidate pool (requirement or all of SP27)
  3. expand terms [LLM]
  4. BM25 search
  5. check credits, combinations, and the rule [pure]
  6. if nothing is feasible, return a conflict with the blocking constraints
  7. otherwise rerank and explain [LLM], validate IDs, keep the top 5, and choose a combination for each
- **Selection**: recheck fit, apply idempotently, save, refresh the shortlist.
- **Jobs**: daily roster snapshot swap (the old one is kept on failure), and daily deletion of plans idle 30 days.

## 5. Security Placement (Security Baseline)

| Concern | Where | Rule |
|---|---|---|
| Authentication, rate limiting, ownership (deny by default) | C2, invoked by C1 on every tool | SECURITY-08, 11 |
| Input validation (types, lengths, sizes, no extra fields) | C1 Pydantic schemas | SECURITY-05 |
| Parameterized SQL, client-scoped queries | C3, C5, C2 | SECURITY-05, 08 |
| No busy blocks or identity in prompts; model output checked against verified IDs | C8 allow-list, C10 `validate_ranked` | SECURITY-11 |
| Redacted structured logs, generic errors, fail closed | C11 | SECURITY-03, 09, 15 |
| Auth and authorization failure metrics for alerting | C2 through C11 | SECURITY-14 |
| Key storage (hash only), secrets in Secrets Manager | C2, configuration | SECURITY-12 |
| Encryption, IAM, network, access logging | Infrastructure Design | SECURITY-01, 02, 06, 07 |

**Misuse cases designed for**:
- guessing or reusing another client's plan ID
- brute-forcing keys
- event details slipped into busy blocks
- prompt injection through course text or student text, contained by structured outputs plus ID validation
- retried selections
- oversized inputs

## 6. Open Items Carried Forward

| Item | Stage |
|---|---|
| Access logging for the Function URL, which has no gateway access logs (SECURITY-02). Rely on Lambda request logging plus CloudWatch, or add CloudFront in front | Infrastructure Design |
| RDS connection pooling (RDS Proxy or another option), VPC and NAT layout, IaC tool | Infrastructure Design |
| Per-step model timeouts and the latency budget | NFR Design |
| `ProfileResponder`: scripted answers or a simulated student (R5) | Functional Design (benchmark unit) |
| Remote MCP API-key header support in Claude Desktop (R2) | Early spike in the MCP surface unit |

## 7. Extension Compliance (this stage)

| Rule | Status | Note |
|---|---|---|
| SECURITY-03, 05, 08, 09, 11, 12, 15 | Compliant (design level) | Placed in dedicated components as shown in section 5. A misuse scenario is documented (SECURITY-11). |
| SECURITY-14 | Compliant (design level) | Metrics and events defined. Alarms, retention, and log integrity are set in Infrastructure Design. |
| SECURITY-01, 02, 06, 07, 10, 13 | N/A at this stage | Infrastructure and supply-chain controls are designed in Infrastructure Design and Code Generation. SECURITY-02 is tracked in section 6. |
| SECURITY-04 | N/A | No HTML-serving endpoints |
| PBT-02, 03, 07, 08, 09 | N/A at this stage (no tests yet) | The pure C6 engine and the Pydantic round-trips are designed so PBT can target them |

There are no blocking findings.
