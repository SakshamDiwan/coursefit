# CourseFit Component Dependencies

## Dependency Matrix

A row "uses" a column. ✓ means a direct call. The scheduling engine (C6) depends on nothing.

| Uses → | C2 Access | C3 Plans | C4 Catalog | C5 Roster | C6 Schedule | C7 Retrieval | C8 Model | C9 Interview | C10 Recommend | C11 Observ. |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| C1 MCP Interface | ✓ | | | | | | | | | ✓ |
| S1 PlanService | | ✓ | ✓ | | ✓ | | | | | ✓ |
| S2 ConversationService | | ✓ | | | | | | ✓ | (via S3) | ✓ |
| S3 RecommendationService | | | ✓ | ✓ | ✓ | ✓ | | | ✓ | ✓ |
| S4 SelectionService | | ✓ | | ✓ | ✓ | | | | (via S3) | ✓ |
| S5 DataOpsService | ✓ | ✓ | ✓ | ✓ | | | | | | ✓ |
| C9 Interview Engine | | | | | | | ✓ | | | ✓ |
| C10 Recommender | | | ✓ | | ✓ | ✓ | ✓ | | | ✓ |
| C7 Retrieval | | | | (data) | | | | | | |
| C14 Benchmark | | | | | | | | | | ✓ (and S1–S4) |

## Communication Patterns
- **Host to CourseFit**: MCP over Streamable HTTP (stateless) to a **Lambda Function URL** over HTTPS, with `Authorization: Bearer <api key>`.
- **Within the Lambda**: in-process Python calls only. No network hops between components (AD-4).
- **CourseFit to PostgreSQL (RDS)**: TLS, parameterized queries, pooled connections (pooling approach decided in Infrastructure Design).
- **CourseFit to OpenRouter**: HTTPS through C8 only.
- **CourseFit to the Cornell Class Roster API**: HTTPS through C5 only, at most 1 request per second. Normally used only by the refresh job and for cache misses.
- **Scheduled jobs**: EventBridge schedules invoke the job Lambdas (same package).
- **Offline**: the operator runs the admin CLI to scrape the catalog into `rules/2026-27.yaml`, a teammate verifies it in a pull request, and the file is bundled with the deployment.

## Component Diagram

```mermaid
flowchart LR
    Host["MCP Host<br/>Claude Desktop / Claude Code / Codex"]
    Op["Operator CLI (C13)"]
    Sched["EventBridge schedules"]

    subgraph Lambda["CourseFit Lambda (Function URL)"]
        C1["C1 MCP Interface"]
        C2["C2 Access and Identity"]
        S1["S1 PlanService"]
        S2["S2 ConversationService"]
        S3["S3 RecommendationService"]
        S4["S4 SelectionService"]
        C9["C9 Interview Engine"]
        C10["C10 Recommender"]
        C6["C6 Scheduling Engine (pure)"]
        C7["C7 Retrieval BM25"]
        C4["C4 Catalog Rules"]
        C5["C5 Roster Data"]
        C3["C3 Plan Store"]
        C8["C8 Model Gateway"]
    end

    subgraph Jobs["Job Lambdas"]
        C12["C12 Roster refresh and plan cleanup"]
    end

    DB[("PostgreSQL on RDS<br/>plans, keys, roster cache")]
    OR["OpenRouter"]
    CU["Cornell Class Roster API"]
    RF["rules/2026-27.yaml (verified)"]

    Host -->|"HTTPS + API key"| C1
    C1 --> C2
    C1 --> S1
    C1 --> S2
    C1 --> S3
    C1 --> S4
    S2 --> C9
    S2 --> S3
    S4 --> S3
    S4 --> C6
    S3 --> C10
    C10 --> C6
    C10 --> C7
    C10 --> C8
    C9 --> C8
    S3 --> C4
    S3 --> C5
    S1 --> C3
    S2 --> C3
    S4 --> C3
    C2 --> DB
    C3 --> DB
    C5 --> DB
    C4 --> RF
    C8 --> OR
    C5 --> CU
    Sched --> C12
    C12 --> C5
    C12 --> C3
    Op --> C2
    Op --> C4
    Op --> C5
```

### Text Alternative

```
MCP Host --(HTTPS + API key)--> C1 MCP Interface
C1 -> C2 Access (auth, rate limit, ownership) -> DB
C1 -> S1 PlanService -> C3 Plan Store -> DB ; C4 Catalog (validate requirement)
C1 -> S2 ConversationService -> C9 Interview -> C8 Model Gateway -> OpenRouter
                             -> S3 RecommendationService
C1 -> S3 RecommendationService -> C4 Catalog (rules file), C5 Roster (DB cache, Cornell API on miss),
                                  C10 Recommender -> C7 Retrieval, C6 Scheduling (pure), C8 Model Gateway
C1 -> S4 SelectionService -> C6 Scheduling, C3 Plan Store, S3 (refresh)
EventBridge -> C12 Jobs -> C5 Roster refresh, C3 idle cleanup
Operator CLI -> C2 keys, C4 scrape/check, C5 inspect/snapshot
```

## Data Flow: One `say` Turn

```
Host: say(plan_id, "I work Fridays after noon; more into policy than coding")
 1. C1 validate -> C2 authenticate + rate limit + require_plan_owner
 2. C3 load plan (version v)
 3. C9 interpret          [LLM]  -> constraint add (Fri 12:00-24:00), preference: policy > coding
 4. C9 apply              [pure]
 5. C9 decide_next        [LLM + ceiling]  -> Ask or Stop
 6a. Ask  -> TurnResult(question)
 6b. Stop -> S3: blocked_time [pure] -> pool -> expand_terms [LLM] -> BM25 -> feasibility [pure]
            -> rerank_and_explain [LLM] -> validate IDs -> top 5 -> TurnResult(results)
 7. C3 save plan (expect v)
 8. C1 return TurnResult + interpretation notes
```

## Coupling Notes for Units Generation
- **C6 has no dependencies.** It can be built and property-tested first, from fixtures.
- **C4 and C5 depend only on files, the database, and HTTP.** The fit and retrieval work can begin from a small fixture roster before the real snapshot exists.
- **C9 and C10 depend on C8.** Tests use a fake `ModelGateway`, so the interview work doesn't wait on OpenRouter access.
- **Shared contracts must be agreed first:** the `Plan` aggregate, the `Course`/`SectionGroup`/`Meeting` models, and the MCP tool schemas. These are the integration points between teammates.
