# Application Design Plan — CourseFit

**Inputs**: `requirements.md`, `stories.md`, `personas.md`, `execution-plan.md`
**Outputs** (in `aidlc-docs/inception/application-design/`): `components.md`, `component-methods.md`, `services.md`, `component-dependency.md`, `application-design.md`

## Design Questions (asked in chat; answers recorded here)

## Question 1
How does a slow request (interview turn or shortlist, possibly 30–60 s) reach the host without hitting API Gateway's roughly 29-second timeout?

A) Lambda Function URL in front of the MCP handler (Lambda can run up to 15 minutes), with CourseFit's own API-key auth and rate limiting

B) API Gateway, with tools split so that no single call waits on long model work (start the work, then poll for results)

C) Run the MCP service in a container (App Runner or ECS Fargate) instead of Lambda

X) Other (please describe after [Answer]: tag below)

[Answer]: A (Lambda Function URL; answered in chat)

## Question 2
How fine-grained should the MCP tools be?

A) Hybrid: typed tools for structured actions (start plan, set busy blocks, select, remove, list, resume, end) plus one conversational tool for free text (answers, constraints, corrections, overrides) that returns either the next question or results

B) Fully typed: a separate tool for every action, including answering a question and adding a constraint

C) Minimal: a single conversational tool that handles everything

X) Other (please describe after [Answer]: tag below)

[Answer]: A (Hybrid; answered in chat)

## Question 3
How should CourseFit use the LLM inside one interview turn?

A) A code-driven pipeline of small, structured LLM steps (extract preferences and constraints, decide the next question or stop, expand search terms, rerank and explain). Code decides the order and checks each output

B) An agent loop: the model calls CourseFit's internal functions (search, fit-check, and so on) as tools and decides the order itself

X) Other (please describe after [Answer]: tag below)

[Answer]: A (Code-driven steps; answered in chat)

## Question 4
How should the code be packaged and deployed?

A) One Python package with a module per component. One Lambda serves MCP requests, and small Lambdas reuse the same package for scheduled jobs. The scheduling engine is pure functions with no database or network access

B) Separate Lambda functions or services per component (catalog, scheduling, interview), calling each other

X) Other (please describe after [Answer]: tag below)

[Answer]: A (One package, few Lambdas; answered in chat)

## Execution Checklist

- [x] 1. Identify components and their responsibilities from the requirements and the 32 stories
- [x] 2. Define the MCP tool contract (tool names, inputs, outputs, error shapes) using the Q2 answer
- [x] 3. Define component interfaces and method signatures (business rules deferred to Functional Design)
- [x] 4. Define the service layer (orchestrators) and the per-turn flow using the Q3 answer
- [x] 5. Define dependencies, communication patterns, and data flow, including the request path from the Q1 answer and scheduled jobs
- [x] 6. Place security controls (auth guard, ownership check, validation, redaction, error handler) in dedicated components (SECURITY-11)
- [x] 7. Generate components.md
- [x] 8. Generate component-methods.md
- [x] 9. Generate services.md
- [x] 10. Generate component-dependency.md (with a validated Mermaid diagram and a text alternative)
- [x] 11. Generate application-design.md (consolidated)
- [x] 12. Validate completeness against the stories and consistency across the artifacts
