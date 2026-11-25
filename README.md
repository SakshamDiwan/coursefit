# CourseFit

Personalized Cornell course recommendations through the Model Context Protocol (MCP). An INFO 5940 AI Innovation Lab team project.

## Team

| Name | NetID |
| --- | --- |
| Saksham Diwan | sd966 |
| Uday Tyagi | ut25 |
| Aditya Mahesh Chandak | ac3354 |
| Sai Vamsi Allada | sa2492 |
| Aryan Patel | ap2568 |

**Development tools** — Claude Code and Codex with AI-DLC v1.0.1

## Problem and rationale

A Cornell student looking for a three-credit elective might know it needs to count toward their minor, fit around work, and cover something related to AI. Do they want to build models, study how people use them, or explore how public institutions regulate them? Searching the Cornell Class Roster assumes the student already knows which terms and departments to try. CourseFit uses a **short, adaptive interview** to connect a student's interests, learning objectives, and career goals to courses they might otherwise overlook. An LLM can interpret an answer like *“I want to understand how a city decides whether to use facial recognition”* and find relevant descriptions across departments, even when the course title uses different language.

## Planned LLM integration

The primary deliverable is an **MCP service** that owns the interview and recommendation process. CourseFit decides when a preference is clear enough to search and when a follow-up could change the shortlist. It retrieves candidates, weighs the student's preferences, and explains why one course ranks above another using evidence from the course descriptions. The **host assistant** translates a student's request into structured input, displays the next question CourseFit returns, and presents the results. A simple React demo app can run a client agent that calls the same service.

CourseFit will call hosted language models through **OpenRouter**. We will benchmark candidates on the same student scenarios before selecting a model.

- **Interview quality** — Does a follow-up uncover a preference that changes the shortlist?
- **Recommendation quality** — Do students prefer the results to a keyword-search baseline?
- **Explanation accuracy** — Does the course data support each reason given?
- **Speed and cost** — How quickly does the model respond, and what does a completed interview cost?

## MVP scope and feasibility

For the **December 16, 2026** MVP, we will scrape published requirements for a limited set of Cornell majors and minors and match them to courses offered in the target term.

1. **Student inputs** — The student selects from the supported majors and minors, identifies the requirement they want to satisfy, and specifies their desired credits. They can supply an iCalendar (`.ics`) file or start with an empty calendar.
2. **Fit checks** — Deterministic tools check credits, supported requirement rules, and meeting conflicts, including required discussions and labs.
3. **Ranked shortlist** — Each recommendation identifies a feasible section combination, the requirement it matches, and the preferences that drove its ranking.

Limiting the initial program coverage gives the team a set of rules it can check against Cornell's published requirements before the demo. Recommending individual courses lets us test whether the interview improves discovery for students with either busy or empty calendars. **Stretch goals** include transcript-based degree audits, full-semester schedule optimization, and recommendations for labs, clubs, and events.

## Collaboration plan

We will meet weekly in person and use WhatsApp between meetings for questions, blockers, and decisions. We will run one shared AI-DLC Inception workflow. Everyone will discuss the agent's questions, review the resulting documents, and request changes or approve them together. After we agree on the units of work, we will divide implementation across the five teammates. Each owner will work on a GitHub branch and open a pull request for at least one teammate to review before merging.

## Repository guide

- [Vision](docs/vision.md) and [technical environment](docs/technical-environment.md) are the project's AI-DLC input documents.
- [Codex instructions](AGENTS.md), [Claude Code instructions](CLAUDE.md), and the [shared rule details](.aidlc-rule-details/) configure the development workflow.
