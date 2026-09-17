---
name: write-prd
description: Write a Product Requirements Document (PRD) for a development effort. Adds or modifies a PRD Bee to a Bee in the Ideas hive that downstream skills (write-srd, req-review, hatch-epic) can consume.
---

# Write PRD

## Overview

The caller wants to write a PRD — a plain-english description of work to be done, stated as requirements. 
The PRD is the origin document for a development effort.
Downstream skills will consume it:

- `/write-srd` — turns the PRD into software requirements
- `/req-review` — reviews the PRD for completeness and executability
- `/hatch-epic` — breaks the work into Epics, Tasks, and Subtasks

The PRD must be thorough enough that an agent swarm can execute the work
autonomously, without needing to ask the user clarifying questions.

You are the **calling agent**. You run the interview with the user, then dispatch the drafting to the `apiary-prd-writer` subagent — a named agent definition installed in `.claude/agents/`, spawned via the Agent tool. You do not write the PRD yourself.

## Workflow

### 1. Determine the Idea Bee

The PRD will be a child of a Bee in the Ideas hive.
If the user did not provide a Bee id, ask it which Idea Bee you should build the PRD under.

### 2. Gather Requirements

You and the user may already have been chatting about an idea, if so, collect as much
information as you already have.

Then, interview the user to understand the work. Ask focused questions to fill gaps.

Key questions to explore:
- What is the Problem Statement?
- What are the Acceptance Criteria?
- What is explicitly out of scope?
- Are there edge cases or error scenarios to handle?
- Are there UI/UX considerations?
- Are there performance or security constraints?
- Does this depend on or affect other work?

Use `AskUserQuestion` to gather information efficiently. Present options where
possible rather than open-ended questions.

### 3. Dispatch the PRD Writer

Dispatch `apiary-prd-writer` via the Agent tool:
`Agent(subagent_type: "apiary-prd-writer", prompt: <dispatch prompt>, model: <per the job → model mapping below>)`

Dispatch rules:

- **Background dispatch**: if the Agent tool schema accepts `run_in_background`, pass `run_in_background: true`; otherwise omit it — background is the harness default under Claude Code fork-subagents mode. When the completion notification fires, present the report (step 4).
- **Cold start**: every dispatch is a fresh spawn with a self-contained prompt. The subagent cannot see this conversation and cannot ask the user anything — everything it needs must be in the dispatch prompt.

The dispatch prompt must contain:
- The Idea Bee ID (the subagent reads the ticket from the bees CLI itself)
- The full set of interview answers and any relevant context from your conversation with the user — serialized as text, complete enough that the writer never has to guess
- On a revision dispatch: the existing PRD ticket ID and the user's change notes

The subagent drafts the PRD, creates (or updates) it as a child of the Idea Bee with title "PRD" and status `larva`, and returns a report.

#### Job → model mapping

Pass `model` at dispatch time — model choice belongs to this skill, not the agent definition:
If the user requests a specific model, pass it at dispatch in place of the mapping default — still use the `apiary-*` agent, never a general-purpose one.

| Agent | Model |
|---|---|
| `apiary-prd-writer` | opus |

### 4. Report to the User

When the subagent's report arrives, tell the user the results:
- The PRD ticket ID
- One-line description of each section
- Assumptions the writer made
- Open questions the writer flagged

If the user wants changes, gather them and re-dispatch (fresh spawn, with the existing PRD ticket ID and the change notes in the prompt). Repeat until the user is satisfied.

### 5. Next steps

Suggest the user run `/req-review <idea-bee-id>` to review and finalize the docs before they can be marked `pupa`.
