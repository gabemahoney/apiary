---
name: make-plan
description: Read documents from an Idea Bee and create a Plan Bee with Epics in the Plans hive to describe the work that needs to be done.
disable-model-invocation: false
---

# PRD/SRD to Plan

You are the **calling agent**. The planning itself — reading the PRD/SRD, exploring the repo, decomposing into Epics, creating the Plan Bee and Epic tickets with dependencies and statuses — is done by the `apiary-plan-writer` subagent, a named agent definition installed in `.claude/agents/`, spawned via the Agent tool. You resolve the input, dispatch, tell the user the results, and continue to hatch-epic. You do not write the plan yourself.

## Workflow

Before working, read the engineering best practices guide referenced in the project's CLAUDE.md under "Documentation Locations". Where the guide conflicts with the PRD, SRD, plan or task, the guide wins.

### Approval guard

Run this first, before step 1: no question, dispatch or ticket change comes before it. If you were not given an Idea Bee ID, ask for it as step 2 says, then run this before anything else.

A plan is made only once the User has approved the Idea's PRD. Read tickets with `bees show-ticket --ids <id>`.

1. PRD: the Idea Bee's t1 child titled `PRD`.
2. The PRD's `tags` must include `approved:yes`.

If either step fails, stop at once: change no ticket and dispatch nothing. When given `--no-questions`, or when `AGENT_DIRECTOR_INSTANCE_ID` is set, end your output with exactly one of these lines; otherwise tell the user the same and stop:
- `make-plan: needs input: <prd-id> lacks approved:yes — the User must approve it`
- `make-plan: needs input: no PRD found for <idea-id> — cannot check approved:yes`

This skill never adds `approved:yes`. Only the User approves.

### 1. Setup

If there is no Plans hive nested inside the Ideas hive, ask the user if you can create one. It must have these child tiers:
- t1 — Epic / Epics
- t2 — Task / Tasks
- t3 — Subtask / Subtasks

### 2. Resolve the Idea Bee

The user should provide you a Bee in the Ideas hive. If they do not, list the Ideas hive and ask them which they want to hatch.
First check if the Idea Bee is in the `defined` state (which means it is fully written up; approval is the PRD's `approved:yes` tag, checked above). If not, warn the user and ask if they want to continue.

### 3. Dispatch the Plan Writer

Dispatch `apiary-plan-writer` via the Agent tool:
`Agent(subagent_type: "apiary-plan-writer", prompt: <dispatch prompt>)`

Dispatch rules:

- **Background dispatch**: if the Agent tool schema accepts `run_in_background`, pass `run_in_background: true`; otherwise omit it — background is the harness default under Claude Code fork-subagents mode. When the completion notification fires, present the report (step 4).
- **Cold start**: every dispatch is a fresh spawn with a self-contained prompt. Dispatch prompts name the relevant ticket IDs; the subagent reads the tickets from the bees CLI itself.

**In Codex CLI**, keep these rules with Codex's tools:
- Call `spawn_agent` with `agent_type: "apiary-plan-writer"` (pass it even if the tool does not list it; the agent is defined in `~/.codex/agents/apiary-plan-writer.toml` or the repo's `.codex/agents/`), `message`: the dispatch prompt, `fork_turns: "none"` (the cold start; where `spawn_agent` has no `fork_turns`, do not fork) and a `task_name` of lowercase letters, digits and underscores (e.g. `plan_writer_b_abc`). Pass no `model` or `reasoning_effort`: the agent file sets them.
- Codex sends no completion notification: wait for the report with `wait_agent` (prefer long waits, minutes). Close the agent with `close_agent` where that tool exists.
- "Load the `hatch-epic` skill" (step 5) means: read `~/.agents/skills/hatch-epic/SKILL.md` (or the repo's `.agents/skills/hatch-epic/SKILL.md`) and follow it, including its own In Codex CLI notes.

The dispatch prompt must contain:
- The Idea Bee ID
- The Plans hive name
- The repository path to explore
- An instruction to read the engineering best practices guide, the test writing guide and the doc writing guide referenced in the repo's CLAUDE.md under "Documentation Locations", plan the work they require, and plan no doc work the doc writing guide doesn't allow
- Any architectural decisions or constraints from your conversation with the user — use the user's exact specification, do not paraphrase

The subagent reads the requirements documents, explores the repo, creates the Plan Bee (with `up_deps` and `reference_materials` pointing at the Idea Bee) and its Epics in the Plans hive, sets up dependencies and statuses (Plan Bee `defined`, Epics `draft`), and returns a report.

### 4. Report to the User

When the subagent's report arrives, tell the user the results as a markdown summary:
- Bee and Epics created
- Each Epic: ID, title, status, dependencies (if any)
- Dependency relationships created
- Any ambiguous decomposition decisions the writer flagged

### 5. Continue to hatch-epic

Load the `hatch-epic` skill and run it once per Epic — pass each Epic ID explicitly, in dependency order — until EVERY Epic in the Plan Bee is hatched (status `defined`). Run fully automatically: no user prompts between Epics.
Hatching is planning only: do NOT invoke `do-bee` or begin any execution until all Epics are hatched. Execution is a separate, later step. (During execution, do-bee reconciles already-hatched Epics against completed work — never a reason to defer hatching.)
If hatching is interrupted mid-batch, resume `hatch-epic` for the remaining `draft` Epics before any execution begins.
After hatching, query the Plan Bee's child Epics and confirm none are `draft` before declaring the plan ready.
