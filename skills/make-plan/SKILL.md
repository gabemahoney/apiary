---
name: make-plan
description: Read documents from an Idea Bee and create a Plan Bee with Epics in the Plans hive to describe the work that needs to be done.
disable-model-invocation: false
---

# PRD/SRD to Plan

You are the **calling agent**. The planning itself — reading the PRD/SRD, exploring the repo, decomposing into Epics, creating the Plan Bee and Epic tickets with dependencies and statuses — is done by the `apiary-plan-writer` subagent, a named agent definition installed in `.claude/agents/`, spawned via the Agent tool. You resolve the input, dispatch, tell the user the results, and continue to hatch-epic. You do not write the plan yourself.

## Workflow

### 1. Setup

If there is no Plans hive nested inside the Ideas hive, ask the user if you can create one. It must have these child tiers:
- t1 — Epic / Epics
- t2 — Task / Tasks
- t3 — Subtask / Subtasks

### 2. Resolve the Idea Bee

The user should provide you a Bee in the Ideas hive. If they do not, list the Ideas hive and ask them which they want to hatch.
First check if the Idea Bee is in the `pupa` state (which means its ready to be worked on). If not, warn the user and ask if they want to continue.

### 3. Dispatch the Plan Writer

Dispatch `apiary-plan-writer` via the Agent tool:
`Agent(subagent_type: "apiary-plan-writer", prompt: <dispatch prompt>, model: <per the job → model mapping below>)`

Dispatch rules:

- **Background dispatch**: if the Agent tool schema accepts `run_in_background`, pass `run_in_background: true`; otherwise omit it — background is the harness default under Claude Code fork-subagents mode. When the completion notification fires, present the report (step 4).
- **Cold start**: every dispatch is a fresh spawn with a self-contained prompt. Dispatch prompts name the relevant ticket IDs; the subagent reads the tickets from the bees CLI itself.

The dispatch prompt must contain:
- The Idea Bee ID
- The Plans hive name
- The repository path to explore
- Any architectural decisions or constraints from your conversation with the user — use the user's exact specification, do not paraphrase

The subagent reads the requirements documents, explores the repo, creates the Plan Bee (with `up_deps` and `reference_materials` pointing at the Idea Bee) and its Epics in the Plans hive, sets up dependencies and statuses (Plan Bee `pupa`, Epics `larva`), and returns a report.

#### Job → model mapping

Pass `model` at dispatch time — model choice belongs to this skill, not the agent definition:
If the user requests a specific model, pass it at dispatch in place of the mapping default — still use the `apiary-*` agent, never a general-purpose one.

| Agent | Model |
|---|---|
| `apiary-plan-writer` | opus |

### 4. Report to the User

When the subagent's report arrives, tell the user the results as a markdown summary:
- Bee and Epics created
- Each Epic: ID, title, status, dependencies (if any)
- Dependency relationships created
- Any ambiguous decomposition decisions the writer flagged

### 5. Continue to hatch-epic

Load the `hatch-epic` skill and execute it for each Epic in dependency order until all are hatched.
Do not ask the user for permission — proceed automatically.
