---
name: write-srd
description: Write a Software Requirements Document (SRD) from a Product Requirements Document (PRD). Explores the codebase, then defines what must be true about the software without solving the problem.
---

# Write SRD from PRD

## Overview

Given a PRD, produce a companion Software Requirements Document (SRD) that defines
what must be true about the software for the PRD to be satisfied.

You are the **calling agent**. The entire job — reading the PRD, exploring the codebase, writing the SRD, creating the SRD child bee — is done by the `apiary-srd-writer` subagent, a named agent definition installed in `.claude/agents/`, spawned via the Agent tool. You resolve the input, dispatch, and tell the user the results. You do not write the SRD yourself.

## 1. Resolve the Input

The user will provide either a Bee in the Idea hive, a PRD bee child of such a bee or nothing.
If nothing, ask them for the Bee in the Idea hive that has the PRD as a child.

## 2. Dispatch the SRD Writer

Dispatch `apiary-srd-writer` via the Agent tool:
`Agent(subagent_type: "apiary-srd-writer", prompt: <dispatch prompt>, model: <per the job → model mapping below>)`

Dispatch rules:

- **Background dispatch**: if the Agent tool schema accepts `run_in_background`, pass `run_in_background: true`; otherwise omit it — background is the harness default under Claude Code fork-subagents mode. When the completion notification fires, present the report (step 3).
- **Cold start**: every dispatch is a fresh spawn with a self-contained prompt. Dispatch prompts name the relevant ticket IDs; the subagent reads the tickets from the bees CLI itself.

The dispatch prompt must contain:
- The Idea Bee ID (and the PRD ticket ID if the user provided one)
- The repository path to explore

The subagent reads the PRD, explores the codebase, writes the SRD as a child of the Idea Bee with title "SRD" and status `larva`, and returns a report.

### Job → model mapping

Pass `model` at dispatch time — model choice belongs to this skill, not the agent definition:

| Agent | Model |
|---|---|
| `apiary-srd-writer` | opus |

## 3. Report to the User

When the subagent's report arrives, tell the user the results:
- The SRD ticket ID
- Each SR group with a one-line description
- Any **RESEARCH NEEDED** flags

## 4. Next steps

Suggest the user run `/req-review <idea-bee-id>` to review and finalize the docs before they can be marked `pupa`.
