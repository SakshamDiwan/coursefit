# CourseFit Requirements

## Intent Analysis

| Item | Assessment |
|---|---|
| **User request** | "Using AI-DLC, I want to start a new project. Please read docs/vision and docs/technical-environment, then begin the AI-DLC workflow." |
| **Request type** | New project (greenfield) |
| **Scope** | System-wide: a hosted MCP service, Cornell catalog and roster data pipeline, schedule fit checks, LLM interview and ranking, a Postgres session store, AWS deployment, and a model benchmark harness |
| **Complexity** | Complex |
| **Requirements depth** | Comprehensive |
| **Deadline** | MVP demo on 2026-12-16 |

The inputs are [docs/vision.md](../../../docs/vision.md), [docs/technical-environment.md](../../../docs/technical-environment.md), and the team's answers in [requirement-verification-questions.md](requirement-verification-questions.md) (collected in chat on 2026-11-25 and recorded in `audit.md`).

## Decisions That Change the Input Documents

The team's answers overrode parts of the original vision and technical-environment documents. Both documents were updated to match on 2026-11-25. If they ever conflict again, this document takes precedence.

| # | Input documents said | Now |
|---|---|---|
| D1 | A React demo client and a loopback FastAPI bridge | **Removed.** The demo runs in Claude Desktop and Claude Code or Codex, so the team uses the MCP service the way students will. |
| D2 | CourseFit parses `.ics` files (icalendar, recurring-ical-events) and owns calendar privacy handling | **The student's AI assistant reads the calendar** and sends CourseFit busy times in CourseFit's structured format. An `.ics` file is optional and never reaches CourseFit. |
| D3 | Local stdio service, in-memory sessions, no persistent stores for student data | **Hosted service on AWS Lambda with PostgreSQL on RDS.** Sessions persist, keyed by session ID and client ID. |
| D4 | Cloud hosting is outside the pilot | **Cloud hosting is in scope**, with a simple architecture and sensible defaults. |
| D5 | Discovery evaluation: classmates compare a keyword-search list with CourseFit's list (target: 3 of 5) | **Removed**, together with any keyword-baseline feature. |
| D6 | No fallback model | **Retry once, then use a configured fallback model.** Fallback is off during benchmark runs. |

---

## 1. Functional Requirements

### 1.1 Access and sessions

- **FR-1.1 Client authentication.** Every MCP request must carry a per-user API key. The team generates the keys and hands them out. The server maps each key to a **client ID**. A request with no key or an unknown key is rejected.
- **FR-1.2 Planning sessions.** A client can create one or more planning sessions. Each session has a server-generated **session ID** and belongs to exactly one client ID. A session holds:
  - the target term
  - the catalog year
  - the selected requirement (optional)
  - total target credits
  - an optional per-course credit limit
  - busy blocks
  - binding constraints
  - preferences
  - recorded assumptions
  - interview answers
  - selected section combinations
- **FR-1.3 Resume.** A client can list its sessions and resume one by session ID. The server must refuse any session that the requesting client doesn't own.
- **FR-1.4 Session end.** An `end_session` tool deletes a session and all of its data right away. A session with no activity for **30 days** is deleted automatically.

### 1.2 Academic goal input

- **FR-2.1 Term.** The only supported term for the MVP is **Spring 2027 (SP27)**, under the 2026–27 catalog.
- **FR-2.2 Requirement picker.** CourseFit lists supported requirement categories with program, catalog year, and source URL. Each category must already be verified (see FR-7.2).
- **FR-2.3 Coverage.** The demo includes a small fixed set: **2–3 Information Science BA categories and 2–3 Business minor categories.** The team chooses them before construction and records them with their source URLs.
- **FR-2.4 Optional requirement.** A student can search without choosing a requirement. In that case CourseFit considers every SP27 offering.
- **FR-2.5 Credits.** The student gives total desired credits. A per-course credit limit is optional and kept separate from the total.

### 1.3 Availability and constraints

- **FR-3.1 Busy-block format.** CourseFit accepts busy time from the host in two forms:
  - **weekly blocks**: day of week, start time, end time, and optional `from` and `until` dates
  - **one-off dated blocks**: start and end datetimes

  All times are interpreted in `America/New_York` unless an explicit offset is given.
- **FR-3.2 Empty availability.** A session with no busy blocks is valid.
- **FR-3.3 Host guidance.** The tool descriptions tell the host how to build busy blocks from a calendar:
  - skip all-day events
  - skip events marked free (`TRANSP:TRANSPARENT`)
  - include tentative events
  - send only times, with no titles, attendees, locations, or notes

  CourseFit's input schema has no fields that could hold event details.
- **FR-3.4 Replacing busy blocks.** Submitting a new busy-block set replaces the previous one. If a submission fails validation, the previous set stays unchanged.
- **FR-3.5 Natural-language constraints.** A statement like "I work Fridays after noon" is parsed by CourseFit's model into a structured constraint and **binds immediately**. The response shows how it was interpreted. The student can change or remove it with a follow-up, and the shortlist is recomputed.
- **FR-3.6 Constraint types.** Calendar busy blocks and explicit must-haves are **binding**. Statements such as "I prefer mornings" are **preferences** that ranking can trade off.

### 1.4 Interview

- **FR-4.1 Adaptive follow-ups.** CourseFit draws interests, learning objectives, career goals, and preferences from what the student has already said. It asks a follow-up only when the answer could change which courses are found or their order.
- **FR-4.2 Soft ceiling.** After about **4 follow-ups**, CourseFit shows results by default. The student can keep refining.
- **FR-4.3 Results on demand.** The student can ask for results at any point, including before any follow-up.
- **FR-4.4 Must-haves vs. inferences.** Explicit must-haves are stored separately from preferences the model inferred.

### 1.5 Retrieval, fit checks, and ranking

- **FR-5.1 Retrieval.** The model expands the student's interests into search terms. A lexical (BM25-style) search over SP27 course titles and descriptions retrieves candidates. The model then reranks a top set. Retrieval sits behind an interface so semantic search can be tried later.
- **FR-5.2 Requirement check.** When a requirement is selected, only courses that match its verified rule qualify. Catalog identifiers and verified cross-listings are joined to roster offerings. A course whose mapping is unresolved, or that has a missing meeting time, can't support a verified match.
- **FR-5.3 Section feasibility.** CourseFit tests every allowed combination of lecture, discussion, and lab against:
  - busy blocks
  - binding constraints
  - meetings of already-selected courses

  Intervals are timezone-aware with exclusive end times, so back-to-back meetings don't conflict. Meetings apply only on the dates they occur, including partial-term sessions.
- **FR-5.4 Asynchronous sections.** Sections that are explicitly asynchronous are labeled separately and never cause a conflict.
- **FR-5.5 Credit check.** A course must fit the remaining target credits and any per-course limit.
- **FR-5.6 Variable credits.** Ranking assumes the largest credit value that still fits the remaining credits and states that assumption. The student picks the actual value when selecting the course.
- **FR-5.7 Conditional rules.** For rule conditions CourseFit can't check without the student's history (unique credit, double counting), it states the assumption it made on the result. The student can override that assumption. The override is stored in the session, affected courses are re-evaluated, and the shortlist refreshes.
- **FR-5.8 Prerequisites and restrictions.** These are **shown, not enforced.** Each result includes the roster's prerequisite and enrollment-restriction text, labeled as not checked.
- **FR-5.9 Shortlist size.** A shortlist has **5 courses**.
- **FR-5.10 Model output validation.** The final response accepts only course and section identifiers from the verified feasible candidate set. Any other identifier is discarded.

### 1.6 Results and explanations

- **FR-6.1 Result contents.** Each result includes:
  - the course and its credits
  - one recommended feasible section combination, with meeting times and class numbers
  - the number of other feasible combinations
  - the matched requirement, if any
  - an explanation linking the student's priorities to evidence in the course description
  - trade-offs, stated assumptions, and prerequisite or restriction text
- **FR-6.2 Alternative combinations.** The student can ask to see every feasible section combination for a result before selecting one.
- **FR-6.3 No feasible results.** If no course satisfies every binding constraint, CourseFit names the blocking constraints and waits for the student to decide what to change.
- **FR-6.4 Selection.** When the student selects a section combination, CourseFit:
  - rechecks that it fits
  - records the selection and its credits once (repeating the same selection is idempotent)
  - updates the remaining credits
  - adds the meetings to the plan
  - refreshes the shortlist

  The student can also remove a selection.

### 1.7 Cornell data

- **FR-7.1 Catalog scraper.** An offline script scrapes the 2026–27 Information Science BA and Business minor pages into a version-controlled data file. Each candidate rule records program, catalog year, category, permitted courses or conditions, and source URL.
- **FR-7.2 Rule verification.** A teammate reviews each rule against the published page and marks it `verified` through a pull request. The service loads only verified rules.
- **FR-7.3 Roster data.** A snapshot job fetches SP27 offerings from the Class Roster API v2.0 into the shared Postgres cache:
  - sections
  - meetings
  - class numbers
  - credits
  - descriptions
  - prerequisite and restriction text

  The cache is refreshed **daily**. Live lookups fill gaps.
- **FR-7.4 Roster rate limit.** All roster calls are limited to 1 request per second. Only public Cornell data is cached.
- **FR-7.5 Field inspection.** Before CourseFit relies on SP27 roster fields, the team inspects them and records the field mapping.

### 1.8 Model access

- **FR-8.1 Primary model.** Inference goes through OpenRouter. The primary model is set with `COURSEFIT_MODEL`.
- **FR-8.2 Retry and fallback.** A failed or timed-out call is retried once, then sent to `COURSEFIT_FALLBACK_MODEL`. If OpenRouter is down entirely, CourseFit returns a clear error and leaves the session unchanged.
- **FR-8.3 Model recording.** Every model call records which model actually answered.

### 1.9 Benchmark harness

- **FR-9.1 Scenario runs.** A scripted harness runs the fixed scenarios against each candidate model with fallback disabled. For each run it records:
  - model ID
  - latency
  - token cost from OpenRouter usage data
  - number of questions asked
  - what each answer revealed
  - the final shortlist
- **FR-9.2 Candidate models.** The initial candidates are `google/gemini-3.1-flash-lite`, `deepseek/deepseek-v4-flash`, `anthropic/claude-sonnet-4.6`, and `anthropic/claude-opus-4.8`. Other models can be added.

---

## 2. Non-Functional Requirements

### 2.1 Performance
- **NFR-1.1** A shortlist is returned within **60 seconds** of the student's last answer (demo target).
- **NFR-1.2** Requests should normally be served from the roster cache. A cold-cache search must stay within the 1 request/second limit to Cornell.

### 2.2 Security and privacy (Security Baseline extension enforced)
- **NFR-2.1 Minimal storage.** CourseFit stores only what planning needs: busy blocks, goals, constraints, preferences, assumptions, interview answers, and selections. It never stores raw calendar contents or event details.
- **NFR-2.2 Encryption.** Data is encrypted in transit (TLS 1.2+ on every hop, including to RDS) and at rest (RDS storage encryption with KMS). *(SECURITY-01)*
- **NFR-2.3 API keys.** Keys are high-entropy random values, stored only as hashes, and can be revoked. Secrets (OpenRouter key, DB credentials) live in AWS Secrets Manager, never in code or IaC. *(SECURITY-12)*
- **NFR-2.4 Session ownership.** Every tool that references a session ID verifies that the caller owns it. Access is denied by default. *(SECURITY-08)*
- **NFR-2.5 Input validation.** All tool inputs are validated with Pydantic schemas that enforce types, lengths, list sizes, and formats. All SQL is parameterized. Request body size is limited at the gateway. *(SECURITY-05)*
- **NFR-2.6 Model calls.** Requests to models contain only the course evidence and preference data needed for the task, never busy-block details or client identity. Text retrieved from Cornell and student text are treated as untrusted. Model output is validated against verified candidates (FR-5.10). *(SECURITY-11)*
- **NFR-2.7 Logging.** Logs are structured JSON in CloudWatch with timestamp, request ID, level, and message. They never include API keys, request bodies, busy blocks, or student free text. Log retention is at least 90 days. *(SECURITY-03, SECURITY-14)*
- **NFR-2.8 Rate limiting and alerts.** Requests are rate-limited per API key. Alerts fire on repeated authentication or authorization failures. *(SECURITY-11, SECURITY-14)*
- **NFR-2.9 Errors.** Users get generic error responses with no stack traces or internal details. A global error handler is in place, and the system fails closed. *(SECURITY-09, SECURITY-15)*
- **NFR-2.10 Supply chain.** Dependencies are pinned with lock files, scanned for vulnerabilities, and covered by an SBOM. *(SECURITY-10)*
- **NFR-2.11 Network and IAM.** IAM follows least privilege. RDS sits in private subnets and accepts traffic only from the Lambda security group. *(SECURITY-06, SECURITY-07)*

### 2.3 Reliability
- **NFR-3.1** The Resiliency Baseline extension is **not enabled**. High availability and disaster recovery are out of scope. Use single-AZ defaults and standard RDS automated backups.
- **NFR-3.2** Failed busy-block updates, selections, and model calls leave the session in its previous consistent state.

### 2.4 Maintainability and testability
- **NFR-4.1 Stack.** Python 3.12+, uv, the official Python MCP SDK, Pydantic, httpx, and Beautiful Soup. The prohibited-library rules in the technical-environment document still apply: no LangChain or CrewAI, no self-hosted models, and no browser automation for roster data. The persistent-store prohibition is lifted only for the session store defined here.
- **NFR-4.2 Tests and linting.** pytest for unit and integration tests, respx for HTTP fixtures, and an in-process MCP test client with a fake model. Ruff for linting.
- **NFR-4.3 Property-based testing (Partial mode).** Hypothesis is required for the deterministic core, enforcing PBT-02, 03, 07, 08, and 09. That covers:
  - interval overlap
  - weekly-block expansion
  - section-combination feasibility
  - credit arithmetic
  - schema round-trips
- **NFR-4.4 Required example tests.** Example-based tests must cover:
  - an empty calendar
  - the same selection retried twice
  - a lecture that fits while its required lab conflicts
  - back-to-back meetings
  - a one-minute overlap

### 2.5 Compatibility
- **NFR-5.1 Transport.** The service speaks MCP over Streamable HTTP.
- **NFR-5.2 Hosts.** It must be verified working in **Claude Desktop** and in **Claude Code or Codex**.

---

## 3. Evaluation (updated from the vision)

1. **Correctness.** Across the ten demo scenarios, every recommended section combination must pass every hard constraint. The scenarios must include:
   - busy availability
   - empty availability
   - a required-lab conflict
   - an unsupported requirement
   - a second selection that changes what fits
2. **Value per question.** Compare the shortlist before and after the follow-ups. Record how many questions were asked, what each answer revealed, and whether it improved the student's options. Check each explanation against the course or rule it cites.
3. **Model choice.** Use the benchmark harness (FR-9) to compare relevance, explanation accuracy, latency, and cost per completed interview.

The discovery comparison against keyword search has been removed (D5).

## 4. Out of Scope

- Transcript ingestion and degree audits
- Full-semester optimization and multi-semester planning
- Recommendations for labs, clubs, and events
- Enrollment and seat alerts
- Professor ratings
- NetID sign-in
- High availability and disaster recovery
- The React client and the FastAPI bridge
- Keyword-baseline mode
- `.ics` parsing inside CourseFit
- Semantic or embedding retrieval (possible later)

## 5. Risks and Items for Design Stages

- **R1: Request timeouts.** API Gateway's integration timeout (about 29–30 s by default) is below the 60-second target. Application or Infrastructure Design must pick an approach, such as a Lambda Function URL with streaming, a raised timeout quota, or a tool flow where no single call waits on long model work.
- **R2: Remote MCP auth in hosts.** Claude Desktop's remote connectors may not support a static API-key header the way Codex and Claude Code do. That needs to be checked early. A local proxy such as `mcp-remote` with a header might be the fallback.
- **R3: Lambda and Postgres.** Lambda connecting to Postgres needs VPC placement and connection pooling (RDS Proxy or equivalent). Lambda also needs a way to reach OpenRouter and Cornell from the VPC (a NAT gateway, which costs money).
- **R4: Scheduled jobs.** The daily roster refresh and the 30-day session cleanup need scheduled jobs, for example EventBridge triggering a Lambda.
- **R5: Simulated student in benchmarks.** The harness needs a way to answer follow-up questions from a fixed student profile, such as scripted answers or a simulated student model. This is decided in design.
- **R6: Schedule.** Three weeks to Dec 16 with expanded scope: hosting, persistence, and auth were added, while React and the bridge were removed.
