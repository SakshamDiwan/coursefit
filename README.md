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

The primary deliverable is a **hosted MCP service** that owns the interview and recommendation process. CourseFit decides when a preference is clear enough to search and when a follow-up could change the shortlist. It retrieves candidates, weighs the student's preferences, and explains why one course ranks above another using evidence from the course descriptions. The student's own **host assistant** does three things:
- translates the student's request into structured input
- reads the student's calendar, if they share it, and sends CourseFit only busy times
- presents CourseFit's questions and results

We will demo and dogfood CourseFit in **Claude Desktop** and **Claude Code or Codex**. Each student connects with a personal API key, and their plans are saved so they can come back later.

CourseFit will call hosted language models through **OpenRouter**, retrying a failed call once and then switching to a configured fallback model. Our initial benchmark includes:
- **Gemini 3.1 Flash Lite** and **DeepSeek V4 Flash** as low-cost options
- **Claude Sonnet 4.6** as a mid-priced option
- **Claude Opus 4.8** as a higher-cost reasoning model

A scripted harness runs the same Cornell student scenarios against each model to test whether the extra cost buys better follow-up questions and course rankings. Other OpenRouter models can enter the benchmark, and our own results will determine the demo model. The benchmark measures:

- **Interview quality** — Does a follow-up uncover a preference that changes the shortlist?
- **Recommendation quality** — Does every recommendation satisfy the student's hard constraints, and does the ranking reflect their stated priorities?
- **Explanation accuracy** — Does the course data support each reason given?
- **Speed and cost** — How quickly does the model respond, and what does a completed interview cost?

## MVP scope and feasibility

For the **December 16, 2026** MVP, CourseFit covers **Spring 2027**. It runs on **AWS Lambda** with **PostgreSQL on RDS**. A reviewed script scrapes the Cornell University Catalog for a small, team-verified set of Information Science BA and Business minor requirements. A snapshot of Spring 2027 offerings from the **Cornell Class Roster API** is refreshed daily. It covers sections, credits, meeting times, and prerequisite text. [Cornell Class Roster API documentation](https://classes.cornell.edu/content/FA26/api-details)

1. **Student inputs** — The student sets their desired credits and, optionally, a supported requirement. Their assistant can send busy times from their calendar, or they can start with an empty schedule. Constraints such as "I work Fridays after noon" apply immediately and stay editable.
2. **Fit checks** — Deterministic code checks credits, supported requirement rules, and meeting conflicts, including required discussions and labs. Prerequisites are shown but not enforced. Conditions CourseFit can't check are stated as assumptions the student can override.
3. **Ranked shortlist** — Five courses, each with a recommended feasible section combination, the requirement it matches, and the preferences that drove its ranking. Selecting a course updates remaining credits and refreshes the list.

Limiting the initial program coverage gives the team a set of rules it can check against Cornell's published requirements before the demo. **Stretch goals** include transcript-based degree audits, full-semester schedule optimization, semantic course search, and recommendations for labs, clubs, and events.

## Collaboration plan

We will meet weekly in person and use WhatsApp between meetings for questions, blockers, and decisions. We will run one shared AI-DLC Inception workflow. Everyone will discuss the agent's questions, review the resulting documents, and request changes or approve them together. After we agree on the units of work, we will divide implementation across the five teammates. Each owner will work on a GitHub branch and open a pull request for at least one teammate to review before merging.

## Repository guide

- [Vision](docs/vision.md) and [technical environment](docs/technical-environment.md) are the project's AI-DLC input documents.
- [Codex instructions](AGENTS.md), [Claude Code instructions](CLAUDE.md), and the [shared rule details](.aidlc-rule-details/) configure the development workflow.
