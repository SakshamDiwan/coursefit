# Technical Environment: CourseFit

## Languages and Package Managers

| Part | Language | Package manager |
| --- | --- | --- |
| Backend (roster client, calendar, free slots, fit, agent, local API, MCP server) | **Python 3.12+** | **uv** (`uv add`, `uv run`); `pyproject.toml` + `uv.lock` committed |
| Web UI (`web/`) | **TypeScript 5.x** (strict mode) on **Node.js 22 LTS** | **npm**; `package.json` + `package-lock.json` committed |

No other languages. No JavaScript files in `web/src/` (TypeScript only).

## Frameworks and Libraries

### Backend (Python)

| Purpose | Library | Notes |
| --- | --- | --- |
| Local API for the web UI | **FastAPI** + **uvicorn** | JSON endpoints under `/api/`; Pydantic models for request bodies |
| LLM client | **ollama** (official Python client) | Tool calling via `ollama.chat(..., tools=[...])` |
| MCP server | **mcp** (official Python SDK, `FastMCP`) | stdio transport for Claude Desktop / Claude Code / other MCP clients |
| HTTP | **httpx** | Calls to the Cornell Class Roster API |
| Calendar parsing | **icalendar** + **recurring-ical-events** | Parse `.ics` and expand recurring events correctly |
| Times | Python standard library `datetime` + `zoneinfo` | All times converted to `America/New_York` |

### Web UI (TypeScript)

| Purpose | Library | Notes |
| --- | --- | --- |
| UI framework | **React 19** | Function components and hooks; state in `useState`/`useReducer` |
| Build/dev server | **Vite** | Dev server proxies `/api` to the backend at `http://127.0.0.1:8000` |
| Styling | **Tailwind CSS** | No other CSS framework |
| HTTP | Browser `fetch` | Wrapped in a few typed functions in `web/src/api.ts` |
| Calendar view | Hand-built weekly grid (CSS grid) | Shows busy blocks and the candidate class's meetings side by side |

Keep code simple: plain functions and dicts. No class hierarchies, plugin systems or dependency-injection layers unless a requirement needs them.

## LLM

- **Provider:** Ollama, running locally on each team member's laptop. No hosted LLM API.
- **Planned model:** `qwen3:8b` (supports tool calling). Fallback for slower laptops: `qwen3:4b`. Final choice is an open question to settle in the first week of Construction.
- The model name and Ollama host are read from environment variables (`COURSEFIT_MODEL`, `OLLAMA_HOST`) with the defaults above.
- When CourseFit runs as an MCP server, the host assistant (Claude, ChatGPT, etc.) is the LLM; CourseFit only provides tools.

## External Data

- **Cornell Class Roster API v2.0:** `https://classes.cornell.edu/api/2.0/`. Public, read-only, GET only, JSON. No API key.
  - Methods used: `config/rosters.json`, `config/subjects.json`, `search/classes.json`.
  - `search/classes` requires `roster` and either `subject` or a filter such as `distrReqs[]`. Repeat parameters for multiple values (`distrReqs[]=HST-AS&distrReqs[]=ALC-AS`).
  - The API has no day or time filters, so time fitting is done locally.
  - Rosters from Fall 2025 onward use a new data source; several old catalog fields are empty. Inspect real responses for the target term (`SP27`) before relying on any field.
  - **Rate limit: at most 1 request per second.** Enforced in the client.
- No other external data sources (no professor ratings, no review sites).

## Cloud Provider and Deployment Model

- **No cloud provider. The product runs locally only.**
- Local API: `uv run uvicorn coursefit.api:app --host 127.0.0.1 --port 8000`, calling a model served by Ollama at `http://localhost:11434`.
- Web UI: `cd web && npm run dev`, opened at `http://localhost:5173`. Vite proxies `/api/*` to the local API, so no CORS setup is needed.
- MCP server: `uv run python mcp_server.py`, registered in an MCP client (e.g. Claude Desktop or Claude Code) as a local stdio server.
- Roster responses are cached as JSON files in a local `.cache/` directory (git-ignored) to respect the rate limit and speed up demos.

## Architecture and Project Structure

```text
coursefit/
  roster.py           # Class Roster API client + disk cache + rate limit
  calendar_import.py  # .ics -> weekly busy blocks
  slots.py            # busy blocks + constraints -> free slots
  fit.py              # does a class's meeting pattern fit the free slots?
  tools.py            # tool functions + JSON schemas shared by agent and MCP server
  agent.py            # Ollama tool-calling loop and system prompt
  api.py              # FastAPI app: /api/calendar, /api/chat
mcp_server.py         # FastMCP server exposing tools.py
tests/                # Python tests
web/                  # React + TypeScript UI (Vite)
  src/
    api.ts            # typed fetch wrappers for /api/*
    types.ts          # shared TypeScript types (Message, Busy, ClassOption)
    components/       # Chat, CalendarUpload, WeekGrid, ClassCard
```

- `tools.py` is the single source of truth for tools. The agent (used by the web UI through `api.py`) and the MCP server both import from it; no tool logic is duplicated.
- The local API is stateless. The UI uploads the `.ics` once to `/api/calendar`, keeps the returned busy blocks in React state, and sends them with every `/api/chat` request.
- The API returns the agent's reply plus the structured list of recommended classes (with meeting times), so the UI can render class cards and the weekly grid without parsing model text.
- Times inside the system are represented as minutes since midnight and Cornell day codes (`M T W R F S Su`).
- The LLM never computes times or conflicts. It calls `get_free_slots` and `check_fit`, which are deterministic.

## Testing

| Type | Tool | Rule |
| --- | --- | --- |
| Unit tests | **pytest** | Every function in `roster.py`, `calendar_import.py`, `slots.py`, `fit.py` and `tools.py` |
| HTTP mocking | **respx** | Tests never call the live Class Roster API |
| Calendar fixtures | Real `.ics` exports from Google Calendar, Apple Calendar and Outlook, stored in `tests/fixtures/` with personal details removed | |
| Agent tests | pytest with a fake `chat` function returning scripted tool calls | Tests never call a real model |
| API tests | FastAPI `TestClient` | Agent is replaced with a stub; tests check status codes and response shape |
| UI tests | **Vitest** + **React Testing Library** | Tests for `api.ts` (with mocked `fetch`) and for the WeekGrid and ClassCard components |
| Type checking | `tsc --noEmit` (strict) | Must pass with zero errors |
| Lint/format | **ruff** (Python), **ESLint** + **Prettier** (TypeScript) | `uv run ruff check .`, `npm run lint` |
| Coverage | pytest-cov | ≥ 80% line coverage on the deterministic modules (`roster`, `calendar_import`, `slots`, `fit`, `tools`) |

A short manual script of 10 demo scenarios (stored in `tests/demo-scenarios.md`) is run before the final demo.

## Do NOT Use

| Prohibited | Reason | Use Instead |
| --- | --- | --- |
| LangChain, LlamaIndex, CrewAI, AutoGen | Hide a simple tool-calling loop behind heavy abstractions; harder to debug with a local model | The `ollama` client with a hand-written loop in `agent.py` |
| OpenAI, Anthropic, Gemini paid APIs | Team has no API key; must run free and offline | Ollama with a local open model |
| requests, urllib | Inconsistent with the rest of the codebase; no timeouts by default | httpx |
| Selenium, Playwright, BeautifulSoup, any scraping | Class Roster has an official API; scraping is brittle and may violate site terms | Class Roster API v2.0 |
| RateMyProfessors or CU Reviews clients | Ratings and reviews are out of scope for the MVP | Nothing (out of scope) |
| Hand-written RRULE parsing, `dateutil.rrule` directly | Recurrence exceptions and time zones are easy to get wrong | recurring-ical-events |
| Flask, Django | Project uses FastAPI for its small local API | FastAPI |
| Streamlit, Gradio | Too limited for the calendar view and class cards we want | React + TypeScript |
| Next.js, Remix, any server-side rendering | Local single-page app; SSR adds a second server for no benefit | Vite + React |
| Redux, Zustand, MobX | App state is small (messages, busy blocks, results) | React `useState`/`useReducer` |
| axios | `fetch` is built in and enough | `fetch` in `web/src/api.ts` |
| Plain JavaScript files, `any` types | Defeats TypeScript's checks | TypeScript strict mode, explicit types in `types.ts` |
| PostgreSQL, SQLite, Redis, any database | No persistent user data in the MVP | JSON file cache in `.cache/` |
| pip, poetry, pipenv, conda | Project uses uv only | uv |
| black, flake8, isort | Replaced by ruff | ruff |
| Using the LLM for time arithmetic or conflict checks | LLMs make arithmetic mistakes; results must be exact | `slots.py` and `fit.py` |

## Security Basics

- **Reference framework:** OWASP Top 10 for LLM Applications. Controls below map to prompt injection (LLM01), sensitive information disclosure (LLM02), improper output handling (LLM05) and excessive agency (LLM06).
- **Authentication:** none. Single user, local only; the API binds to `127.0.0.1` and the UI reaches it through the Vite proxy. No CORS origins are opened.
- **Secrets:** none required (Ollama is local, the roster API has no key). `.env` files are git-ignored anyway, and no secrets are ever committed.
- **Calendar privacy (LLM02):** uploaded `.ics` files are parsed in memory and never written to disk, logged or sent anywhere. Only computed busy/free time blocks are passed to the model, not event titles, locations or attendees.
- **Input validation:** `.ics` uploads capped at 2 MB and rejected if they fail to parse; tool arguments validated before use (roster code matches `^(FA|SP|SU|WI)\d{2}$`, subject matches `^[A-Z]{2,6}$`, times within 0–1439 minutes).
- **Prompt injection (LLM01):** course titles and descriptions from the roster are treated as data. The system prompt tells the model to ignore instructions inside tool results.
- **Output handling (LLM05):** every recommended class must come from a `check_fit` result in the same conversation; the app drops any class the model names that did not pass the check.
- **Limited agency (LLM06):** all tools are read-only. CourseFit cannot enroll, write to calendars or send anything on the student's behalf.
- **Network:** outbound requests only to `classes.cornell.edu` and the local Ollama host, with a 10-second timeout.

## Example Code

### A typical endpoint (local API)

```python
from fastapi import FastAPI, HTTPException, UploadFile
from pydantic import BaseModel

from coursefit import agent, calendar_import

app = FastAPI()

MAX_ICS_BYTES = 2_000_000


class ChatRequest(BaseModel):
    messages: list[dict]
    busy: dict[str, list[tuple[int, int]]] = {}


@app.post("/api/calendar")
async def upload_calendar(file: UploadFile) -> dict:
    data = await file.read()
    if len(data) > MAX_ICS_BYTES:
        raise HTTPException(413, "Calendar file must be under 2 MB.")
    try:
        return {"busy": calendar_import.busy_blocks(data)}
    except ValueError:
        raise HTTPException(400, "Could not read that .ics file.")


@app.post("/api/chat")
def chat(req: ChatRequest) -> dict:
    # Returns {"reply": str, "classes": [ ...recommended classes with meetings... ]}
    return agent.run(req.messages, busy=req.busy)
```

### A typical MCP tool

The same tools are exposed over MCP for Claude, ChatGPT and other MCP clients:

```python
from mcp.server.fastmcp import FastMCP

from coursefit import roster

mcp = FastMCP("coursefit")


@mcp.tool()
def search_classes(term: str, subject: str = "", keyword: str = "", distribution: str = "") -> list[dict]:
    """Search the Cornell Class Roster for a term (e.g. SP27).

    Give a subject (e.g. INFO), a distribution code (e.g. HST-AS), or both.
    Returns one summary dict per class: subject, number, title, credits,
    distribution codes and meeting times.
    """
    if not subject and not distribution:
        return [{"error": "Provide a subject or a distribution code."}]
    return roster.search(term, subject=subject, keyword=keyword, distribution=distribution)


if __name__ == "__main__":
    mcp.run()
```

### A typical UI function (TypeScript)

```typescript
import type { Busy, ChatReply, Message } from "./types";

export async function sendChat(messages: Message[], busy: Busy): Promise<ChatReply> {
  const res = await fetch("/api/chat", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ messages, busy }),
  });
  if (!res.ok) {
    throw new Error(`Chat request failed (${res.status})`);
  }
  return res.json();
}
```

### A typical backend function

```python
DAYS = ["M", "T", "W", "R", "F"]


def free_slots(busy, day_start=8 * 60, day_end=22 * 60, buffer=10):
    """Turn busy blocks into free slots for each weekday.

    busy: {"M": [(600, 650), ...], ...} in minutes since midnight.
    Each busy block is padded by `buffer` minutes on both sides.
    Returns {"M": [(start, end), ...], ...}.
    """
    free = {}
    for day in DAYS:
        blocks = sorted((max(day_start, s - buffer), min(day_end, e + buffer)) for s, e in busy.get(day, []))
        cursor, slots = day_start, []
        for s, e in blocks:
            if s >= e:
                continue
            if s > cursor:
                slots.append((cursor, s))
            cursor = max(cursor, e)
        if cursor < day_end:
            slots.append((cursor, day_end))
        free[day] = slots
    return free
```

### A typical test

```python
from coursefit.slots import free_slots


def test_gaps_around_two_classes():
    busy = {"M": [(600, 650), (780, 830)]}  # 10:00-10:50, 1:00-1:50
    assert free_slots(busy, buffer=10)["M"] == [(480, 590), (660, 770), (840, 1320)]


def test_empty_day_is_fully_free():
    assert free_slots({})["T"] == [(480, 1320)]


def test_event_outside_window_is_ignored():
    busy = {"W": [(1380, 1410)]}  # 11:00pm
    assert free_slots(busy)["W"] == [(480, 1320)]
```
