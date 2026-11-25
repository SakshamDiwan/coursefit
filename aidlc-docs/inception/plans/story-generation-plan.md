# Story Generation Plan — CourseFit

**Role**: Product owner
**Inputs**: `aidlc-docs/inception/requirements/requirements.md`, `docs/vision.md`, `docs/technical-environment.md`
**Outputs**: `aidlc-docs/inception/user-stories/personas.md`, `aidlc-docs/inception/user-stories/stories.md`

## Breakdown Approaches Considered

| Approach | Fit for CourseFit | Trade-off |
|---|---|---|
| User-journey-based | Follows the vision's six-step MVP experience (goal, availability, interview, recommendations, selection, resume) | Operator work doesn't fit a student journey |
| Feature-based | Maps cleanly to tools and modules | Loses the student's point of view, and acceptance criteria drift toward implementation |
| Persona-based | Keeps operator and student needs distinct | Student stories are spread across journey steps |
| Domain-based | Matches catalog, roster, scheduling, and interview domains | Again more about implementation than users |
| Epic-based | Gives a hierarchy the team can divide | Needs an organizing axis underneath it |

The planning questions below record the chosen approach, story size, acceptance-criteria format, and persona set.

## Planning Questions (asked in chat; answers recorded here)

## Question 1
Which personas should the stories cover?

A) Student personas only

B) Student personas plus a team operator persona (issues API keys, verifies catalog rules, refreshes roster data, runs benchmarks)

C) Student personas, a team operator, and the host assistant as a system actor

X) Other (please describe after [Answer]: tag below)

[Answer]: B (Students plus team operator; answered in chat)

## Question 2
How should stories be organized?

A) Epics following the student journey, plus one operator epic

B) Feature-based epics (one per tool or module)

C) Persona-based groups

X) Other (please describe after [Answer]: tag below)

[Answer]: A (Student journey epics plus one operator epic; answered in chat)

## Question 3
What acceptance-criteria format should stories use?

A) Given / When / Then scenarios

B) Plain checklist of verifiable statements

C) Given / When / Then for core flows, checklist for simple stories

X) Other (please describe after [Answer]: tag below)

[Answer]: C (Given/When/Then for core flows, checklist for simple stories; answered in chat)

## Question 4
How large should a story be?

A) Medium: one teammate can finish it in about 1–3 days

B) Small: about half a day each

C) Large: one per journey step

X) Other (please describe after [Answer]: tag below)

[Answer]: A (1-3 days each, roughly 25-35 stories; answered in chat)

## Execution Checklist (Part 2 — Generation)

- [x] 1. Draft personas.md with goals, context, frustrations, and how each persona reaches CourseFit
- [x] 2. Define the epics using the approved organization
- [x] 3. Write stories for academic goal input (term, credits, optional requirement, requirement picker)
- [x] 4. Write stories for availability and constraints (busy blocks, replacement, plain-language constraints, editing)
- [x] 5. Write stories for the interview (adaptive follow-ups, soft ceiling, results on demand)
- [x] 6. Write stories for recommendations (shortlist, explanations, alternative section sets, prerequisites shown, assumptions and overrides, no-feasible conflict)
- [x] 7. Write stories for building the plan (select, variable credits, remove, idempotent re-select, refreshed shortlist)
- [x] 8. Write stories for access and saved plans (API key authentication, list and resume, end_session, 30-day cleanup, ownership isolation)
- [x] 9. Write operator stories (issue and revoke keys, scrape and verify rules, roster snapshot and daily refresh, benchmark harness, model fallback configuration)
- [x] 10. Add acceptance criteria to every story in the approved format, including error and misuse cases (security baseline SECURITY-11)
- [x] 11. Check every story against INVEST (Independent, Negotiable, Valuable, Estimable, Small, Testable)
- [x] 12. Map personas to stories, and stories to requirement IDs (FR-x, NFR-x)
- [x] 13. Mark which acceptance criteria back the ten demo correctness scenarios
