# CourseFit Personas

These personas are fictional archetypes drawn from the vision document. The three student personas cover the main demo situations: a busy schedule, an empty schedule with open-ended interests, and a requirement with conditions. The operator persona covers the work the team does to run CourseFit.

---

## P1 — Jordan, the working student

| | |
|---|---|
| **Who** | Junior, Information Science BA, works about 15 hours a week at a campus job |
| **Reaches CourseFit through** | Claude Desktop, with Google Calendar connected |
| **Goal** | Find 6 more credits for Spring 2027, including one Info Sci requirement, that don't clash with work shifts |
| **Situation** | Busy calendar with recurring shifts, a weekly club meeting, and a few one-off commitments. Works Friday afternoons, and the shift isn't on their calendar. |
| **Frustrations** | Finds a lecture that fits, then learns the required lab conflicts with work. Checks times by hand across Class Roster tabs. |
| **What success looks like** | Every suggested section set actually fits. When a constraint they type is misread, they can fix it in one message. |

**Key scenarios**: busy schedule, required-lab conflict, plain-language constraint and correction, second selection changes what fits.

---

## P2 — Sam, the curious elective-seeker

| | |
|---|---|
| **Who** | Sophomore, undeclared, leaning toward Government |
| **Reaches CourseFit through** | Claude Code or Codex (also uses it for coursework) |
| **Goal** | "Something about how cities decide whether to use facial recognition." Three credits, any department |
| **Situation** | Doesn't share a calendar (empty schedule). Has no requirement in mind. Doesn't know which departments or terms to search. |
| **Frustrations** | Keyword search on the roster returns nothing useful, or only CS courses, when the interest is policy and ethics |
| **What success looks like** | A follow-up question that sharpens what they want, and a shortlist with at least one course from a department they wouldn't have tried |

**Key scenarios**: empty schedule, no requirement, adaptive follow-ups, asking for results early, explanations grounded in course descriptions.

---

## P3 — Riley, the minor-focused planner

| | |
|---|---|
| **Who** | Junior, Biological Sciences major, Business minor |
| **Reaches CourseFit through** | Claude Desktop |
| **Goal** | Satisfy a Business minor category (e.g., financial accounting) in Spring 2027 and reach 9 credits across two or three courses |
| **Situation** | Moderate calendar. Unsure whether a course might double-count with their major. Some candidate courses are variable-credit. Comes back over several days as plans change. |
| **Frustrations** | Catalog pages list options but don't say what's offered this term. Unique-credit rules are confusing. |
| **What success looks like** | Results that clearly say which category a course matches and what CourseFit assumed. Can override an assumption. The plan is still there next week. |

**Key scenarios**: requirement selected, unsupported requirement, assumption override, variable credits, building toward a credit goal, resuming a saved plan, deleting a plan.

---

## P4 — Team operator

| | |
|---|---|
| **Who** | A member of the CourseFit team (any of the five) |
| **Reaches CourseFit through** | Repository scripts, AWS console or CLI, pull requests |
| **Goal** | Keep CourseFit's data correct and current, control who can use it, and choose the demo model using evidence |
| **Responsibilities** | Issue and revoke API keys. Scrape and verify catalog rules. Load and refresh Spring 2027 roster data. Configure primary and fallback models. Run the benchmark harness. |
| **Frustrations** | Unverified rules leaking into the picker. A stale or missing roster on demo day. Benchmark runs that can't be compared. |
| **What success looks like** | Repeatable one-command operations, and a clear record of what was verified or measured and when |

---

## Persona-to-Epic Map

| Epic | P1 Jordan | P2 Sam | P3 Riley | P4 Operator |
|---|:-:|:-:|:-:|:-:|
| E1 Set the academic goal | ✓ | ✓ | ✓ | |
| E2 Share availability and constraints | ✓ | ✓ | ✓ | |
| E3 Answer useful questions | ✓ | ✓ | ✓ | |
| E4 Compare explained recommendations | ✓ | ✓ | ✓ | |
| E5 Build toward a credit goal | ✓ | | ✓ | |
| E6 Access and saved plans | ✓ | ✓ | ✓ | |
| E7 Operate CourseFit | | | | ✓ |
