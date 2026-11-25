# Requirements Verification Questions — CourseFit

These questions cover gaps or ambiguities left by `docs/vision.md` and `docs/technical-environment.md`. Topics those documents already settle are not asked here. That includes the tech stack, the prohibited libraries, the 2 MB calendar limit, back-to-back meetings counting as no conflict, the in-memory session state, and the four benchmark models.

Fill in the letter after each `[Answer]:` tag. If no option fits, choose the last option (Other) and describe what you want after the tag. Add notes freely, since the team will review these together.

---

## Section A — Scope and academic coverage

## Question 1
Which roster terms should the term picker offer for the December 16 demo?

A) Only Spring 2027 (SP27), which is the term students are pre-enrolling for now

B) Fall 2026 and Spring 2027, the two terms of the 2026–27 catalog year

C) Every term `config/rosters.json` returns, with requirement checks shown only for terms covered by a verified catalog year

X) Other (please describe after [Answer]: tag below)

[Answer]: A (answered in chat)

## Question 2
Which requirement categories must be verified and offered in the picker by the demo? (The vision says to add coverage "one requirement at a time.")

A) A small fixed set: 2–3 Information Science BA categories (e.g., introductory programming, one concentration area) and 2–3 Business minor categories (e.g., financial accounting, one elective group)

B) All Business minor categories plus the Information Science BA core only, with concentrations left out

C) Whatever the team can verify by a cutoff date, as long as there is at least one category per program

X) Other (please describe after [Answer]: tag below)

[Answer]: A (answered in chat after clarification of what 'verified and offered in the picker' means)

## Question 3
Is selecting a requirement optional? Can a student ask for a general elective ("any three-credit course about AI policy") without choosing a major, minor, or requirement?

A) Yes. With no requirement selected, CourseFit searches every offering in the term

B) Yes, but the search is limited to a curated subset (e.g., departments related to the student's stated interests) to keep retrieval fast

C) No. Every search requires a supported requirement

X) Other (please describe after [Answer]: tag below)

[Answer]: A (answered in chat)

## Question 4
How should rules with conditions be handled, such as the Business minor's unique-credit or no-double-counting conditions? Checking them requires knowing the student's other courses.

A) Show the condition as an unchecked caveat on the result ("you must confirm this course isn't counted toward your major")

B) Ask the student a yes/no question about the condition when it applies, then treat the answer as a hard constraint

C) Leave out categories with such conditions until there is a transcript feature

X) Other (please describe after [Answer]: tag below)

[Answer]: X) Other: State every assumption clearly and let the student override it. Follow-up: an override flips the assumption for the session and CourseFit re-ranks.

## Question 5
Should CourseFit check prerequisites and enrollment restrictions (e.g., "permission of instructor," "juniors and seniors only," "prerequisite: CS 2110")?

A) No checks. Show the roster's restriction and prerequisite text on each result so the student can judge

B) Show the text, and also ask the student about their class year so obvious class-level restrictions can be filtered out

C) Treat prerequisites as preferences the student can confirm ("Have you taken CS 2110?") before a course is ranked

X) Other (please describe after [Answer]: tag below)

[Answer]: X) Other: Prerequisites are not strictly enforced at Cornell, so mention them but do not restrict results.

---

## Section B — Interview and recommendation behavior

## Question 6
How should natural-language constraints like "I work Fridays after noon" become binding constraints?

A) The model parses them into structured busy intervals or rules, and CourseFit echoes the interpretation back for the student to confirm before treating it as binding

B) The model parses them and they become binding right away. The result explanation shows how each one was interpreted

C) The student enters hard constraints through a structured form (day and time pickers). Only preferences come from free text

X) Other (please describe after [Answer]: tag below)

[Answer]: X) Other: Parse and bind immediately, but keep it editable through student follow-ups.

## Question 7
Should there be a ceiling on follow-up questions? The vision says there is no fixed count and the student can ask for results at any time.

A) No ceiling. The model decides when to stop based on whether another answer could change the shortlist

B) A soft ceiling (e.g., after 4 follow-ups CourseFit offers results by default), and the student can keep answering

C) A hard ceiling (e.g., at most 5 follow-ups) to bound cost and latency

X) Other (please describe after [Answer]: tag below)

[Answer]: B (answered in chat)

## Question 8
How many courses should a shortlist contain?

A) 5

B) Up to 10

C) A student-configurable number with a default of 5 and a maximum of 10

X) Other (please describe after [Answer]: tag below)

[Answer]: A (answered in chat)

## Question 9
When a course has several feasible section combinations (lecture + discussion + lab), what does a result show?

A) One recommended combination (chosen by preferences such as time of day), plus the number of other feasible combinations

B) Every feasible combination

C) One recommended combination, with the student able to expand and see all the alternatives before selecting

X) Other (please describe after [Answer]: tag below)

[Answer]: C (answered in chat)

## Question 10
How should variable-credit courses (e.g., 1–4 credits) count toward the target credits and per-course credit limits?

A) The student picks the credit value when selecting the course. Ranking assumes the maximum value that still fits the remaining credits

B) Always assume the minimum credit value

C) Always assume the maximum credit value

X) Other (please describe after [Answer]: tag below)

[Answer]: A (answered in chat)

## Question 11
What should course retrieval (search beyond exact keywords) be built on for the MVP? The docs call retrieval a benchmark decision, but construction needs a starting point.

A) Lexical search (e.g., BM25) over course titles and descriptions, with the model expanding the student's interests into search terms, then reranking the candidates

B) Embedding-based semantic search (embeddings from an API reachable through OpenRouter or a similar hosted provider), followed by model reranking

C) Build both behind one interface so the benchmark can compare them. Lexical search is the default

X) Other (please describe after [Answer]: tag below)

[Answer]: A for now (answered in chat). Semantic search may be explored later.

## Question 12
The discovery evaluation compares a keyword-search shortlist with a personalized one. Should the keyword baseline be a mode of the CourseFit service itself?

A) Yes. A `baseline` mode on the recommendation tool that uses the same feasible offerings and fit checks, but plain keyword matching and no interview

B) No. The evaluation script builds the baseline outside the service from the same cached roster data

X) Other (please describe after [Answer]: tag below)

[Answer]: X) Other: Remove the keyword baseline feature entirely. Follow-up: remove the discovery evaluation (keyword vs personalized comparison) as well.

## Question 13
What should happen if an OpenRouter call fails or times out mid-interview?

A) Retry once, then return a clear error to the host. The planning session stays intact so the student can try again

B) Retry once, then fall back to keyword-ranked feasible results labelled as unpersonalized

C) Let the student switch to a different configured model and continue the same session

X) Other (please describe after [Answer]: tag below)

[Answer]: X) Other: Retry once, then call a configured fallback model. Accept failure if OpenRouter itself is down. Follow-up: fallback is disabled during benchmark runs.

---

## Section C — Calendar handling

## Question 14
How should all-day events in an imported calendar (holidays, "Out of office," birthdays) be treated?

A) Ignore all-day events. Only timed events produce busy intervals

B) Treat all-day events as busy for the whole day

C) Ignore them by default, and list them in the import summary so the student can mark any as busy

X) Other (please describe after [Answer]: tag below)

[Answer]: B (answered in chat; superseded: the host now parses calendars, so this becomes guidance to the host)

## Question 15
How should events marked "free" or "tentative" (iCalendar `TRANSP:TRANSPARENT` or `STATUS:TENTATIVE`) be treated?

A) Free (transparent) events are ignored. Tentative events count as busy

B) Both are ignored. Only confirmed, opaque events count as busy

C) Both count as busy

X) Other (please describe after [Answer]: tag below)

[Answer]: A (answered in chat; superseded: the host now parses calendars, so this becomes guidance to the host)

---

## Section D — Clients, hosts, and data pipeline

## Question 16
Which MCP host(s) must the team verify CourseFit against for the demo (launched over stdio)?

A) Claude Desktop only

B) Claude Desktop and one coding-agent host (Claude Code or Codex)

C) Any MCP host that supports stdio tools. No specific host is required beyond an automated in-process test client

X) Other (please describe after [Answer]: tag below)

[Answer]: X) Other: Claude Desktop plus Claude Code or Codex.

## Question 17
What is the "client agent" in the React demo? The docs say it "presents questions and collects answers."

A) No LLM in the client. The React UI calls CourseFit's tools directly through the bridge, shows CourseFit's next question as text, and sends the student's typed answer back

B) The bridge runs its own small host LLM loop (through OpenRouter) that turns the student's chat into structured tool calls, like a third-party host would

X) Other (please describe after [Answer]: tag below)

[Answer]: X) Other: Remove the React demo completely. Demo in Claude Desktop or Codex to dogfood the MCP.

## Question 18
How are catalog program rules produced and stored?

A) An offline scraper script extracts candidate rules into a version-controlled data file (JSON or YAML). A teammate reviews and marks each rule verified. The service loads only verified rules at startup

B) The service scrapes catalog pages live at startup and exposes only the rules on a hand-maintained allow-list

X) Other (please describe after [Answer]: tag below)

[Answer]: A (answered in chat)

## Question 19
How should Cornell roster data be cached?

A) On-disk cache keyed by term and query, with a time-to-live (e.g., 24 hours), shared across sessions and restarts

B) In-memory cache only, rebuilt each time the service starts

C) A snapshot script that pre-fetches the demo term's offerings to disk ahead of time, plus live lookups as a fallback

X) Other (please describe after [Answer]: tag below)

[Answer]: X) Other: Pre-download plus live fallback, with a daily cache refresh.

## Question 20
How should a planning session end and its private data be discarded? (The docs require discarding it when the session ends but don't define "ends.")

A) An explicit `end_session` tool, plus an idle timeout (e.g., 30 minutes), plus process exit

B) Only when the MCP process exits (one session per host connection)

C) Idle timeout and process exit only. No explicit tool

X) Other (please describe after [Answer]: tag below)

[Answer]: X) Other: end_session tool plus a one-month idle timeout. Sessions are stored in a database on a hosted MCP backend (see follow-ups in audit.md).

---

## Section E — Evaluation and quality

## Question 21
Is the model benchmark harness part of what we build in AI-DLC Construction?

A) Yes. A scripted harness that runs the fixed scenarios against each `COURSEFIT_MODEL`, records model ID, latency, token cost (from OpenRouter usage data), questions asked, and the shortlist, and writes a results file

B) Partly. The service records latency and cost per interview in structured, privacy-safe logs, and the team runs and compares scenarios by hand

C) No. Evaluation happens outside the codebase

X) Other (please describe after [Answer]: tag below)

[Answer]: A (answered in chat)

## Question 22
What accessibility and usability bar should the React demo client meet?

A) Basic: semantic HTML, keyboard navigable, readable contrast. Nothing formally audited

B) WCAG 2.1 AA for the core flow (inputs, questions, results, selection)

C) None beyond "works in a current desktop Chrome for the live demo"

X) Other (please describe after [Answer]: tag below)

[Answer]: N/A: React demo removed

---

## Section F — AI-DLC extension opt-ins

## Question: Security Extensions
Should security extension rules be enforced for this project?

Note: the technical-environment document already sets privacy requirements (in-memory calendars, no event details in logs or model requests, loopback-only bridge, key kept server-side). This extension would add a broader baseline of blocking security rules on top of those.

A) Yes — enforce all SECURITY rules as blocking constraints (recommended for production-grade applications)

B) No — skip all SECURITY rules (suitable for PoCs, prototypes, and experimental projects)

X) Other (please describe after [Answer]: tag below)

[Answer]: A (Yes, enforce)

## Question: Resiliency Extensions
Should the resiliency baseline be applied to this project?

**What this extension is.** Enabling it applies a set of **directional, design-time best practices** for building resilient systems, derived from the **AWS Well-Architected Framework (Reliability Pillar)** and resilience-review guidance. It steers requirements, design, and code toward fault tolerance, high availability, observability, and recoverability — covering 15 practice areas across business goals, change management, observability, high availability, disaster recovery, and continuous improvement.

**What this extension is NOT.** Enabling it does **not** make your workload production-ready, nor does it certify or guarantee any availability, RTO, or RPO target. It is a **starting point** that scaffolds good resiliency decisions early — it is not a substitute for a formal **AWS Well-Architected Review** of the built system.

Treat the output as a well-grounded **first draft of your resiliency posture** to build on and validate — not a finished, production-certified result.

A) Yes — apply the resiliency baseline as directional best practices and design-time guidance (recommended for business-critical workloads, as an informed starting point that you can validate and harden before go-live)

B) No — skip the resiliency baseline (suitable for PoCs, prototypes, and experimental projects where rapid iteration matters more than reliability)

X) Other (please describe after [Answer]: tag below)

[Answer]: X) Other: Don't worry about HA and DR. Recorded as No (skip the resiliency baseline).

## Question: Property-Based Testing Extension
Should property-based testing (PBT) rules be enforced for this project?

Note: CourseFit has pure, rule-heavy logic that suits PBT well, such as interval overlap, recurrence expansion, section-combination feasibility, and credit arithmetic.

A) Yes — enforce all PBT rules as blocking constraints (recommended for projects with business logic, data transformations, serialization, or stateful components)

B) Partial — enforce PBT rules only for pure functions and serialization round-trips (suitable for projects with limited algorithmic complexity)

C) No — skip all PBT rules (suitable for simple CRUD applications, UI-only projects, or thin integration layers with no significant business logic)

X) Other (please describe after [Answer]: tag below)

[Answer]: B (Partial: pure logic only)
