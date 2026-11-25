# CourseFit technical environment

## Runtime and deployment

| Component | Planned environment | Responsibility |
| --- | --- | --- |
| CourseFit MCP service | Python 3.12+, uv, official Python MCP SDK, Pydantic | Own planning sessions, interview policy, retrieval, deterministic checks, ranking, and explanations. |
| Cornell data and calendars | httpx, Beautiful Soup for catalog HTML, icalendar, recurring-ical-events, standard-library datetime and zoneinfo | Extract program rules, query roster offerings, expand events, and check section combinations. |
| Demo client | React 19, TypeScript 5.x in strict mode, Vite, Node.js 22, npm | Collect student inputs, let the student select a file, and present CourseFit's questions and results. |
| Local demo bridge | FastAPI and uvicorn, Python MCP client | Translate browser actions into MCP tool calls. Keep the interview and ranking in the service. |
| Hosted inference | OpenRouter API over HTTPS using httpx | Interpret preferences and course descriptions, guide questioning, and explain the ordering of verified options. |

The team will run CourseFit on a laptop. A compatible host launches the Python MCP service over stdio. For the React demo, a loopback-only FastAPI bridge holds the MCP client connection and Vite proxies browser requests to it. The client agent presents questions and collects answers. A file-picker action can call the import tool directly. Cloud infrastructure for hosting CourseFit is outside the pilot. Inference requires an internet connection and a server-side OpenRouter key. Configure the candidate model with `COURSEFIT_MODEL` and manage dependencies with uv and npm lockfiles. [MCP transports](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports) · [OpenRouter API](https://openrouter.ai/docs/api/reference/overview)

## Program rules and term offerings

- **University Catalog.** Scrape the published 2026–27 Information Science BA and Business minor pages. Represent each supported rule with program, catalog year, category, permitted course options or conditions, and source URL. Review extracted rules before exposing them to users. Unverified categories stay out of the requirement picker. [Information Science BA](https://catalog.cornell.edu/programs/information-science-ba/) · [Business minor](https://catalog.cornell.edu/programs/business-minor/)
- **Cornell Class Roster API v2.0.** Populate the term picker from `config/rosters.json`. Pass the selected roster code to `search/classes.json` to retrieve offerings for that term. Read section groups, meeting dates and times, class numbers, credits, and descriptions. Inspect the target term's fields before relying on them. Cache public roster responses by term and query, and limit live requests to one per second. [API documentation](https://classes.cornell.edu/content/FA26/api-details) · [Available rosters](https://classes.cornell.edu/api/2.0/config/rosters.json)
- **Joining the sources.** Use catalog course identifiers and verified cross-listings to find offered sections. Carry the supporting rule and roster references through to the result. An unresolved mapping or missing meeting time cannot support a verified match. Label explicitly asynchronous sections separately.

## Planning session and recommendation flow

1. **Collect inputs.** The local MCP service keeps an in-memory planning session with term, applicable catalog year, academic goals, target credits, hard constraints, preferences, and selected sections. An empty calendar is a valid starting state. Distinguish total desired credits from a per-course credit limit.
2. **Import availability.** The host supplies decoded iCalendar text to the proposed `import_calendar` tool. Client attachment interfaces may vary. The React file picker reads the selected file and sends its contents through the bridge to that tool. Expand recurrence and exceptions only over the target term, resolve time zones to America/New_York, and use the dated busy intervals for fit checks. A successful reimport replaces the previous calendar intervals. A failed import leaves them intact. Apply student-supplied scheduling constraints separately. [MCP tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)
3. **Elicit preferences and retrieve courses.** The service uses its OpenRouter model to interpret student goals, search beyond exact keywords, and choose a follow-up when the answer could improve the results. Keep explicit must-haves separate from inferred preferences. Students may request results immediately. Retrieval and weighting methods are benchmark decisions.
4. **Check and rank.** Verify the selected requirement and any hard credit or scheduling constraints in code. Test each allowed combination of lecture, discussion, and lab against calendar intervals, stated constraints, and selected courses. Rank feasible candidates using student preferences. Give the model only the course evidence and preference information needed for the task. Accept only verified course and section identifiers in the final response.
5. **Return results or explain the conflict.** The service returns structured options, supporting course and catalog evidence, and preference trade-offs. If none qualify, report the blocking constraint and await the student's decision. On selection, recheck fit, record the chosen sections and credits once, then recalculate remaining credits and available choices.

## Security and calendar privacy

Parse the calendar in memory and retain only its busy intervals for the planning session. CourseFit excludes the raw file and event details, including titles, attendees, locations, and notes, from logs and OpenRouter requests. Keep busy intervals in deterministic checks. The file picker in the React demo bypasses model conversation history. A student attaching a file to a third-party host also shares it with that host under its own handling practices.

Bind local HTTP services to loopback and restrict browser origins to the demo client. Keep `OPENROUTER_API_KEY` in the service environment. Disable request-body logging for file and tool calls, including errors. Validate file size (2 MB maximum), format, and recurrence expansion bounds before accepting an import. Isolate each planning session and discard its private data when it ends. Cache only public Cornell data. Treat retrieved text as source material and validate model output against the service's verified candidates. CourseFit changes only the student's temporary plan.

## Prohibited libraries and alternatives

| Avoid in the pilot | Reason | Use instead |
| --- | --- | --- |
| Ollama or self-hosted model runtimes | The team will compare hosted models through one provider interface. | OpenRouter with a configurable model ID |
| LangChain, CrewAI, or similar orchestration frameworks | The pilot needs direct control of follow-up decisions and fit checks. | An explicit Python tool loop and typed MCP tools |
| Browser automation for term roster extraction | Cornell publishes term offerings through an API. | httpx for roster JSON and an HTML parser for published program requirements |
| Hand-written iCalendar recurrence parsing | Exceptions and time-zone transitions affect whether a course fits. | icalendar and recurring-ical-events |
| Persistent stores for student calendars or sessions | Private planning data has a session lifetime. | In-memory state and a separate cache for public Cornell data |

## Test and evaluation environment

Use pytest for the service, respx for external HTTP fixtures, FastAPI TestClient for the demo bridge, and Vitest with React Testing Library for the client. Run Ruff and TypeScript type checking. Exercise the MCP tools with an in-process test client and a fake model. Use synthetic calendars or sanitized exports for calendar tests, including empty files with valid calendar structure, recurrence exceptions, and date-boundary cases. Verify that neither model requests nor captured logs contain distinctive fixture event details. Test that an empty calendar, an unchanged selection retried twice, and a selected lecture with a conflicting required lab yield the expected plan.

Set `COURSEFIT_MODEL` to the OpenRouter model ID for each evaluation run. The initial candidates span several price points.

- **Google Gemini 3.1 Flash Lite** (`google/gemini-3.1-flash-lite`) is the low-cost, low-latency candidate. Test whether it can ask a useful follow-up and return a grounded shortlist quickly. [OpenRouter listing](https://openrouter.ai/google/gemini-3.1-flash-lite)
- **DeepSeek V4 Flash** (`deepseek/deepseek-v4-flash`) adds a low-cost reasoning option. Compare it with Gemini on interview quality, response time, and cost per completed interview. [OpenRouter listing](https://openrouter.ai/deepseek/deepseek-v4-flash)
- **Anthropic Claude Sonnet 4.6** (`anthropic/claude-sonnet-4.6`) gives us a mid-priced comparison with Opus. Use the same student answers and feasible courses to see whether students prefer either model's ranking. [OpenRouter listing](https://openrouter.ai/anthropic/claude-sonnet-4.6)
- **Anthropic Claude Opus 4.8** (`anthropic/claude-opus-4.8`) is the higher-cost reasoning candidate. Test whether it makes better distinctions among feasible courses when a student has competing preferences. [OpenRouter listing](https://openrouter.ai/anthropic/claude-opus-4.8)

Additional OpenRouter models can enter the same benchmark. The team will choose the demo model using its measured interview quality, ranking relevance, explanation accuracy, latency, and cost per completed interview.

For interview comparisons, give the evaluator a fixed student profile for each scenario and have them answer each model’s questions using only that profile. For ranking comparisons, give every model the same transcript and feasible candidate set. Compare personalized rankings with the keyword baseline, review whether each explanation cites supporting data, and measure interview length, response time, and cost. Record model IDs with the results. The vision document defines the user-facing evaluation targets.

## Construction examples

These examples specify the intended interfaces for the team. The local bridge forwards requests to CourseFit. A small deterministic function illustrates the time-boundary convention used by the fit checker.

### Typical endpoint — discover supported requirements

```python
from fastapi import APIRouter, Request

router = APIRouter()

@router.get("/api/requirements")
async def list_requirements(catalog_year: str, request: Request):
    client = request.app.state.mcp_client
    result = await client.call_tool("list_requirements", {"catalog_year": catalog_year})
    return result.model_dump(mode="json", by_alias=True)
```

The service returns supported requirement categories with program, catalog year, and source URL. The bridge forwards the MCP result. The same tool can be called directly by another host.

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
