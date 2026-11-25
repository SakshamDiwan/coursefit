# CourseFit vision

## Product and audience

CourseFit helps Cornell undergraduates choose courses during pre-enrollment and add/drop. A student may know they need nine more credits, have a Business minor requirement in mind, and be interested in the financial side of startups. They still need to discover relevant courses, decide which ones interest them, and find a lecture and any required discussion or lab that fit together. CourseFit turns that search into a short conversation and a shortlist the student can refine one selection at a time.

The primary deliverable for December 16, 2026 is a hosted Model Context Protocol (MCP) service. It owns the interview, course discovery, fit checks, ranking, and explanations. The student's own AI assistant connects to it, collects input, reads the student's calendar if they want to share it, and presents the results. CourseFit interprets the student's preferences and chooses each follow-up. The team will demo and use CourseFit through Claude Desktop and Claude Code or Codex, the same way a student would. Longer term, Cornell could make the service available through different student-facing assistants.

## MVP experience

1. **Describe the academic goal.** Choose Spring 2027 as the target term and set the desired credits. Optionally pick a supported major or minor requirement. CourseFit shows which rules it can check for the 2026–27 catalog. A student can also skip the requirement and ask for something like "any three-credit course about AI policy."
2. **Supply availability.** The student's assistant can read their calendar and send CourseFit the busy times as weekly blocks or one-off dated blocks. Event titles and other details are never sent. Sharing a calendar is optional, and an empty schedule is a valid start. Students can add constraints in ordinary language, such as "I work Fridays after noon." CourseFit applies each constraint immediately, shows how it understood it, and lets the student correct it with a follow-up. Busy times and explicit must-haves are binding. "I prefer mornings" is a preference the student can trade off.
3. **Answer a useful question.** CourseFit draws out interests, learning objectives, career aspirations, and preferences from what the student has already said. It asks a follow-up only when the answer could change the search or ordering. For example, "Would you rather build a forecasting model or use financial statements to make a business decision?" distinguishes two plausible directions. After about four follow-ups CourseFit offers results by default. The student can keep refining or ask for results at any point.
4. **Compare explained recommendations.** The shortlist has five courses. Each result shows:
   - the course and its credits
   - one recommended set of required sections, with meeting times and class numbers
   - how many other feasible section sets exist (the student can ask to see them)
   - any supported requirement it matches

   The explanation connects the student's priorities to evidence in the course description and points out relevant trade-offs. Prerequisites and enrollment restrictions are shown but not enforced, because Cornell does not strictly enforce them. When CourseFit has to assume something it can't check, such as whether a course would double-count toward the student's major, it says so. The student can override that assumption, and the shortlist updates. If nothing meets a must-have, CourseFit explains the conflict and lets the student decide what to change.
5. **Build toward a credit goal.** When the student selects a section set, CourseFit adds its meetings to the plan, updates the remaining credits, and refreshes the shortlist. For a nine-credit goal, selecting a three-credit course leaves six to find. For variable-credit courses, the student chooses the credit value when selecting. The student makes each selection.
6. **Come back later.** Plans are saved, so the student can resume one later. A student can delete a plan at any time, and plans idle for 30 days are deleted automatically.

## Academic coverage

Start with the **Information Science BA** and the **Business minor** in the **2026–27 catalog**. For the demo, the team will verify a small fixed set of two to three categories from each program. More categories can be added one at a time. Before a requirement appears in the picker, the team checks its rule against the published page and records its source and catalog year. The Spring 2027 roster separately determines whether a course is offered and which sections can be taken together. [Cornell program directory](https://catalog.cornell.edu/programs/)

For example, the Information Science BA lists CS 1110 and CS 1112 as introductory programming options. The Business minor lists AEM 2210 among its financial accounting options and has separate unique-credit conditions. CourseFit can explain that a listed course matches a selected category. It states any condition it cannot check as an assumption. Determining a particular student's remaining degree requirements would also require their academic history. [Information Science BA](https://catalog.cornell.edu/programs/information-science-ba/) · [Business minor](https://catalog.cornell.edu/programs/business-minor/)

## How we will evaluate the MVP

- **Correctness.** Across the ten planned demo scenarios, every recommended section combination must pass every stated hard constraint. The scenarios include a busy schedule, an empty schedule, a required lab conflict, an unsupported requirement, and a second selection that changes what fits.
- **Value per question.** Compare the shortlist before and after follow-ups. Record the number of questions, what each answer revealed, and whether it improved the student's choices. Check each explanation against the course or rule it cites.
- **Model choice.** A scripted harness benchmarks hosted models through OpenRouter on the same scenarios. It compares relevance and explanation accuracy alongside response time and cost per interview. The demo target is a shortlist within 60 seconds after the student's last answer.

## Outside the MVP

Stretch goals:
- transcript ingestion and individual degree audits
- automatic full-semester optimization
- multi-semester planning
- semantic (embedding-based) course search
- recommendations for research labs, clubs, professional communities, or events

Outside the December deliverable:
- enrollment and seat alerts
- professor ratings
- NetID sign-in
- a standalone web client
- campus-wide deployment with high availability guarantees

## Open questions

- Which additional requirement categories can the team verify after the demo, particularly where course substitutions or double-counting rules apply?
- Which retrieval method and preference weights give students the most useful shortlist?
- Which OpenRouter model offers the strongest interview and explanation quality at acceptable latency and cost?
