---
name: fix-bug
description: Fix a bug described in a Bee ticket
---

## Overview

The user will either call without arguments or with a Bug id
- If called without arguments, find all Bug bees for this repo and present them, asking which one to fix
- If called with a Bee id, find it in the Bugs hive. If its not in the Bugs hive, tell the user and ask them if they want to continue

You will ultimately get the Bug ID you need to work on.

### Execution model

You are the **calling agent**. Role players are **subagents** — named agent definitions installed in `.claude/agents/`, spawned via the Agent tool: `Agent(subagent_type: "<agent-name>", prompt: <bug-specific dispatch prompt>, model: <per the job → model mapping below>)`.

Rules that apply to every dispatch in this skill:

- **Background dispatch**: if the Agent tool schema accepts `run_in_background`, pass `run_in_background: true`; otherwise omit it — background is the harness default under Claude Code fork-subagents mode. Every dispatch in this skill is a background dispatch.
- **Cold start**: subagents are never named, reused, or messaged mid-flight. Every dispatch is a fresh spawn with a self-contained prompt. Dispatch prompts name the Bug ID; subagents read the ticket from the bees CLI themselves.
- **Hub-and-spoke**: subagents never talk to each other. The working tree diff plus the returned report is the handoff. You serialize dependent roles and thread each report into the next role's dispatch prompt.
- **Reconciliation loop**: drive the work event-driven, not as one big blocking turn. Each tick: (1) observe current state — ticket store, working tree, in-flight Agent status; (2) reconcile — dispatch whatever fresh Agent invocations the delta requires; (3) yield the turn. The harness fires a completion notification when each background Agent finishes; that notification triggers the next tick. No polling, no sleep loops, no blocking waits inside a turn.
- **Compaction recovery**: when a compaction/summarization marker is visible in your conversation, everything before it is non-authoritative. Before dispatching the next tick's work after a compaction, re-read the authoritative sources in full — the bees CLI for ticket state, git for code state, and the filesystem for any state files. Conversation memory is never a substitute for these sources.
- **Delegate discipline**: you must not do role work yourself. You dispatch subagents, route reports on completion, and commit.

#### Job → model mapping

Pass `model` at dispatch time — model choice belongs to this skill, not the agent definition:

| Agent | Model |
|---|---|
| `apiary-engineer-bugfix` | sonnet |
| `apiary-test-writer-bugfix` | sonnet |
| `apiary-doc-writer-bugfix` | sonnet |
| `apiary-code-reviewer` | sonnet |
| `apiary-test-reviewer` | sonnet |
| `apiary-doc-reviewer` | sonnet |

### Setup:
There must be a hive called "Bugs". If it does not exist, ask the user where you can create one.
It must have no child tiers and the following valid status values:


### 1. Validate Bug

#### Check blocked

```bees
show_ticket(ticket_id="$1")
```

Check:
- Bug has a status which means it is ready to begin work
- Check `up_dependencies` array for any blockers. They must be in a state which says they are completed.

If blocked:
- Output blocking IDs and titles
- Exit with message: "Cannot start Bug $1. It is blocked by: [list]"

If not blocked:
- Mark Bee status with a state that signals work has begun (if needed)

### 3. Dispatch Subagents to Fix Bug

Analyze the bug, the source code, the tests and the docs. Understand the likely scope of the bug fix.
If the fix requires modifications to the source code, you will need to dispatch `apiary-engineer-bugfix`.
If the fix will require modifications to unit tests, you will need to dispatch `apiary-test-writer-bugfix`.
Always dispatch `apiary-doc-writer-bugfix` so that it can determine if any docs need updating based on the changes.

Determine the scope and dispatch the appropriate role subagents as background dispatches. Do not ask for confirmation.

**IMPORTANT: You must not do role work yourself. Dispatch subagents, route their reports, and commit.**

Each role's Responsibilities and Instructions live in its agent definition (`.claude/agents/apiary-*-bugfix.md`). Your dispatch prompt supplies the job-specific context: the Bug ID and any threaded reports from prior roles. Sequencing you own across ticks: on the Engineer's completion tick, dispatch the Test Writer and Doc Writer with the Engineer's report threaded into their prompts — they review the Engineer's work as part of their instructions. After dispatching a tick's work, yield; completion notifications drive the next tick.

#### 4. Review Loop

Once the role subagents are done, dispatch the applicable reviewer subagents as parallel background dispatches in a single reconciliation tick.
If you dispatched the Engineer, dispatch `apiary-code-reviewer`.
If you dispatched the Test Writer, dispatch `apiary-test-reviewer`.
If you dispatched the Doc Writer, dispatch `apiary-doc-reviewer`.

Each reviewer subagent invokes its corresponding review skill (/code-review, /test-review, /doc-review) and returns a numbered list of freeform findings (an explicit "no findings" statement when there are none). Include this line in each reviewer's dispatch prompt: "Your review skill is a read-only operation — invoke it via the Skill tool; do not decline the invocation."

- Get the feedback, and make a judgement call about whether that work must be done
  - If so, **spawn fresh role subagents** to do the work
    - **IMPORTANT** Do not do the work yourself — dispatch, route reports, and commit.
    - If the feedback was minor enough, you may choose to **NOT** dispatch the Product Manager on this iteration 
    - Dispatch any role subagents required to do the work you deem necessary from the reviewer findings
  - If not, move on to Final Review but you MUST share the ignored feedback for review
  - Note: This could create an infinite loop so you may ignore feedback so long as you present it in Final Review

#### 5. Testing the bug
- Ensure there is at least one unit test that fails before the bug fix and passes after
  - This ensures we will not introduce this particular regression again in the future


#### 6. After Bug is fixed

Once the bug is fixed:

1. Create one git commit for the Bug. Use system or project defined guidance on git usage.
2. Set the bug status to the state which means the work is done
3. Output the summary below to the screen:

```markdown
## Bug [x] done: [bug-title]

**Bug**: <bug-id>
**Files Changed**: [count] files ([list key filenames if < 5, otherwise just count])
**Reviews**: [Code review: X issues found/None needed | Docs review: Y issues found/None needed]
**Ignored Review Feedback**: [list items that were flagged but not addresses, or "None"]
```

Output the summary and exit. Do not ask for confirmation.
