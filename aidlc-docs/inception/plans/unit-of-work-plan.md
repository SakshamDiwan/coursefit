# Unit of Work Plan — CourseFit

**Inputs**: application design (C1–C14, S1–S6), `stories.md` (32 stories), `execution-plan.md` (preliminary unit candidates)
**Deployment model**: modular monolith. One Python package, one MCP Lambda, and job Lambdas (AD-4). Units are **modules**, not separately deployable services.

## Draft Decomposition (to be confirmed by the questions below)

| Draft unit | Components and services |
|---|---|
| U1 Platform and Access | C2, C3, C11, C12 (cleanup), S1, IaC |
| U2 Cornell Data | C4 (including scraper), C5, C12 (refresh), S5 data operations, C13 catalog and roster commands |
| U3 Scheduling Engine | C6, S4 |
| U4 Interview and Recommendation | C7, C8, C9, C10, S2, S3 |
| U5 MCP Surface and Evaluation | C1, C14, S6, host verification, C13 key commands |

## Questions (asked in chat; answers recorded here)

## Question 1
How should the shared contracts (Plan aggregate, Course/Section/Meeting models, MCP tool schemas, ToolError) be handled?

A) A small Unit 0 "Shared Contracts", written jointly in the first 1–2 days before the five units start

B) Owned by U1 Platform and Access, with others reviewing

C) Each unit defines its own models and the team reconciles them later

X) Other (please describe after [Answer]: tag below)

[Answer]: A (Unit 0 Shared Contracts, written jointly first; answered in chat)

## Question 2
How should the Construction stages run across units?

A) In parallel: each owner runs the AI-DLC Construction stages for their unit on their own branch. U0 merges first, and integration happens in Build and Test

B) Sequentially in one shared session: unit by unit, in dependency order

X) Other (please describe after [Answer]: tag below)

[Answer]: A (In parallel per owner on branches; answered in chat)

## Question 3
Should unit owners be assigned now?

A) Leave owners TBD, to be assigned at the team meeting

B) Assign owners now (names provided in the answer)

X) Other (please describe after [Answer]: tag below)

[Answer]: A (Owners TBD, assigned at team meeting; answered in chat)

## Question 4
What code layout should the repository use?

A) `src/coursefit/<component>/` (one installable package), `tests/<unit>/`, plus top-level `infra/`, `rules/`, `benchmark/scenarios/`

B) `src/<unit>/` folders per unit (AI-DLC's default monolith layout), `tests/<unit>/`

X) Other (please describe after [Answer]: tag below)

[Answer]: X) Other: Use uv workspaces. One workspace member per logical unit, where a unit is either an app or a package. Apps: MCP app deployed on Lambda, admin CLI, benchmark. Packages: reusable components used by apps. Follow-ups: tests live per package with shared assets (infra/, rules/, benchmark/scenarios/, tests/integration/) at the root. Services live in a shared 'planning' package. Unit set confirmed (see below).

## Confirmed Decomposition (from answers)

uv workspace with `packages/` (reusable libraries) and `apps/` (runnable or deployable). 9 units, owners TBD.

| Unit | Kind | Workspace member | Components and services |
|---|---|---|---|
| U0 Contracts | package | `packages/coursefit-contracts` | Shared Pydantic models: Plan aggregate, Course/SectionGroup/Section/Meeting, busy blocks, TurnResult, ToolError, IDs |
| U1 Platform | package | `packages/coursefit-platform` | C2 Access, C3 Plan Store, C11 Observability, plan cleanup logic |
| U2 Cornell | package | `packages/coursefit-cornell` | C4 Catalog Rules (+ scraper), C5 Roster Data (+ refresher) |
| U3 Schedule | package | `packages/coursefit-schedule` | C6 Scheduling Engine (pure) |
| U4 Recommend | package | `packages/coursefit-recommend` | C7 Retrieval, C8 Model Gateway, C9 Interview, C10 Recommender |
| U5 Planning | package | `packages/coursefit-planning` | S1 Plan, S2 Conversation, S3 Recommendation, S4 Selection services |
| U6 MCP Server | app | `apps/mcp-server` | C1 MCP Interface, C12 job handlers (same bundle, separate Lambda entry points), `infra/` IaC |
| U7 Admin CLI | app | `apps/admin-cli` | C13 (keys, catalog scrape and check, roster inspect and snapshot) |
| U8 Benchmark | app | `apps/benchmark` | C14, S6 (calls U5 services directly, fallback disabled) |

## Generation Checklist (Part 2)

- [x] 1. Finalize unit boundaries and responsibilities from the answers
- [x] 2. Document the code organization strategy (greenfield) in unit-of-work.md
- [x] 3. Generate `unit-of-work.md` with definitions, responsibilities, components, interfaces owned, and the definition of done per unit
- [x] 4. Generate `unit-of-work-dependency.md` with the dependency matrix, integration contracts, build order, and parallelization
- [x] 5. Generate `unit-of-work-story-map.md` mapping all 32 stories (and demo scenarios S1–S10) to units
- [x] 6. Validate that every story is assigned, there are no circular dependencies, and every component has exactly one owning unit
- [x] 7. Record per-unit Construction stage applicability (Functional Design, NFR, Infrastructure)
