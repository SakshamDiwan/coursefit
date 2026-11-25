# CourseFit User Stories

**Organization**: six student-journey epics (E1–E6) plus one operator epic (E7).
**Size**: each story is roughly 1–3 days of work for one teammate.
**Acceptance criteria**: Given/When/Then for core flows, and a checklist for simpler stories.
**Personas**: P1 Jordan (working student), P2 Sam (elective-seeker), P3 Riley (minor planner), P4 Team operator. See [personas.md](personas.md).

In the stories, "the assistant" means the student's MCP host (Claude Desktop, Claude Code, or Codex). The student talks to the assistant, and the assistant calls CourseFit.

---

## E1 — Set the academic goal

### US-1.1 Start a plan for Spring 2027
**As** any student, **I want** to start a plan by giving my target term and total credits, **so that** CourseFit knows what I'm planning for.
**Personas**: P1, P2, P3 · **Requirements**: FR-1.2, FR-2.1, FR-2.5

- **Given** an authenticated client, **when** the assistant starts a plan for SP27 with 9 target credits, **then** CourseFit returns a new session ID owned by that client, with remaining credits set to 9.
- **Given** a request for any term other than SP27, **when** a plan is started, **then** CourseFit rejects it and names SP27 as the supported term.
- **Given** a per-course credit limit of 4, **when** the plan is created, **then** it is stored separately from the 9-credit total.
- **Given** target credits that are zero, negative, or above a sane maximum (e.g., 30), **when** a plan is started, **then** the input is rejected with a clear validation message.

### US-1.2 Browse supported requirements
**As** a student with a major or minor goal, **I want** to see which requirement categories CourseFit can check, **so that** I know what it can and can't verify for me.
**Personas**: P1, P3 · **Requirements**: FR-2.2, FR-2.3, FR-7.2

- [ ] The list contains only categories marked verified, with program, catalog year (2026–27), category name, and source URL.
- [ ] Unverified or draft categories never appear.
- [ ] The demo set has 2–3 Information Science BA categories and 2–3 Business minor categories.

### US-1.3 Choose a requirement, or none
**As** a student, **I want** to attach one supported requirement to my plan or leave it empty, **so that** results either satisfy that requirement or search the whole term.
**Personas**: P1, P2, P3 · **Requirements**: FR-2.4, FR-5.2

- **Given** a plan with the Business minor financial accounting category, **when** results are produced, **then** every course matches that category's verified rule and names the category.
- **Given** a plan with no requirement, **when** results are produced, **then** candidates come from every SP27 offering.
- **Given** a student asks for a requirement that isn't supported, **when** the assistant tries to set it, **then** CourseFit says it can't check that requirement, lists what it does support, and leaves the plan unchanged. *(Demo scenario S4)*

---

## E2 — Share availability and constraints

### US-2.1 Send busy times from my calendar
**As** a student with a calendar, **I want** my assistant to send CourseFit my busy times, **so that** recommendations avoid them without CourseFit seeing my event details.
**Personas**: P1, P3 · **Requirements**: FR-3.1, FR-3.3, NFR-2.1

- **Given** weekly blocks (e.g., Tue/Thu 13:00–17:00 from 2027-01-20 until 2027-05-05) and one-off dated blocks, **when** the assistant submits them, **then** CourseFit stores them and reports how many weekly and one-off blocks are active.
- **Given** a block without a UTC offset, **when** it is stored, **then** it is interpreted in America/New_York.
- **Given** a submission containing any field other than days, times, and dates (e.g., an event title), **when** it is validated, **then** the extra field is rejected and never stored.
- [ ] The tool description tells hosts to skip all-day events and events marked free, include tentative events, and send times only.

### US-2.2 Start with an empty schedule
**As** a student who doesn't want to share a calendar, **I want** to plan without one, **so that** I can still get recommendations.
**Personas**: P2 · **Requirements**: FR-3.2

- **Given** a plan with no busy blocks, **when** results are requested, **then** CourseFit returns a shortlist and no course is excluded for schedule reasons. *(Demo scenario S2)*

### US-2.3 Replace my busy times safely
**As** a student whose schedule changed, **I want** to resend my busy times, **so that** results reflect the new schedule, and a bad update doesn't wipe the old one.
**Personas**: P1, P3 · **Requirements**: FR-3.4, NFR-3.2

- **Given** existing busy blocks, **when** a valid new set is submitted, **then** the old set is fully replaced.
- **Given** existing busy blocks, **when** an invalid set is submitted (e.g., end before start, an unknown day, too many blocks), **then** CourseFit rejects it and the previous set is unchanged.

### US-2.4 Add a constraint in plain language
**As** a student, **I want** to say things like "I work Fridays after noon," **so that** I don't have to translate my life into a form.
**Personas**: P1 · **Requirements**: FR-3.5, FR-3.6, FR-4.4

- **Given** the student says "I work Fridays after noon," **when** CourseFit processes it, **then** it stores a binding constraint (Friday 12:00 to end of day), applies it immediately, and the response states the interpretation in plain words.
- **Given** the student says "I prefer mornings," **when** CourseFit processes it, **then** it is stored as a preference, not a binding constraint.
- **Given** a binding constraint, **when** results are produced, **then** no recommended section meets during it. *(Demo scenario S1)*

### US-2.5 Correct or remove a constraint
**As** a student whose constraint was misread, **I want** to fix it with a follow-up message, **so that** good courses aren't hidden by mistake.
**Personas**: P1 · **Requirements**: FR-3.5

- **Given** a stored constraint "Friday 12:00 onward," **when** the student says "actually only until 5pm," **then** the constraint becomes Friday 12:00–17:00, the shortlist is recomputed, and the response shows the change. *(Demo scenario S7)*
- **Given** a stored constraint, **when** the student asks to remove it, **then** it no longer affects results.

---

## E3 — Answer useful questions

### US-3.1 Get a follow-up that matters
**As** a student with a vague interest, **I want** CourseFit to ask me a question only when my answer could change the results, **so that** the conversation stays short and useful.
**Personas**: P2, P1, P3 · **Requirements**: FR-4.1, FR-4.4

- **Given** "I'm interested in startups and finance," **when** CourseFit decides what to do next, **then** it either asks a question that separates two plausible directions (e.g., building forecasting models versus using financial statements for decisions) or returns results. It does not ask about anything the student already said.
- **Given** an answer, **when** it is recorded, **then** explicit must-haves are stored separately from inferred preferences.
- [ ] Each follow-up's purpose (what it might change) is recorded for the value-per-question evaluation.

### US-3.2 Results by default after about four questions
**As** a student, **I want** CourseFit to show results after a handful of questions, **so that** I'm not stuck in an endless interview.
**Personas**: P1, P2, P3 · **Requirements**: FR-4.2

- **Given** four follow-ups have been answered, **when** CourseFit decides the next step, **then** it returns results by default and offers to keep refining.
- **Given** the student chooses to keep refining, **when** they answer more questions, **then** CourseFit continues and the shortlist updates.

### US-3.3 Ask for results any time
**As** an impatient student, **I want** results right now, even before any questions, **so that** I stay in control.
**Personas**: P2 · **Requirements**: FR-4.3

- **Given** a plan with only a term and credits, **when** the student asks for results, **then** CourseFit returns a shortlist based on what it knows and says that answering questions could improve it.

---

## E4 — Compare explained recommendations

### US-4.1 See a feasible five-course shortlist
**As** a student, **I want** five courses that genuinely fit my schedule, credits, and requirement, **so that** I can trust any of them.
**Personas**: P1, P2, P3 · **Requirements**: FR-5.1, FR-5.3, FR-5.5, FR-5.9, FR-5.10, FR-6.1

- **Given** busy blocks, binding constraints, and prior selections, **when** a shortlist is produced, **then** every recommended lecture, discussion, and lab passes every hard constraint on every date it meets. *(Demo scenario S1)*
- **Given** a meeting that ends at 15:00 and a busy block starting at 15:00, **when** fit is checked, **then** they do not conflict. A one-minute overlap does conflict.
- **Given** a course whose lecture fits but whose only required lab conflicts, **when** fit is checked, **then** the course is not recommended. *(Demo scenario S3)*
- **Given** a course with a missing meeting time or an unresolved catalog-to-roster mapping, **when** a requirement is selected, **then** it cannot count as a verified requirement match.
- **Given** model output naming a course or section outside the verified feasible set, **when** results are assembled, **then** that item is discarded.
- [ ] Explicitly asynchronous sections are labeled as asynchronous and never cause a conflict.

### US-4.2 Understand why each course was recommended
**As** a student, **I want** each result to explain how it connects to what I said, citing the course description, **so that** I can judge the recommendation.
**Personas**: P2, P3 · **Requirements**: FR-6.1, NFR-2.6

- **Given** a shortlist, **when** it is returned, **then** each result includes the course, credits, a recommended section set with times and class numbers, any matched requirement, an explanation citing the course description, and trade-offs (e.g., "meets at 8:40am, and you prefer afternoons").
- [ ] Every claim in an explanation is supported by the cited course data or rule (checked in evaluation).
- [ ] The model request contains only course evidence and preferences, never busy blocks or client identity.

### US-4.3 See alternative section sets
**As** a student, **I want** to see the other section combinations that fit for a course, **so that** I can pick a time I like better.
**Personas**: P1, P3 · **Requirements**: FR-6.1, FR-6.2

- **Given** a result with "3 other feasible combinations," **when** the student asks to see them, **then** CourseFit lists all of them with times and class numbers, and each one passes the fit checks.

### US-4.4 See prerequisites without being blocked
**As** a student, **I want** to see a course's prerequisites and restrictions, **so that** I can decide for myself, since Cornell doesn't strictly enforce them.
**Personas**: P1, P2, P3 · **Requirements**: FR-5.8

- [ ] Each result shows the roster's prerequisite and enrollment-restriction text, labeled "not checked by CourseFit."
- [ ] No course is excluded or demoted because of prerequisite text.

### US-4.5 See and override CourseFit's assumptions
**As** a minor-focused student, **I want** CourseFit to tell me what it assumed (e.g., that a course won't double-count with my major) and let me correct it, **so that** results match my real situation.
**Personas**: P3 · **Requirements**: FR-5.7

- **Given** a selected category with a unique-credit condition, **when** a matching course is shown, **then** the result states the assumption CourseFit made.
- **Given** the student says "that course would double-count with my major," **when** CourseFit records the override, **then** the course no longer counts toward the category, the shortlist refreshes, and the override stays in the session. *(Demo scenario S8)*

### US-4.6 Learn why nothing fits
**As** a student with tight constraints, **I want** CourseFit to tell me which constraint is blocking every option, **so that** I can decide what to relax.
**Personas**: P1 · **Requirements**: FR-6.3

- **Given** must-haves that no SP27 course satisfies, **when** results are requested, **then** CourseFit returns no shortlist, names the blocking constraint(s) (e.g., "every matching course meets during your Tue/Thu afternoon block"), and waits for the student's decision. It does not silently relax anything. *(Demo scenario S6)*

---

## E5 — Build toward a credit goal

### US-5.1 Select a course and update my plan
**As** a student, **I want** to pick a section set and have CourseFit update my remaining credits and refresh the options, **so that** I can build my semester one choice at a time.
**Personas**: P1, P3 · **Requirements**: FR-6.4

- **Given** 9 target credits, **when** the student selects a 3-credit course's recommended section set, **then** CourseFit rechecks fit, records the selection, adds its meetings to the plan, shows 6 credits remaining, and refreshes the shortlist.
- **Given** a first selection, **when** the shortlist refreshes, **then** courses that now conflict with the selected meetings are gone. *(Demo scenario S5)*
- **Given** a selection that no longer fits (e.g., busy blocks changed), **when** it is submitted, **then** it is rejected with the reason and the plan is unchanged.

### US-5.2 Selecting twice doesn't double-count
**As** a student whose assistant retried a request, **I want** a repeated selection to have no extra effect, **so that** my credits stay correct.
**Personas**: P1, P3 · **Requirements**: FR-6.4

- **Given** a selection already recorded, **when** the same selection is submitted again, **then** the plan and remaining credits are identical to after the first submission.

### US-5.3 Choose credits for a variable-credit course
**As** a student, **I want** to pick how many credits I'll take in a 1–4 credit course, **so that** my credit total is right.
**Personas**: P3 · **Requirements**: FR-5.6

- **Given** 3 remaining credits and a 1–4 credit course, **when** it is ranked, **then** CourseFit assumes 3 credits and says so.
- **Given** the student selects it with 2 credits, **when** the selection is recorded, **then** remaining credits drop by 2. *(Demo scenario S9)*
- **Given** a credit value outside the course's allowed range or above the remaining or per-course limit, **when** it is submitted, **then** the selection is rejected.

### US-5.4 Remove a selection
**As** a student who changed their mind, **I want** to remove a selected course, **so that** its credits and meetings are freed up.
**Personas**: P1, P3 · **Requirements**: FR-6.4

- [ ] Removing a selection restores its credits and removes its meetings from fit checks.
- [ ] The shortlist refreshes afterwards.
- [ ] Removing a selection that doesn't exist returns a clear message and changes nothing.

---

## E6 — Access and saved plans

### US-6.1 Connect with my API key
**As** a student, **I want** to add CourseFit to my assistant with my personal key, **so that** only I can use my plans.
**Personas**: P1, P2, P3 · **Requirements**: FR-1.1, NFR-2.3, NFR-2.8

- **Given** a valid key, **when** the assistant calls any tool, **then** the request is processed as that key's client.
- **Given** a missing, unknown, or revoked key, **when** any tool is called, **then** the request is rejected with a generic authentication error and no data is returned.
- **Given** repeated failed authentication attempts, **when** the threshold is crossed, **then** an operator alert fires and the requests are rate-limited.
- [ ] Verified working in Claude Desktop and in Claude Code or Codex.

### US-6.2 Resume a saved plan
**As** a student coming back days later, **I want** to list my plans and pick up where I left off, **so that** I don't start over.
**Personas**: P3 · **Requirements**: FR-1.3

- **Given** a client with two saved plans, **when** the assistant lists plans, **then** it sees only that client's plans with term, credits, selections, and last activity.
- **Given** a plan ID, **when** the assistant resumes it, **then** goals, busy blocks, constraints, preferences, assumptions, answers, and selections are all restored. *(Demo scenario S10)*

### US-6.3 My plans are private to me
**As** a student, **I want** nobody else to read or change my plans, even if they learn a plan ID, **so that** my schedule stays private.
**Personas**: P1, P2, P3 · **Requirements**: FR-1.3, NFR-2.4

- **Given** client A's plan ID, **when** client B calls any tool with it, **then** CourseFit responds as if the plan doesn't exist and logs an authorization failure (without plan contents).
- [ ] Every tool that takes a plan ID enforces this check. There is an automated test per tool.

### US-6.4 Delete my plan
**As** a student, **I want** to delete a plan when I'm done, **so that** my data doesn't linger.
**Personas**: P1, P2, P3 · **Requirements**: FR-1.4

- **Given** a plan, **when** `end_session` is called by its owner, **then** the plan and all its data are deleted immediately and any later call with that ID behaves as if it never existed.

### US-6.5 Forgotten plans expire
**As** a student who stops using CourseFit, **I want** my idle plans deleted automatically, **so that** my data isn't kept forever.
**Personas**: P1, P2, P3 · **Requirements**: FR-1.4

- [ ] A scheduled job deletes plans with no activity for 30 days.
- [ ] Any activity resets the 30-day timer.
- [ ] The job logs how many plans it deleted, without their contents.

---

## E7 — Operate CourseFit

### US-7.1 Issue and revoke API keys
**As** an operator, **I want** to issue a key for a classmate and revoke it later, **so that** I control who can use the demo.
**Personas**: P4 · **Requirements**: FR-1.1, NFR-2.3

- [ ] One command creates a client and prints a new high-entropy key once. Only a hash of the key is stored.
- [ ] One command revokes a key. Requests with it fail right away.
- [ ] Keys never appear in logs.

### US-7.2 Scrape catalog rules into a reviewable file
**As** an operator, **I want** a script that pulls requirement rules from the catalog pages into a file in the repo, **so that** teammates can review them in a pull request.
**Personas**: P4 · **Requirements**: FR-7.1

- [ ] Running the script for the IS BA and Business minor pages writes candidate rules with program, catalog year, category, permitted courses or conditions, and source URL.
- [ ] New or changed rules are marked unverified.
- [ ] Rerunning the script on unchanged pages produces the same file.

### US-7.3 Verify rules before students see them
**As** an operator, **I want** only rules a teammate approved to reach the requirement picker, **so that** students never get an unchecked rule.
**Personas**: P4 · **Requirements**: FR-7.2, FR-2.3

- **Given** a rule marked verified through a reviewed pull request, **when** the service starts, **then** the rule is available.
- **Given** an unverified rule, **when** the service starts, **then** it is ignored.
- [ ] The chosen demo categories (2–3 per program) and their source URLs are recorded.

### US-7.4 Load and refresh Spring 2027 roster data
**As** an operator, **I want** SP27 course data loaded ahead of time and refreshed daily, **so that** demos are fast and don't depend on live Cornell calls.
**Personas**: P4 · **Requirements**: FR-7.3, FR-7.4, FR-7.5, NFR-1.2

- [ ] The SP27 field mapping is inspected and recorded before use.
- [ ] A snapshot job loads offerings into the cache: sections, meetings, class numbers, credits, descriptions, prerequisite and restriction text.
- [ ] A scheduled job refreshes the cache daily.
- [ ] Live lookups fill gaps when an item isn't cached.
- [ ] All Cornell requests stay at or below one per second.
- [ ] A failed refresh leaves the previous snapshot in place and is logged.

### US-7.5 Configure primary and fallback models
**As** an operator, **I want** to set the primary and fallback models, **so that** a flaky model call doesn't break a student's session.
**Personas**: P4 · **Requirements**: FR-8.1, FR-8.2, FR-8.3

- **Given** the primary model times out, **when** CourseFit retries once and it fails again, **then** the call goes to `COURSEFIT_FALLBACK_MODEL` and the answering model is recorded.
- **Given** OpenRouter is unavailable entirely, **when** a call fails, **then** the student gets a clear, generic error and the session is unchanged.

### US-7.6 Benchmark candidate models
**As** an operator, **I want** one command that runs the fixed scenarios against each candidate model, **so that** we pick the demo model using evidence.
**Personas**: P4 · **Requirements**: FR-9.1, FR-9.2, NFR-1.1

- [ ] The harness runs each scenario per model with the fallback disabled.
- [ ] Per run, it records model ID, latency, cost from OpenRouter usage, question count, what each answer revealed, and the final shortlist.
- [ ] Scenario follow-ups are answered from a fixed student profile, so runs are comparable.
- [ ] Results are written to a file that compares models, including whether the 60-second shortlist target was met.

---

## Demo Correctness Scenarios

The vision calls for ten demo scenarios in which every recommendation passes every hard constraint. These are the proposed ten, with the stories whose acceptance criteria back them. The team can swap scenarios before the demo.

| ID | Scenario | Persona | Backed by |
|---|---|---|---|
| S1 | Busy schedule plus a plain-language work constraint | P1 | US-2.4, US-4.1 |
| S2 | Empty schedule, no requirement | P2 | US-2.2, US-1.3 |
| S3 | Lecture fits but required lab conflicts | P1 | US-4.1 |
| S4 | Unsupported requirement requested | P3 | US-1.3 |
| S5 | Second selection changes what fits | P1 | US-5.1 |
| S6 | No course satisfies the must-haves | P1 | US-4.6 |
| S7 | Misread constraint corrected | P1 | US-2.5 |
| S8 | Double-count assumption overridden | P3 | US-4.5 |
| S9 | Variable-credit course selected at a lower value | P3 | US-5.3 |
| S10 | Plan resumed in a later session | P3 | US-6.2 |

---

## Story Summary and Traceability

| Epic | Stories | Requirements covered |
|---|---|---|
| E1 Set the academic goal | US-1.1 – 1.3 | FR-1.2, FR-2.1 – 2.5, FR-5.2, FR-7.2 |
| E2 Share availability and constraints | US-2.1 – 2.5 | FR-3.1 – 3.6, FR-4.4, NFR-2.1, NFR-3.2 |
| E3 Answer useful questions | US-3.1 – 3.3 | FR-4.1 – 4.4 |
| E4 Compare explained recommendations | US-4.1 – 4.6 | FR-5.1 – 5.10, FR-6.1 – 6.3, NFR-2.6 |
| E5 Build toward a credit goal | US-5.1 – 5.4 | FR-5.6, FR-6.4 |
| E6 Access and saved plans | US-6.1 – 6.5 | FR-1.1 – 1.4, NFR-2.3, NFR-2.4, NFR-2.8 |
| E7 Operate CourseFit | US-7.1 – 7.6 | FR-1.1, FR-7.1 – 7.5, FR-8.1 – 8.3, FR-9.1 – 9.2, NFR-1.1, NFR-1.2 |

**Total**: 32 stories.

Cross-cutting non-functional requirements apply to every story and are verified in design and Build and Test rather than as standalone stories:
- encryption (NFR-2.2)
- input validation (NFR-2.5)
- logging (NFR-2.7)
- errors (NFR-2.9)
- supply chain (NFR-2.10)
- network and IAM (NFR-2.11)
- testing and property-based testing (NFR-4.x)

**INVEST check**: Each story delivers a value a student or operator can observe and can be estimated at 1–3 days. Each is testable through its acceptance criteria. Stories within an epic share data but can be built independently behind the session and catalog interfaces. Stories that depend on the fit checker (US-4.1) are noted for Units Generation.

**Misuse cases covered** (security baseline SECURITY-11):
- reading another client's plan (US-6.3)
- invalid or revoked keys and brute-force attempts (US-6.1)
- event details smuggled into busy blocks (US-2.1)
- retried selections (US-5.2)
- model output naming courses that weren't verified (US-4.1)
