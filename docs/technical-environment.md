# CourseFit technical environment

## Runtime and deployment

| Component | Planned environment | Responsibility |
| --- | --- | --- |
| CourseFit MCP service | Python 3.12+, uv, official Python MCP SDK (Streamable HTTP transport), Pydantic | Own planning sessions, interview policy, retrieval, deterministic checks, ranking, and explanations. |
| Session and cache store | PostgreSQL on Amazon RDS | Persist planning sessions per client, and cache public Cornell roster data. |
| Hosting | AWS Lambda, with sane defaults and a simple architecture | Serve MCP requests over HTTPS and run scheduled jobs (daily roster refresh, idle-session cleanup). |
| Cornell data | httpx, Beautiful Soup for catalog HTML, standard-library datetime and zoneinfo | Extract program rules offline, query roster offerings, and check section combinations. |
| Hosted inference | OpenRouter API over HTTPS using httpx | Interpret preferences and course descriptions, guide questioning, and explain the ordering of verified options. |
| MCP hosts (clients) | Claude Desktop, and Claude Code or Codex | Collect student input, read the student's calendar if shared, call CourseFit tools, and present results. |

CourseFit runs as one hosted MCP service on AWS Lambda, backed by PostgreSQL on RDS. Students add it to their AI assistant as a remote MCP server and authenticate with a per-user API key that the team issues. The server maps each key to a client ID, and every planning session belongs to exactly one client. The team demos and dogfoods CourseFit through Claude Desktop and through Claude Code or Codex. There is no separate web client. High availability and disaster recovery are out of scope, so use single-AZ defaults and standard RDS automated backups.

Configuration:
- `COURSEFIT_MODEL` sets the primary model.
- `COURSEFIT_FALLBACK_MODEL` sets the model used after a failed retry.
- The OpenRouter key and database credentials live in AWS Secrets Manager.
- Manage dependencies with a uv lockfile.

[MCP transports](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports) · [OpenRouter API](https://openrouter.ai/docs/api/reference/overview)

Settle these early in design:
- **Timeouts.** API Gateway's default integration timeout is about 29–30 seconds, which is less than the 60-second shortlist target.
- **Host authentication.** Each host may handle a static API key header differently.
- **Lambda networking.** Lambda needs VPC access to RDS (with connection pooling) and outbound access to OpenRouter and Cornell.

## Program rules and term offerings

- **University Catalog.** An offline script scrapes the published 2026–27 Information Science BA and Business minor pages into a version-controlled data file. Each supported rule records program, catalog year, category, permitted course options or conditions, and source URL. A teammate reviews each extracted rule against the published page and marks it verified through a pull request. The service loads only verified rules, so unverified categories stay out of the requirement picker. For the demo, the team verifies two to three categories per program. [Information Science BA](https://catalog.cornell.edu/programs/information-science-ba/) · [Business minor](https://catalog.cornell.edu/programs/business-minor/)
- **Cornell Class Roster API v2.0.** The MVP supports the Spring 2027 roster (`SP27`) only. Pass the roster code to `search/classes.json` to retrieve offerings, and read these fields:
  - section groups
  - meeting dates and times
  - class numbers
  - credits
  - descriptions
  - prerequisite and enrollment-restriction text

  Inspect the term's fields before relying on them, and record the mapping. A snapshot job loads the term's offerings into the Postgres cache ahead of time and refreshes it daily. Live lookups fill any gaps. Limit live requests to one per second. [API documentation](https://classes.cornell.edu/content/FA26/api-details) · [Available rosters](https://classes.cornell.edu/api/2.0/config/rosters.json)
- **Joining the sources.** Use catalog course identifiers and verified cross-listings to find offered sections. Carry the supporting rule and roster references through to the result. An unresolved mapping or missing meeting time cannot support a verified match. Label explicitly asynchronous sections separately.

## Planning session and recommendation flow

1. **Collect inputs.** A planning session is stored in Postgres under its session ID and owning client ID. It holds:
   - the term and applicable catalog year
   - an optional requirement
   - total target credits and an optional per-course credit limit
   - busy blocks and hard constraints
   - preferences and stated assumptions
   - interview answers
   - selected sections

   A client can list its sessions and resume one. An `end_session` tool deletes a session immediately, and sessions idle for 30 days are deleted by a scheduled job.
2. **Receive availability.** CourseFit does not parse calendar files. The host reads the student's calendar, if shared, and sends busy time in CourseFit's format:
   - **weekly blocks:** day of week, start and end time, and optional `from` and `until` dates
   - **one-off dated blocks:** start and end datetimes

   Times are in America/New_York unless an offset is given. The tool description tells hosts to skip all-day events and events marked free, to include tentative events, and to send times only. The input schema has no fields for titles, attendees, locations, or notes. A successful submission replaces the previous busy blocks, and a failed one leaves them intact. Student-stated constraints such as "I work Fridays after noon" are parsed by CourseFit's model and applied immediately. The response shows how each was interpreted, and the student can edit or remove it.
3. **Elicit preferences and retrieve courses.** The service uses its OpenRouter model to interpret student goals and choose a follow-up when the answer could improve the results. It offers results by default after about four follow-ups, and students can request results immediately. Keep explicit must-haves separate from inferred preferences. Retrieval works in three steps: the model expands the student's interests into search terms, a lexical (BM25-style) search runs over Spring 2027 titles and descriptions, and the model reranks a top set. Retrieval sits behind an interface so semantic search can be tried later. With no requirement selected, every offering in the term is a candidate.
4. **Check and rank.** Verify the selected requirement and any hard credit or scheduling constraints in code. Test each allowed combination of lecture, discussion, and lab against busy blocks, stated constraints, and selected courses. For variable-credit courses, assume the largest value that fits the remaining credits and state the assumption. Show prerequisite and restriction text without enforcing it. State unchecked rule conditions, such as unique credit, as assumptions the student can override. Rank feasible candidates using student preferences. Give the model only the course evidence and preference information needed for the task. Accept only verified course and section identifiers in the final response.
5. **Return results or explain the conflict.** Return a five-course shortlist. Each course has one recommended section combination, a count of the alternatives (listable on request), supporting course and catalog evidence, assumptions, and preference trade-offs. If none qualify, report the blocking constraint and wait for the student's decision. On selection, recheck fit, record the chosen sections and credits once, then recalculate remaining credits and available choices.

**Model failures.** Retry a failed or timed-out OpenRouter call once, then use `COURSEFIT_FALLBACK_MODEL`. Record which model answered each call. If OpenRouter is unavailable entirely, return a clear error and leave the session unchanged.

## Security and privacy

The team has opted into the AI-DLC security baseline. Its rules are blocking constraints at every stage.

**Data minimization**
- Store only what planning needs: busy blocks, goals, constraints, preferences, assumptions, interview answers, and selections.
- Never receive or store raw calendar files or event details.
- Keep busy blocks in deterministic checks. Never send them, or the client's identity, to OpenRouter.
- A student who shares a calendar with their assistant also shares it with that host under the host's own handling practices.

**Encryption and secrets**
- Encrypt all traffic with TLS 1.2+, including connections to RDS.
- Enable RDS storage encryption with KMS.
- Keep `OPENROUTER_API_KEY` and database credentials in Secrets Manager.
- Store API keys only as hashes and support revoking them.

**Access control and input handling**
- Deny by default. Every tool that references a session must verify the caller owns it.
- Validate every tool input with Pydantic schemas that enforce types, lengths, list sizes, and formats.
- Parameterize all SQL.
- Limit request size at the gateway, and rate-limit requests per API key.
- Treat retrieved text and student text as untrusted source material, and validate model output against the service's verified candidates.

**Logging, errors, and infrastructure**
- Write structured JSON logs to CloudWatch with request IDs and at least 90-day retention.
- Never log API keys, request bodies, busy blocks, or student free text.
- Alert on repeated authentication or authorization failures.
- Return generic error messages through a global error handler.
- Grant IAM roles least privilege, and keep RDS in private subnets reachable only from the Lambda security group.

CourseFit changes only the student's own plan.

## Prohibited libraries and alternatives

| Avoid in the pilot | Reason | Use instead |
| --- | --- | --- |
| Ollama or self-hosted model runtimes | The team will compare hosted models through one provider interface. | OpenRouter with configurable primary and fallback model IDs |
| LangChain, CrewAI, or similar orchestration frameworks | The pilot needs direct control of follow-up decisions and fit checks. | An explicit Python tool loop and typed MCP tools |
| Browser automation for term roster extraction | Cornell publishes term offerings through an API. | httpx for roster JSON and an HTML parser for published program requirements |
| Storing raw calendars or event details | CourseFit only needs busy times to check fit. | Host-supplied busy blocks with a schema that has no event-detail fields |
| Persistent stores other than the session database | Private planning data needs one place with known encryption, ownership checks, and deletion. | PostgreSQL on RDS for sessions and public roster cache |

## Test and evaluation environment

Use these tools:
- **pytest** for the service
- **respx** for external HTTP fixtures
- **Hypothesis** for property-based tests of the deterministic core: interval overlap, weekly-block expansion, section-combination feasibility, credit arithmetic, and schema round-trips (partial PBT mode, with seeds logged in CI)
- **An in-process MCP test client with a fake model** to exercise the MCP tools
- **A disposable PostgreSQL** (container or test database) for storage tests
- **Ruff** for linting, plus dependency vulnerability scanning

Use synthetic busy blocks, including date-boundary and partial-term cases. Verify that neither model requests nor captured logs contain busy blocks, keys, or distinctive student text. Test that an empty schedule, an unchanged selection retried twice, and a selected lecture with a conflicting required lab yield the expected plan. Also test that one client cannot read another client's session.

A scripted benchmark harness runs the fixed scenarios against each `COURSEFIT_MODEL`, with the fallback model disabled. It records model ID, latency, cost from OpenRouter usage data, questions asked, what each answer revealed, and the final shortlist. The initial candidates span several price points.

- **Google Gemini 3.1 Flash Lite** (`google/gemini-3.1-flash-lite`) is the low-cost, low-latency candidate. Test whether it can ask a useful follow-up and return a grounded shortlist quickly. [OpenRouter listing](https://openrouter.ai/google/gemini-3.1-flash-lite)
- **DeepSeek V4 Flash** (`deepseek/deepseek-v4-flash`) adds a low-cost reasoning option. Compare it with Gemini on interview quality, response time, and cost per completed interview. [OpenRouter listing](https://openrouter.ai/deepseek/deepseek-v4-flash)
- **Anthropic Claude Sonnet 4.6** (`anthropic/claude-sonnet-4.6`) gives us a mid-priced comparison with Opus. Use the same student answers and feasible courses to see whether students prefer either model's ranking. [OpenRouter listing](https://openrouter.ai/anthropic/claude-sonnet-4.6)
- **Anthropic Claude Opus 4.8** (`anthropic/claude-opus-4.8`) is the higher-cost reasoning candidate. Test whether it makes better distinctions among feasible courses when a student has competing preferences. [OpenRouter listing](https://openrouter.ai/anthropic/claude-opus-4.8)

Additional OpenRouter models can enter the same benchmark. The team will choose the demo model using its measured interview quality, ranking relevance, explanation accuracy, latency, and cost per completed interview.

For interview comparisons, give each model a fixed student profile per scenario, and answer its follow-ups using only that profile. For ranking comparisons, give every model the same transcript and feasible candidate set. Review whether each explanation cites supporting data, and measure interview length, response time, and cost. Record model IDs with the results. The vision document defines the user-facing evaluation targets.

## Construction examples

These examples show the intended interfaces. A small deterministic function shows the time-boundary convention the fit checker uses.

### Typical MCP tool — discover supported requirements

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("coursefit", stateless_http=True)

@mcp.tool()
async def list_requirements(catalog_year: str) -> list[RequirementSummary]:
    """List verified requirement categories for a catalog year."""
    return await rules.list_verified(catalog_year)
```

The service returns supported requirement categories with program, catalog year, and source URL. Authentication middleware resolves the caller's API key to a client ID before any tool runs. Any MCP host can call the same tool.

### Typical function — compare two dated meeting intervals

```python
from datetime import datetime

def overlaps(
    first_start: datetime, first_end: datetime,
    second_start: datetime, second_end: datetime,
) -> bool:
    """Compare validated, timezone-aware intervals with exclusive end times."""
    return first_start < second_end and second_start < first_end
```

### Typical test — a lab starts when a work shift ends

```python
from datetime import datetime, timezone

from coursefit.fit import overlaps

def test_back_to_back_meetings_fit_but_a_one_minute_overlap_conflicts():
    def at(hour: int, minute: int = 0) -> datetime:
        return datetime(2027, 2, 1, hour, minute, tzinfo=timezone.utc)

    assert not overlaps(at(14), at(15), at(15), at(16))
    assert overlaps(at(14), at(15, 1), at(15), at(16))
```
