# User Stories Assessment

## Request Analysis
- **Original Request**: Build CourseFit, a hosted MCP service that interviews Cornell students and recommends Spring 2027 courses that fit their schedule, credits, and requirements, with explanations.
- **User Impact**: Direct. Students use it through their own AI assistant.
- **Complexity Level**: Complex
- **Stakeholders**:
  - Students
  - The five-person INFO 5940 team, as builders and as operators who issue API keys, verify catalog rules, and run benchmarks
  - Course staff who evaluate the demo

## Assessment Criteria Met
- [x] High Priority: New user-facing functionality (interview, shortlist, selection, saved plans)
- [x] High Priority: Customer-facing API. The MCP tool surface is consumed by third-party hosts (Claude Desktop, Claude Code, Codex)
- [x] High Priority: Multiple personas. Students with different goals, plus team operators
- [x] High Priority: Complex business logic with many scenarios (section feasibility, credits, assumptions and overrides, no-feasible-result conflicts)
- [x] High Priority: Cross-team collaboration. Five teammates will split units of work and review each other's pull requests
- [x] Medium Priority: Security changes that affect users (API keys, session ownership, data deletion)
- [x] Benefits: Acceptance criteria double as the ten demo scenarios and as test cases

## Decision
**Execute User Stories**: Yes

**Reasoning**: Every high-priority indicator applies. The team agreed to review Inception together and then divide implementation. Stories with testable acceptance criteria give each owner a clear contract. They also make the correctness evaluation (ten scenarios, every hard constraint passes) concrete.

## Expected Outcomes
- A shared, testable definition of each student-facing behavior before the team splits the work
- Acceptance criteria that map directly to the demo scenarios and pytest cases
- Operator stories that keep easy-to-forget work in scope (key issuance, rule verification, roster refresh, cleanup, benchmarks)
- Persona-to-story mapping that informs Units Generation
