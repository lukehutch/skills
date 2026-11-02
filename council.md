---
name: council
description: LLM Council — five advisors with opposing lenses, anonymized peer review, then a synthesis with a definite recommendation. Use for consequential decisions with real stakes, genuine uncertainty, and multiple defensible options (architecture, a pivot, a contested claim, competing designs). Always use when the user says "council this", "run the council", "pressure-test this", "stress-test this", "war room this", or "debate this".
---

# Council deliberation (LLM Council)

<!-- Source: github.com/tenfoldmarc/llm-council-skill — merged 2026-07-12, reviewed as data. Safe to delete. -->

For consequential decisions with real stakes, genuine uncertainty, and multiple defensible options — architecture, a pivot, a contested claim, competing designs. Not for questions with one right answer, creation tasks, or trivial choices; just answer those. Always triggered by "council this," "run the council," "pressure-test this," "stress-test this," "war room this," "debate this." Brainstorming generates and scores candidates; the council pressure-tests the choice among them.

- **Five advisors in parallel** (subagents or cleanly separated passes), lenses in tension: **Contrarian** — assumes a fatal flaw exists and hunts it; **First-Principles** — strips assumptions, asks whether this is even the right question; **Expansionist** — hunts the upside everyone missed; **Outsider** — zero context, catches the curse of knowledge; **Executor** — only whether it's doable and what the first step is. Enrich the question with real context (project files, memory, referenced data), then frame it neutrally — never steer. Each leans fully into its lens without self-balancing; balance comes from the panel.
- **Peer-review anonymized, order-randomized outputs**, judged on merit. Each reviewer answers: which is strongest and why; which has the biggest blind spot and what it is; what did ALL of them miss.
- **Synthesize without smoothing.** Independent convergence = high confidence; genuine clashes get both sides, never averaged; report blind spots only peer review surfaced. End with a definite recommendation — never "it depends" — plus the one concrete first step. The strongest argument wins even as a lone dissent.
- **Persist the transcript** (framed question, advisor outputs, reviews, synthesis) to a file, so a later session doesn't re-litigate it.
