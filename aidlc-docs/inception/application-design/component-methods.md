# CourseFit Component Methods

These signatures describe interfaces only. Detailed business rules come later in Functional Design, per unit. Types are Pydantic models or frozen dataclasses, named in `PascalCase`. "Plan" is the persisted planning session.

---

## C1 — MCP Interface

Every tool runs through the same chain: `authenticate` → `check_rate_limit` → validate the input schema → (for tools with a plan) `require_plan_owner` → call the service → `error_boundary`.

| # | Tool | Input | Output | Stories |
|---|---|---|---|---|
| 1 | `list_requirements` | `catalog_year: str = "2026-27"` | `list[RequirementSummary]` (id, program, category, catalog_year, source_url) | US-1.2 |
| 2 | `start_plan` | `term: Literal["SP27"]`, `target_credits: int`, `per_course_credit_limit: int \| None`, `requirement_id: str \| None` | `PlanSummary` | US-1.1, 1.3 |
| 3 | `update_goal` | `plan_id`, `target_credits?`, `per_course_credit_limit?`, `requirement_id?` (use `null` to clear) | `PlanSummary` | US-1.3 |
| 4 | `set_busy_blocks` | `plan_id`, `weekly: list[WeeklyBlockIn]`, `dated: list[DatedBlockIn]` (size limits; extra fields are rejected) | `BusyBlocksSummary` (counts, active date range) | US-2.1, 2.3 |
| 5 | `say` | `plan_id`, `message: str` (max length) | `TurnResult` | US-2.4, 2.5, 3.1–3.3, 4.5 |
| 6 | `get_recommendations` | `plan_id` | `TurnResult` (kind `results` or `conflict`) | US-3.3, 4.1, 4.2, 4.4, 4.6 |
| 7 | `list_section_options` | `plan_id`, `course_id` | `list[SectionCombination]` | US-4.3 |
| 8 | `select_course` | `plan_id`, `course_id`, `combination_id`, `credits: int \| None` | `SelectionResult` (plan summary plus refreshed `TurnResult`) | US-5.1–5.3 |
| 9 | `remove_course` | `plan_id`, `course_id` | `SelectionResult` | US-5.4 |
| 10 | `list_plans` | — | `list[PlanListItem]` | US-6.2 |
| 11 | `get_plan` | `plan_id` | `PlanDetail` (goal, busy-block counts, constraints, preferences, assumptions, selections, pending question) | US-6.2 |
| 12 | `end_plan` | `plan_id` | `{deleted: true}` | US-6.4 |

**`TurnResult`** is a discriminated union on `kind`:
- `question`: `{question, why_it_matters, questions_asked}`
- `results`: `{recommendations: list[Recommendation] (≤5), interpretation_notes, can_refine: bool}`
- `conflict`: `{blocking_constraints: list[BlockingConstraint], suggestions}`
- `ack`: `{interpretation_notes}`, returned when a message only updated constraints and results weren't requested

**`Recommendation`** has these fields:
- `course_id`, `subject`, `number`, `title`, `credits`, `assumed_credits`
- `recommended_combination: SectionCombination`, `other_feasible_count`
- `matched_requirement`
- `explanation`, `evidence_quotes`, `tradeoffs`
- `assumptions: list[Assumption]`
- `prerequisite_text`, `restriction_text` (labeled not checked)
- `is_async`

**Errors**: one generic `ToolError{code, message}` for all tools. Codes: `unauthenticated`, `rate_limited`, `not_found` (also used for plans the caller doesn't own), `invalid_input`, `infeasible_selection`, `model_unavailable`, `internal`.

---

## C2 — Access and Identity

```python
async def authenticate(bearer_token: str | None) -> ClientId                # raises Unauthenticated
async def check_rate_limit(client_id: ClientId) -> None                      # raises RateLimited
async def require_plan_owner(client_id: ClientId, plan_id: PlanId) -> None   # raises NotFound (never reveals existence)
async def issue_key(label: str) -> IssuedKey                                 # plaintext key returned once; only the hash is stored
async def revoke_key(client_id: ClientId) -> None
```

## C3 — Plan Store

```python
class PlanRepository(Protocol):
    async def create(self, client_id: ClientId, goal: Goal) -> Plan
    async def get(self, client_id: ClientId, plan_id: PlanId) -> Plan | None
    async def list(self, client_id: ClientId) -> list[PlanListItem]
    async def save(self, plan: Plan, expected_version: int) -> Plan        # optimistic concurrency; updates last_activity
    async def delete(self, client_id: ClientId, plan_id: PlanId) -> bool
    async def delete_idle(self, older_than: timedelta) -> int
```

The `Plan` aggregate holds:
- `id`, `client_id`, `version`, `term`, `catalog_year`, `goal`
- `busy: BusySet`, `constraints: list[Constraint]`, `preferences: Preferences`
- `assumption_overrides`, `interview: InterviewState` (answers, pending question, count)
- `selections: list[Selection]`
- `last_activity`

## C4 — Catalog Rules

```python
class RuleCatalog(Protocol):
    def list_supported(self, catalog_year: str) -> list[RequirementSummary]
    def get(self, requirement_id: str) -> RequirementRule | None            # verified only
    def eligible_course_ids(self, rule: RequirementRule) -> set[CourseKey]
    def match(self, rule: RequirementRule, course: Course,
              overrides: Mapping[str, bool]) -> RuleMatch                     # matched?, assumptions
# offline
def scrape_program(url: str) -> list[CandidateRule]                          # deterministic for same HTML
def load_rule_file(path: Path) -> RuleFile                                    # validates schema; drops unverified at runtime
```

## C5 — Roster Data

```python
class RosterClient:                                   # httpx, ≤1 req/s
    async def search_classes(self, roster: str, subject: str) -> list[RawClass]
    async def subjects(self, roster: str) -> list[str]
def map_class(raw: RawClass, field_map: FieldMap) -> Course                   # Course -> SectionGroups -> Sections -> Meetings
class RosterRepository(Protocol):
    async def all_courses(self, term: str) -> list[Course]
    async def get_course(self, term: str, course_id: CourseId) -> Course | None   # live fallback on miss
    async def resolve_cross_listing(self, term: str, key: CourseKey) -> CourseId | None
    async def replace_snapshot(self, term: str, courses: list[Course]) -> SnapshotInfo  # atomic swap
class RosterRefresher:
    async def refresh(self, term: str) -> RefreshReport                       # keeps previous snapshot on failure
```

## C6 — Scheduling Engine (pure)

```python
def expand_busy(busy: BusySet, term_window: DateRange, tz: ZoneInfo) -> list[Interval]
def expand_meetings(section: Section, term_window: DateRange, tz: ZoneInfo) -> list[Interval]
def overlaps(a: Interval, b: Interval) -> bool                                # exclusive end
def enumerate_combinations(course: Course) -> list[SectionCombination]
def check_combination(combo: SectionCombination, blocked: BlockedTime) -> FitResult   # ok | conflicts[]
def feasible_combinations(course: Course, blocked: BlockedTime) -> list[SectionCombination]
def blocked_time(plan: Plan, courses_by_id: Mapping[CourseId, Course], term_window: DateRange) -> BlockedTime
def credit_fit(course: Course, remaining: int, per_course_limit: int | None) -> CreditFit  # assumed credits or reason
def apply_selection(plan: Plan, selection: Selection, course: Course) -> Plan  # idempotent; raises InfeasibleSelection
def remove_selection(plan: Plan, course_id: CourseId) -> Plan
def remaining_credits(plan: Plan) -> int
def explain_infeasibility(candidates: list[Course], blocked: BlockedTime,
                          plan: Plan) -> list[BlockingConstraint]
```

## C7 — Retrieval

```python
class Retriever(Protocol):
    def search(self, terms: list[str], *, limit: int,
               restrict_to: set[CourseId] | None = None) -> list[ScoredCourse]
class Bm25Retriever(Retriever):
    @classmethod
    def build(cls, courses: list[Course]) -> "Bm25Retriever"
```

## C8 — Model Gateway

```python
class ModelGateway:
    async def complete_structured(self, task: TaskName, prompt: PromptPayload,
                                  schema: type[T], *, allow_fallback: bool = True) -> ModelResult[T]
# ModelResult: value: T, model_id, latency_ms, cost_usd, used_fallback
# PromptPayload: typed fields only (course evidence, preference text, transcript snippets); no busy/identity fields exist
```

## C9 — Interview Engine

```python
class InterviewEngine:
    async def interpret(self, plan: Plan, message: str) -> Interpretation
    #   Interpretation: preference_updates, must_haves, constraint_ops (add/edit/remove),
    #                   assumption_overrides, answered_question, wants_results, notes
    def apply(self, plan: Plan, interp: Interpretation) -> Plan                 # pure merge
    async def decide_next(self, plan: Plan) -> NextStep                          # Ask(question, why) | Stop
    # soft ceiling (~4) enforced in code, not by the model
```

## C10 — Recommender

```python
class Recommender:
    async def recommend(self, plan: Plan, *, limit: int = 5) -> TurnResult      # results | conflict
    async def section_options(self, plan: Plan, course_id: CourseId) -> list[SectionCombination]
# internal steps
async def expand_terms(plan: Plan) -> list[str]                                  # model
def choose_combination(combos: list[SectionCombination], prefs: Preferences) -> SectionCombination  # deterministic
async def rerank_and_explain(plan: Plan, feasible: list[CandidateEvidence]) -> list[RankedItem]     # model
def validate_ranked(items: list[RankedItem], feasible_ids: set[CourseId]) -> list[RankedItem]       # drops unknown IDs
```

## C11 — Observability

```python
def get_logger(component: str) -> StructuredLogger                             # JSON, request_id, redaction filter
def error_boundary(fn) -> fn                                                    # maps exceptions -> ToolError, logs safely
def emit_metric(name: str, value: float, dims: Mapping[str, str]) -> None
```

## C12 — Scheduled Jobs

```python
def roster_refresh_handler(event, context) -> dict                              # counts only
def plan_cleanup_handler(event, context) -> dict                                # {"deleted": n}
```

## C13 — Admin CLI

```
coursefit-admin keys issue --label <name>      # prints key once
coursefit-admin keys revoke --client <id>
coursefit-admin catalog scrape --program is-ba|business-minor --out rules/2026-27.yaml
coursefit-admin catalog check                  # schema + verified + source_url + demo set
coursefit-admin roster inspect-fields --term SP27
coursefit-admin roster snapshot --term SP27
```

## C14 — Benchmark Harness

```python
class ProfileResponder(Protocol):
    async def answer(self, profile: StudentProfile, question: str) -> str
async def run_scenario(scenario: Scenario, model_id: str, responder: ProfileResponder) -> RunRecord
async def run_benchmark(models: list[str], scenarios: list[Scenario]) -> BenchmarkReport  # fallback disabled
def check_hard_constraints(record: RunRecord) -> list[Violation]
```
