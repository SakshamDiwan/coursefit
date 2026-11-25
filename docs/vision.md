# CourseFit vision

## Product and audience

CourseFit helps Cornell undergraduates choose courses during pre-enrollment and add/drop. A student may know they need nine more credits, have a Business minor requirement in mind, and be interested in the financial side of startups. They still need to discover relevant courses, decide which ones interest them, and find a lecture and any required discussion or lab that fit together. CourseFit turns that search into a short conversation and a shortlist the student can refine one selection at a time.

The primary deliverable for December 16, 2026 is a Model Context Protocol (MCP) service. It owns the interview, course discovery, fit checks, ranking, and explanations. A host assistant collects student input, can translate it into the tool's structured fields, and presents the results. CourseFit interprets the student's preferences and chooses each follow-up. A simple React client can demonstrate the same service. Longer term, Cornell could make the service available through different student-facing assistants.

## MVP experience

1. **Describe the academic goal.** Select a target term, desired credits, and any applicable supported major or minor and requirement. CourseFit shows which rules it can check for the student's catalog year.
2. **Supply availability.** Import an iCalendar file or start with an empty calendar. Events apply on the dates they cover. A calendar with no events in the target term contributes no busy intervals. Add missing constraints in ordinary language, such as “I work Fridays after noon.” Calendar conflicts and explicit must-haves are binding. “I prefer mornings” is a preference the student can trade off.
3. **Answer a useful question.** CourseFit elicits interests, learning objectives, career aspirations, and preferences from what the student has already said. It asks a follow-up when an answer could change the search or ordering. For example, “Would you rather build a forecasting model or use financial statements to make a business decision?” distinguishes two plausible directions. There is no fixed question count. The student can ask for results at any point.
4. **Compare explained recommendations.** Each result identifies the course, credits, a compatible set of required sections with meeting times and class numbers, and any supported requirement it matches. The explanation connects the student's priorities to evidence in the course description and identifies relevant trade-offs. If nothing meets a must-have, CourseFit explains the conflict and lets the student decide what to change.
5. **Build toward a credit goal.** When the student selects a section combination, CourseFit adds its meetings to the plan, updates the remaining credits, and refreshes the shortlist. For a nine-credit goal, selecting a three-credit course leaves six to find. The student makes each selection.

## Academic coverage

Start with the **Information Science BA** and the **Business minor** in the **2026–27 catalog**. Add coverage one requirement at a time. Before offering a requirement in the picker, the team checks its rule against the published page and records its source and catalog year. The term roster separately establishes whether a course is offered and which sections can be taken together. [Cornell program directory](https://catalog.cornell.edu/programs/)

For example, the Information Science BA lists CS 1110 and CS 1112 as introductory programming options. The Business minor lists AEM 2210 among its financial accounting options and has separate unique-credit conditions. CourseFit can explain that a listed course matches a selected category. Determining a particular student's remaining degree requirements would also require their academic history. [Information Science BA](https://catalog.cornell.edu/programs/information-science-ba/) · [Business minor](https://catalog.cornell.edu/programs/business-minor/)

## How we will evaluate the MVP

- **Correctness.** Across the ten planned demo scenarios, every recommended section combination must pass all stated hard constraints. Include a busy calendar, an empty calendar, a required lab conflict, an unsupported requirement, and a second selection that changes what fits.
- **Discovery.** Give classmates a keyword-search shortlist and a personalized shortlist drawn from the same feasible offerings, with the labels hidden. Record which they prefer and why. The existing discovery target is for at least three of five classmates to find a course they would consider taking that they had not previously considered.
- **Value per question.** Compare the shortlist before and after follow-ups. Record the number of questions, what each answer revealed, and whether it improved the student's choices. Check explanations against the cited course or rule.
- **Model choice.** Benchmark hosted models through OpenRouter on the same scenarios. Compare relevance and explanation accuracy alongside response time and cost per interview. The demo target is a shortlist within 60 seconds after the student's last answer.

## Outside the MVP

Transcript ingestion and individual degree audits, automatic full-semester optimization, multi-semester planning, and recommendations for research labs, clubs, professional communities, or events are stretch goals. Enrollment, seat alerts, professor ratings, NetID sign-in, and campus-wide deployment are outside the December deliverable.

## Open questions

- Which additional requirement categories can the team verify in time, particularly where course substitutions or double-counting rules apply?
- Which retrieval method and preference weights give students the most useful shortlist compared with keyword search?
- Which OpenRouter model offers the strongest interview and explanation quality at acceptable latency and cost?
