---
name: write-srd
description: Write a Software Requirements Document (SRD) from a Product Requirements Document (PRD). Explores the codebase, then defines what must be true about the software without solving the problem.
---

# Write SRD from PRD

## Overview

Given a PRD, produce a companion Software Requirements Document (SRD) that defines
what must be true about the software for the PRD to be satisfied.

You are the **calling agent**. The entire job — reading the PRD, exploring the codebase, writing the SRD, creating the SRD child bee — is done by the `apiary-srd-writer` subagent, a named agent definition installed in `.claude/agents/`, spawned via the Agent tool. You resolve the input, dispatch, and tell the user the results. You do not write the SRD yourself.

Before working, read the engineering best practices guide referenced in the project's CLAUDE.md under "Documentation Locations". Where the guide conflicts with the PRD, SRD, plan or task, the guide wins.

## Approval guard

Run this first: no question, dispatch or ticket change comes before it. If you were given nothing, ask for the Idea Bee as step 1 says, then run this before anything else.

An SRD is written only once the User has approved the PRD. Read tickets with `bees show-ticket --ids <id>`.

1. PRD: given the PRD, that ticket. Given the Idea Bee, its t1 child titled `PRD`.
2. The PRD's `tags` must include `approved:yes`.

If either step fails, stop at once: change no ticket and dispatch nothing. When given `--no-questions`, or when `AGENT_DIRECTOR_INSTANCE_ID` is set, end your output with exactly one of these lines; otherwise tell the user the same and stop:
- `write-srd: needs input: <prd-id> lacks approved:yes — the User must approve it`
- `write-srd: needs input: no PRD found for <idea-id> — cannot check approved:yes`

This skill never adds `approved:yes`. Only the User approves.

## 1. Resolve the Input

The user will provide either a Bee in the Idea hive, a PRD bee child of such a bee or nothing.
If nothing, ask them for the Bee in the Idea hive that has the PRD as a child.

## 2. Dispatch the SRD Writer

Dispatch `apiary-srd-writer` via the Agent tool:
`Agent(subagent_type: "apiary-srd-writer", prompt: <dispatch prompt>)`

Dispatch rules:

- **Background dispatch**: if the Agent tool schema accepts `run_in_background`, pass `run_in_background: true`; otherwise omit it — background is the harness default under Claude Code fork-subagents mode. When the completion notification fires, present the report (step 3).
- **Cold start**: every dispatch is a fresh spawn with a self-contained prompt. Dispatch prompts name the relevant ticket IDs; the subagent reads the tickets from the bees CLI itself.

**In Codex CLI**, keep these rules with Codex's tools:
- Call `spawn_agent` with `agent_type: "apiary-srd-writer"` (pass it even if the tool does not list it; the agent is defined in `~/.codex/agents/apiary-srd-writer.toml` or the repo's `.codex/agents/`), `message`: the dispatch prompt, `fork_turns: "none"` (the cold start; where `spawn_agent` has no `fork_turns`, do not fork) and a `task_name` of lowercase letters, digits and underscores (e.g. `srd_writer_b_abc`). Pass no `model` or `reasoning_effort`: the agent file sets them.
- Codex sends no completion notification: wait for the report with `wait_agent` (prefer long waits, minutes). Close the agent with `close_agent` where that tool exists.

The dispatch prompt must contain:
- The Idea Bee ID (and the PRD ticket ID if the user provided one)
- The repository path to explore

The subagent reads the PRD, explores the codebase, writes the SRD as a child of the Idea Bee with title "SRD" and status `draft`, and returns a report.

## 3. Report to the User

When the subagent's report arrives, tell the user the results:
- The SRD ticket ID
- Each SR group with a one-line description
- Any **RESEARCH NEEDED** flags

## 4. Next steps

Suggest the user run `/req-review <idea-bee-id>` to review and finalize the docs before they can be marked `defined`.
