# CourseFit

An LLM agent that finds Cornell courses that fit your actual week. INFO 5940 team project.

## Team

| Name | NetID |
| --- | --- |
| Saksham Diwan | sd966 |
| Uday Tyagi | ut25 |
| Aditya Chandak | TODO |
| Sai Vamsi | TODO |
| Aryan Patel | TODO |

**Coding agent:** Claude Code. Teammates who use a different agent will add and commit that agent's AI-DLC rule files.
**AI-DLC version:** v1.0.1

## Project overview

### 1. Problem and why an LLM

Finding a course at Cornell is a constraint problem wearing a search box. A student who needs "one more 3-credit course that counts for a humanities distribution, fits around my job and my other classes, nothing before 10am or on Fridays, ideally something about cities or food" has to do it by hand: open their calendar, write down the gaps, browse Class Roster subject by subject, open each class, and check every lecture, discussion and lab time against those gaps. The roster's filters can't express "fits around my week" or "something about cities," and most students give up after a few subjects and settle for whatever they already knew about. CourseFit is for Cornell undergraduate and master's students during pre-enrollment and add/drop, especially those hunting for a filler course or a distribution requirement.

An LLM is the right tool because the hard part of the request is language, not lookup. Students describe what they want in loose, mixed terms (hard constraints, soft preferences, vague interests) that no fixed filter form captures. The LLM turns that into concrete searches, decides which roster queries to run and in what order, reads course titles and descriptions to judge topical fit, and explains the trade-offs between options in plain English. Everything that has a single correct answer (parsing the calendar, computing free time, checking conflicts) is done by ordinary code, so the LLM never does arithmetic on times.

### 2. Planned LLM integration

CourseFit is an agent, not a single prompt. The LLM runs a tool-calling loop over a small set of tools that we write: `search_classes` (queries the public Cornell Class Roster API by subject, keyword, level and distribution requirement), `get_class_details` (meeting times, sections, credits, prerequisites), `get_free_slots` (the student's weekly free time, computed from an uploaded `.ics` calendar plus any courses or constraints they state), and `check_fit` (whether every required meeting of a class falls inside those free slots). The agent decides which tools to call, filters and ranks the results, and returns a short list of options with reasons and the class numbers needed to enroll in Student Center.

The same tools are exposed two ways: a local web app (React + TypeScript UI over a small Python API) with its own agent loop, which shows recommendations as class cards on a weekly calendar grid next to the student's busy time, and an MCP server so the tools can be used directly from Claude or ChatGPT once Cornell rolls out free access to students in early 2027. The model is a free, open, tool-calling model run locally with **Ollama** (planned: `qwen3:8b`, with a smaller fallback for slower laptops), called through Ollama's Python client.

### 3. Scope and feasibility

By the final deadline (December 16), CourseFit will let a student upload their calendar, optionally list current courses and add constraints in plain words, then ask for courses in natural language and get back ranked, conflict-free options for the next term, with explanations and class numbers. It will run locally through the web app and as an MCP server. It will not enroll students, sign in with a NetID, audit degree requirements, or use professor ratings or reviews; those are listed as out of scope in our vision document. The core is five separable pieces (roster client, calendar and free-time engine, fit checking, agent loop, local API and MCP server) plus the web UI split across our five team members, and every piece except the agent loop is deterministic and unit-testable, which keeps the project achievable in about three weeks of part-time work.

### 4. Collaboration plan

- **Meetings:** once a week, in person. AI-DLC gates (approvals of requirements, stories, design and units) happen in these meetings or in extra short sessions with everyone present, so the whole team makes those decisions together.
- **Between meetings:** our WhatsApp group for questions, blockers and quick decisions. Code goes through GitHub: each person works on a branch for their unit and opens a pull request, which at least one teammate reviews before merging.
- **Inception:** run once, together. Everyone answers the agent's questions and reviews each document before approval.
- **Construction split (initial; we will re-split if anyone needs to):**

| Unit | Owner |
| --- | --- |
| Cornell Class Roster API client + Web UI | Saksham Diwan |
| Agent loop, tools and prompts | Uday Tyagi |
| Calendar (`.ics`) import and free-time engine | Aditya Chandak |
| Fit checking and result filtering | Sai Vamsi |
| Local API (FastAPI) and MCP server | Aryan Patel |

## Repository layout

```text
CLAUDE.md                 AI-DLC v1.0.1 core workflow (Claude Code)
.aidlc-rule-details/      AI-DLC v1.0.1 rule details
docs/vision.md            Vision document (AI-DLC input)
docs/technical-environment.md  Technical environment document (AI-DLC input)
web/                      React + TypeScript UI (added during Construction)
aidlc-docs/               Generated by AI-DLC (Inception artifacts, audit.md, aidlc-state.md)
```

Application code will live at the repository root during Construction, never inside `aidlc-docs/`.
