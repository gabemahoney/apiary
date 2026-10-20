---
name: req-review
description: Review product and software requirement documents and provide suggestions
disable-model-invocation: false
---

# Overview

Your role is to help the user get Product Requirement and Software Requirement documents
into a good enough state that they can be executed autonomously by a swarm of agents.

You are the **calling agent**. The review itself — reading the documents, reviewing the codebase, running the checklists — is done by the `apiary-req-reviewer` subagent, a named agent definition installed in `.claude/agents/`, spawned via the Agent tool. You resolve the input, dispatch, tell the user the results, and handle status promotion. You do not perform the review yourself.

Before working, read the engineering best practices guide referenced in the project's CLAUDE.md under "Documentation Locations". Where the guide conflicts with the PRD, SRD, plan or task, the guide wins. Reviewers must report any violation of the guide as a blocking finding, even when the spec asks for the violating behaviour.

# 1. Resolve the Input

The user will provide a Bee in the Ideas hive (or its PRD/SRD children) whose documents should be reviewed.
If they do not, ask them which Idea Bee's documents to review.

# 2. Dispatch the Reviewer

Dispatch `apiary-req-reviewer` via the Agent tool:
`Agent(subagent_type: "apiary-req-reviewer", prompt: <dispatch prompt>)`

Dispatch rules:

- **Background dispatch**: if the Agent tool schema accepts `run_in_background`, pass `run_in_background: true`; otherwise omit it — background is the harness default under Claude Code fork-subagents mode. When the completion notification fires, present the report (step 3).
- **Cold start**: every dispatch is a fresh spawn with a self-contained prompt. Dispatch prompts name the relevant ticket IDs; the subagent reads the tickets from the bees CLI itself.

**In Codex CLI**, keep these rules with Codex's tools:
- Call `spawn_agent` with `agent_type: "apiary-req-reviewer"` (pass it even if the tool does not list it; the agent is defined in `~/.codex/agents/apiary-req-reviewer.toml` or the repo's `.codex/agents/`), `message`: the dispatch prompt, `fork_turns: "none"` (the cold start; where `spawn_agent` has no `fork_turns`, do not fork) and a `task_name` of lowercase letters, digits and underscores (e.g. `req_reviewer_b_abc`). Pass no `model` or `reasoning_effort`: the agent file sets them.
- Codex sends no completion notification: wait for the report with `wait_agent` (prefer long waits, minutes). Close the agent with `close_agent` where that tool exists.
- There is no AskUserQuestion (Next Steps): ask in plain text in your reply and end your turn.

The dispatch prompt must contain:
- The Idea Bee ID (or the individual PRD/SRD ticket IDs)
- The repository path relevant to the documents

The subagent is read-only: it reviews and reports; it never updates tickets. All status changes happen in step 4, by you.

# 3. Report to the User

When the subagent's report arrives, tell the user the results:
- An overview of all issues (criticality, short title, short summary, format it pretty), most critical first, with the reviewer's suggested fixes
- The reviewer's per-document readiness verdict

If the user wants issues addressed, the fixes belong to the authoring skills — suggest re-running `/write-prd` or `/write-srd` with the findings, then `/req-review` again. Re-dispatch (fresh spawn) for any re-review.

# Next Steps

After presenting the results, use AskUserQuestion to ask the user for each doc reviewed:
- "Mark [PRD/SRD] as `defined`?"
  - Options: "Yes, mark as defined" / "No, more work needed"
- If yes, update the doc's ticket status to `defined`.

After marking docs, check if **both** the PRD and SRD are now `defined`. If so, use AskUserQuestion to ask:
- "Both PRD and SRD are defined. Mark the Idea Bee as `defined` too?"
  - Options: "Yes, mark as defined" / "No, not yet"
- If yes, update the Idea Bee's status to `defined`.

Then recommend:
- `/make-plan <bee-id>` — create a Plan Bee with Epics ready for implementation
