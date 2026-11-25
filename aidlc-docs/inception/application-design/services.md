# CourseFit Services

Services are thin orchestrators in `coursefit/services/`. MCP tools call them. They coordinate components and own transaction boundaries (load plan, then change it, then save with a version check). They contain no fit or ranking logic themselves.

---

## S1 — PlanService
**Tools**: `start_plan`, `update_goal`, `set_busy_blocks`, `list_plans`, `get_plan`, `end_plan`
**Uses**: C3 Plan Store, C4 Catalog Rules (to validate `requirement_id`), C6 (to validate busy blocks against the term window)

- **Start or update a goal**: validate the term (SP27) and the credits, and check that the requirement is verified. If the requirement isn't supported, return `invalid_input` with the supported list, and leave the plan unchanged.
- **Set busy blocks**: validate the whole set first, then replace the old set atomically. On any validation error nothing is written (FR-3.4).
- **Get plan, list plans, end plan**: these are always scoped to the client. `end_plan` deletes the plan immediately.

## S2 — ConversationService
**Tool**: `say`
**Uses**: C9 Interview Engine, S3 RecommendationService, C3

The per-turn pipeline (AD-3) is driven by code:

```
1. load plan (owner already verified)
2. interp   = InterviewEngine.interpret(plan, message)          # LLM step 1 (structured)
3. plan     = InterviewEngine.apply(plan, interp)               # pure: constraints, prefs, overrides, answers
4. if interp.wants_results            -> result = Recommendation   (step 6)
   elif only constraint/override edits and results were shown before
                                       -> result = Recommendation  (refresh)
   else next = InterviewEngine.decide_next(plan)                 # LLM step 2 + soft ceiling in code
        Ask  -> result = question
        Stop -> result = Recommendation
5. save plan (version check)
6. Recommendation = S3.recommend(plan)
7. return TurnResult with interpretation_notes (how each constraint was read)
```

## S3 — RecommendationService
**Tools**: `get_recommendations`, `list_section_options`, and internally `say` and the selection tools
**Uses**: C10 Recommender, C7 Retrieval, C5 Roster, C4 Catalog, C6 Scheduling

```
1. blocked   = schedule.blocked_time(plan, selected courses, SP27 window)
2. pool      = requirement ? catalog.eligible_course_ids(rule) resolved via roster : all SP27 courses
3. terms     = expand_terms(plan)                               # LLM step 3 (skipped if no interests yet)
4. retrieved = retriever.search(terms, limit=N, restrict_to=pool)   # widen N if too few survive step 5
5. feasible  = [c for c in retrieved
                if credit_fit(c) ok and feasible_combinations(c, blocked) and catalog.match(rule, c).matched]
6. if feasible empty -> conflict: explain_infeasibility(pool, blocked, plan)
7. ranked    = rerank_and_explain(plan, evidence(feasible))     # LLM step 4
8. ranked    = validate_ranked(ranked, ids(feasible))           # drop anything not verified
9. top 5     -> choose_combination per course; attach assumptions, prereq text, async flag, counts
```

Model inputs (steps 3 and 7) contain only preference text, interview answers, and course evidence. Busy blocks and the client ID never enter a prompt (NFR-2.6).

## S4 — SelectionService
**Tools**: `select_course`, `remove_course`
**Uses**: C6, C5, C3, S3

- **Select**:
  1. Load the course and the chosen combination.
  2. Recheck its fit against the current blocked time.
  3. Check that the credits are valid.
  4. Call `apply_selection`, which is idempotent, so the same selection twice gives the same plan.
  5. Save.
  6. Return the plan summary with a refreshed shortlist from S3.
- **Remove**: call `remove_selection`, save, and refresh.
- **Infeasible selection**: return `infeasible_selection` with the reason. The plan is unchanged.

## S5 — DataOpsService (jobs and admin)
**Used by**: C12 Scheduled Jobs, C13 Admin CLI
**Uses**: C5, C3, C2, C4

- `refresh_roster(term)`: fetch at 1 request per second, map the fields, then swap the snapshot atomically. On failure, keep the previous snapshot and log the error.
- `cleanup_idle_plans()`: call `delete_idle(30 days)`.
- `issue_key` and `revoke_key`: pass through to C2.
- `scrape_catalog` and `check_catalog`: pass through to C4's offline tools.

## S6 — BenchmarkService
**Used by**: C14
**Uses**: S1–S4 in process, with `ModelGateway(allow_fallback=False)`

The service runs each scenario:
1. Start the plan.
2. Set its busy blocks.
3. Call `say` with the opening message.
4. Answer each question through `ProfileResponder`.
5. Record the metrics and the shortlist.
6. Check the hard constraints.

---

## Cross-Cutting Orchestration Rules
- **One request, one plan transaction.** Optimistic version checks prevent lost updates when an assistant retries.
- **Fail closed.** Any error before `save` leaves the plan unchanged. Authorization errors return `not_found`.
- **Model failures.** When the primary model and the fallback both fail, the service returns `model_unavailable` and the plan is not saved (FR-8.2, NFR-3.2).
- **Latency budget (target 60 s).** About 2–4 model calls per turn. Each has a timeout, and the request stays inside the Function URL's Lambda timeout. Specific budgets are set in NFR Design.
