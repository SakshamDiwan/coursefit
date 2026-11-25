# CourseFit Story-to-Unit Map

**Primary** is the unit that owns the story's acceptance criteria. **Contributing** units supply parts of it. Every story has exactly one primary unit. U0 contributes the models to almost every story and is listed only where its models are central.

| Story | Title | Primary | Contributing |
|---|---|---|---|
| US-1.1 | Start a plan for Spring 2027 | U5 | U0, U1, U6 |
| US-1.2 | Browse supported requirements | U2 | U6 |
| US-1.3 | Choose a requirement, or none | U5 | U2, U6 |
| US-2.1 | Send busy times from my calendar | U5 | U0 (schema, no event-detail fields), U6 (host guidance in tool description) |
| US-2.2 | Start with an empty schedule | U3 | U5 |
| US-2.3 | Replace my busy times safely | U5 | U0, U1 |
| US-2.4 | Add a constraint in plain language | U4 | U3, U5 |
| US-2.5 | Correct or remove a constraint | U4 | U5 |
| US-3.1 | Get a follow-up that matters | U4 | U5 |
| US-3.2 | Results by default after about four questions | U4 | U5 |
| US-3.3 | Ask for results any time | U5 | U4, U6 |
| US-4.1 | See a feasible five-course shortlist | U3 | U2 (mapping, async labels), U4 (`validate_ranked`), U5 |
| US-4.2 | Understand why each course was recommended | U4 | U2 (descriptions) |
| US-4.3 | See alternative section sets | U3 | U5, U6 |
| US-4.4 | See prerequisites without being blocked | U2 | U4 |
| US-4.5 | See and override CourseFit's assumptions | U2 | U4 (interpret override), U5 |
| US-4.6 | Learn why nothing fits | U3 | U5 |
| US-5.1 | Select a course and update my plan | U5 | U3, U6 |
| US-5.2 | Selecting twice doesn't double-count | U3 | U1 (versioning), U5 |
| US-5.3 | Choose credits for a variable-credit course | U3 | U4 (states the assumption), U5 |
| US-5.4 | Remove a selection | U3 | U5, U6 |
| US-6.1 | Connect with my API key | U1 | U6 (Function URL, host verification) |
| US-6.2 | Resume a saved plan | U1 | U5, U6 |
| US-6.3 | My plans are private to me | U1 | U6 (an ownership test per tool) |
| US-6.4 | Delete my plan | U1 | U5, U6 |
| US-6.5 | Forgotten plans expire | U1 | U6 (scheduled job) |
| US-7.1 | Issue and revoke API keys | U7 | U1 |
| US-7.2 | Scrape catalog rules into a reviewable file | U2 | U7 |
| US-7.3 | Verify rules before students see them | U2 | U7 (`catalog check`) |
| US-7.4 | Load and refresh Spring 2027 roster data | U2 | U6 (scheduled job), U7 (snapshot command) |
| US-7.5 | Configure primary and fallback models | U4 | U6 (environment and secrets) |
| US-7.6 | Benchmark candidate models | U8 | U4, U5 |

## Primary Stories per Unit

| Unit | Primary stories | Count |
|---|---|---|
| U0 Contracts | (foundation for all) | 0 |
| U1 Platform | US-6.1, 6.2, 6.3, 6.4, 6.5 | 5 |
| U2 Cornell | US-1.2, 4.4, 4.5, 7.2, 7.3, 7.4 | 6 |
| U3 Schedule | US-2.2, 4.1, 4.3, 4.6, 5.2, 5.3, 5.4 | 7 |
| U4 Recommend | US-2.4, 2.5, 3.1, 3.2, 4.2, 7.5 | 6 |
| U5 Planning | US-1.1, 1.3, 2.1, 2.3, 3.3, 5.1 | 6 |
| U6 MCP Server | (wires and deploys all; ownership test for every tool; host verification) | 0 |
| U7 Admin CLI | US-7.1 | 1 |
| U8 Benchmark | US-7.6 | 1 |
| **Total** | | **32** |

U0 and U6 have no primary stories but carry heavy cross-cutting work: U0 the shared contracts, U6 the tool layer, infrastructure, deployment, and host verification. The suggested owner grouping in unit-of-work.md balances this.

## Demo Scenarios to Units

| Scenario | Units exercised (primary first) |
|---|---|
| S1 Busy schedule plus plain-language constraint | U3, U4, U5 |
| S2 Empty schedule, no requirement | U3, U5, U4 |
| S3 Required-lab conflict | U3, U2 |
| S4 Unsupported requirement | U5, U2 |
| S5 Second selection changes what fits | U3, U5 |
| S6 Nothing satisfies the must-haves | U3, U5 |
| S7 Misread constraint corrected | U4, U5 |
| S8 Double-count assumption overridden | U2, U4, U5 |
| S9 Variable-credit course at a lower value | U3, U5 |
| S10 Plan resumed later | U1, U5, U6 |

U8 runs all ten for each candidate model. U5's service tests run all ten against the fake model.

## Validation
- [x] All 32 stories are assigned to exactly one primary unit.
- [x] Every component (C1–C14) and service (S1–S6) has exactly one owning unit: C2, C3, C11 in U1; C4, C5 in U2; C6 in U3; C7–C10 in U4; S1–S4 in U5; C1, C12, and infra in U6; C13 in U7; C14 and S6 in U8. S5 data operations are split: the refresh logic is in U2, the cleanup logic in U1, and they're invoked by U6 jobs and U7 commands.
- [x] There are no circular dependencies (see unit-of-work-dependency.md).
- [x] All demo scenarios map to units.
