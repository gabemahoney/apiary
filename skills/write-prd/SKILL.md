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

Before working, read the engineering best practices guide referenced in the project's CLAUDE.md under "Documentation Locations". Where the guide conflicts with the PRD, SRD, plan or task, the guide wins.

### Approval guard

Run this first: no question, dispatch or ticket change comes before it. If you were not given an Idea Bee ID, ask for it as step 1 says, then run this before anything else.

A PRD is written only once the User has approved the Idea: the Idea Bee's `tags` (`bees show-ticket --ids <idea-id>`) must include `approved:yes`.

If the tag is missing, stop at once: change no ticket and dispatch nothing. When given `--no-questions`, or when `AGENT_DIRECTOR_INSTANCE_ID` is set, end your output with exactly this line; otherwise tell the user the same and stop:
- `write-prd: needs input: <idea-id> lacks approved:yes — the User must approve it`

This skill never adds `approved:yes`. Only the User approves.

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
`Agent(subagent_type: "apiary-prd-writer", prompt: <dispatch prompt>)`

Dispatch rules:

- **Background dispatch**: if the Agent tool schema accepts `run_in_background`, pass `run_in_background: true`; otherwise omit it — background is the harness default under Claude Code fork-subagents mode. When the completion notification fires, present the report (step 4).
- **Cold start**: every dispatch is a fresh spawn with a self-contained prompt. The subagent cannot see this conversation and cannot ask the user anything — everything it needs must be in the dispatch prompt.

**In Codex CLI**, keep these rules with Codex's tools:
- Call `spawn_agent` with `agent_type: "apiary-prd-writer"` (pass it even if the tool does not list it; the agent is defined in `~/.codex/agents/apiary-prd-writer.toml` or the repo's `.codex/agents/`), `message`: the dispatch prompt, `fork_turns: "none"` (the cold start; where `spawn_agent` has no `fork_turns`, do not fork) and a `task_name` of lowercase letters, digits and underscores (e.g. `prd_writer_b_abc`). Pass no `model` or `reasoning_effort`: the agent file sets them.
- Codex sends no completion notification: wait for the report with `wait_agent` (prefer long waits, minutes). Close the agent with `close_agent` where that tool exists.
- There is no AskUserQuestion (step 2): ask in plain text in your reply and end your turn.

The dispatch prompt must contain:
- The Idea Bee ID (the subagent reads the ticket from the bees CLI itself)
- The full set of interview answers and any relevant context from your conversation with the user — serialized as text, complete enough that the writer never has to guess
- On a revision dispatch: the existing PRD ticket ID and the user's change notes

The subagent drafts the PRD, creates (or updates) it as a child of the Idea Bee with title "PRD" and status `draft`, and returns a report.

### 4. Report to the User

When the subagent's report arrives, tell the user the results:
- The PRD ticket ID
- One-line description of each section
- Assumptions the writer made
- Open questions the writer flagged

If the user wants changes, gather them and re-dispatch (fresh spawn, with the existing PRD ticket ID and the change notes in the prompt). Repeat until the user is satisfied.

### 5. Next steps

Suggest the user run `/req-review <idea-bee-id>` to review and finalize the docs before they can be marked `defined`.
`/write-srd`, `/make-plan`, `/hatch-epic`, `/do-bee` and `/run-lanes` refuse to run until the User approves the PRD by tagging the PRD ticket `approved:yes`.
