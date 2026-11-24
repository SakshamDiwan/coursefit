# Vision: CourseFit

## Executive Summary

CourseFit is a course-planning agent for Cornell students. A student uploads their calendar, says in plain English what kind of course they need, and CourseFit finds courses in the live Cornell Class Roster that fit both their free time and their requirements, then explains the best options and gives the class numbers to enroll with. It runs locally as a web app and as an MCP server, so the same tools work inside Claude or ChatGPT once Cornell gives students free access to those tools. It saves students the hours they spend cross-checking roster times against their week by hand.

## Problem Statement

Cornell students build schedules with Class Roster, Scheduler and their own calendar open side by side. Class Roster can filter by subject, level and distribution requirement, but it cannot answer the questions students actually have: "what fits in my Tuesday/Thursday afternoon gap?", "which humanities courses avoid Fridays and mornings?", "is there anything about urban planning I could take without dropping my job shifts?". Answering those means opening dozens of class pages and checking every lecture, discussion and lab time against the student's commitments, most of which (jobs, practices, club meetings, research) live only in their personal calendar.

The result is that students search a few familiar subjects, miss courses that would have fit, and fill requirements with whatever they already knew about. The problem peaks during pre-enrollment and add/drop, when students are making these decisions under time pressure.

## Target Users

| User type | Who they are | What they need from CourseFit |
| --- | --- | --- |
| Filler-course seeker | Undergrad or master's student with a mostly fixed schedule who needs one or two more courses to reach a credit count | Courses that fit existing gaps, ranked by how well they match loose interests |
| Requirement hunter | Undergrad who needs a specific distribution requirement (e.g. a humanities or social science code) | Every course carrying that requirement next term that does not conflict with their week |
| Busy student with outside commitments | Student with a job, sport, research or club schedule in their calendar | Options that respect non-class commitments, which Class Roster knows nothing about |
| AI-tool user (post-rollout) | Any Cornell student using Claude or ChatGPT through Cornell's 2027 rollout | The same search and fit tools available directly inside the assistant they already use, via MCP |

## Success Metrics

| Metric | Target | How we measure it |
| --- | --- | --- |
| Conflict-free results | 100% of recommended classes have every required meeting inside the student's free slots | Automated tests on the fit checker plus a scripted set of 10 demo scenarios |
| Constraint adherence | ≥ 90% of recommendations satisfy every hard constraint the student stated (distribution, credits, days, times) | Manual review of the 10 demo scenarios |
| Calendar import | Correctly extracts weekly busy blocks from `.ics` exports from Google Calendar, Apple Calendar and Outlook | Unit tests with one sample export from each |
| Time to answer | A full answer in under 60 seconds on a team member's laptop | Timed runs during the demo scenarios |
| Usefulness | At least 3 of 5 classmates who try it find a course they would consider that they had not seen before | Informal user test before the final demo |

## Full Scope Vision

At maturity CourseFit could grow into a full planning assistant:

- **Search and discovery:** natural-language search over every roster, similarity search ("courses like INFO 5940"), cross-listed course handling.
- **Schedule intelligence:** multi-course schedule building ("give me a full 15-credit schedule"), commute time between buildings, preferred workload balance across days.
- **Requirements:** degree-audit integration, major/minor requirement tracking, multi-semester planning.
- **Course quality signals:** student reviews, grade medians, workload estimates.
- **Enrollment help:** waitlist and seat-availability alerts, enrollment-day reminders, direct export to Scheduler.
- **Distribution:** hosted web app with NetID sign-in, and a published skill/connector in Cornell's AI tools.

## MVP Scope -- Features IN

| Feature | Description | Rationale |
| --- | --- | --- |
| Calendar upload | Student uploads an `.ics` file; recurring weekly events become busy blocks for a typical week (Mon–Fri, configurable day window) | Primary input. Captures classes and outside commitments in one step |
| Optional course list | Student can list course codes they are taking; CourseFit looks up exact meeting times in Class Roster and adds them as busy blocks | Covers students whose classes are not in their calendar, or whose calendar entries are imprecise |
| Plain-language constraints | Student states constraints like "no Fridays", "nothing before 10am", "3–4 credits", "needs a humanities distribution" | Hard constraints most students have but Class Roster cannot express together |
| Free-time engine | Deterministic code merges calendar, course list and constraints into weekly free slots, with a short buffer between blocks | Core of the product; must be exact, so it is not done by the LLM |
| Live roster search | Agent tool that queries the public Cornell Class Roster API by subject, keyword, level and distribution requirement for the target term | Real, current data |
| Fit checking | Deterministic check that every required meeting of a class (lecture plus a chosen discussion/lab if one fits) falls inside free slots | Guarantees conflict-free recommendations |
| Agent loop | LLM decides which searches to run, interprets interests, filters with the fit tool and ranks results | The reason this needs an LLM: fuzzy requests, multi-step search |
| Ranked recommendations | Short list (default 5) of courses with meeting times, credits, distribution codes, why each fits, and class numbers to enroll with | The deliverable a student acts on |
| Local web app | React + TypeScript UI: chat, calendar upload, a weekly grid showing busy time next to each recommended class's meetings, and class cards with times, credits, distribution codes and class numbers | How students and graders use it; seeing the fit on a calendar is faster to trust than reading times |
| MCP server | The same tools exposed as an MCP server usable from Claude, ChatGPT or any MCP client | Ties the product to Cornell's AI tool rollout |
| Local LLM via Ollama | The web app's agent uses a free open model through Ollama | Team has no paid API key; runs offline at no cost |

## MVP Scope -- Features OUT

| Feature | Reason excluded | Target phase |
| --- | --- | --- |
| Enrolling in classes | Cornell provides no enrollment API; enrollment requires the student's own Student Center session | Not planned |
| NetID sign-in / Cornell accounts | Not needed for public roster data; adds security work | Phase 3 |
| Professor ratings and course reviews | No official API for RateMyProfessors or CU Reviews; deliberately dropped to keep the MVP simple and reliable | Phase 2 |
| Degree audit and major requirements | No public data source; would require per-major rules | Phase 3 |
| Grade medians and workload estimates | No reliable public data source | Phase 2 |
| Full-schedule building (multiple new courses at once) | Combinatorial search adds complexity beyond one-course-at-a-time recommendations | Phase 2 |
| Seat availability and waitlist alerts | Needs polling and notifications | Phase 2 |
| Walking time between buildings | Room locations require NetID sign-in on Class Roster | Phase 2 |
| Cloud hosting | Course requires local runs only | Phase 3 |
| Mobile app | Not needed for the course demo | Not planned |

## Risks and Open Questions

### Risks

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Local model is slow or unreliable at tool calling | Poor answers or long waits during the demo | Keep the tool set small with clear schemas; test candidate models early; keep a smaller fallback model |
| Class Roster API response format changed in Fall 2025 and some catalog fields are now empty | Missing credits, descriptions or distribution data | Inspect real responses for the target term before writing the client; read fields from enrollment groups where catalog fields are empty |
| Class Roster API rate limit (max 1 request per second) | Slow searches across many subjects | Cache responses on disk for the session; prefer distribution-requirement searches, which do not need a subject |
| Messy calendar exports (recurrence exceptions, time zones, all-day events) | Wrong free slots | Use established iCalendar libraries; convert all times to America/New_York; ignore all-day events; test against real exports |
| Agent recommends classes that do not actually fit | Loss of trust | Recommendations only come from classes that passed the deterministic fit check |

### Open Questions

1. Which term does CourseFit plan for by default: always the next upcoming roster (Spring 2027 for the demo), or a term the student picks?
2. Which week of the uploaded calendar represents the student's "typical week"? Use only recurring events, or a specific week the student chooses?
3. How long should the buffer between a calendar event and a class be (e.g. 10 or 15 minutes), and should students be able to change it?
4. If a class has several discussion or lab sections, is it a fit when at least one section fits? (Proposed: yes, and we report which section.)
5. How should courses typed by the student that also appear in their calendar be de-duplicated when we show their current schedule back to them?
6. Do we include online or asynchronous classes with no meeting times, and how do we label them?
7. How should the agent rank fitting results when the student gives no interests: by distribution match, by time-of-day preference, or by something else?
8. Which Ollama model gives acceptable tool-calling quality and speed on the team's laptops?
