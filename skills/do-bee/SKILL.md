---
name: do-bee
description: Proceed through each Epic in a Bee, doing the work described therin. Report questions and status back to caller.
---

## Overview

This skill orchestrates the work for a complete Bee ticket by:
1. Finding the Bee to work on and validating it is ready
2. Finding the best Epic to work on
   2.1. Validating the Epic is unblocked
   2.2. Validating the Epic description still makes sense after reviewing work completed in previous Epics
3. Spawning role subagents to complete the work described in the Epic
   3.1. Sending questions and requests for clarification or guidance to the caller
   3.2. Creating one git commit per Task that includes all changes for that Task
4. Looping 2-3 until all Epics are done, then:
   4.1. Dispatching reviewer subagents
   4.2. Addressing issues found by the reviewers
   4.3. Getting User approval
   4.4. Marking Bee and all child tickets as closed
   4.5. Outputting a final summary

### Execution model

You are the **calling agent**. Role players (Engineer, Test Writer, Doc Writer, Product Manager, and the reviewers) are **subagents** — named agent definitions installed in `.claude/agents/`, spawned via the Agent tool: `Agent(subagent_type: "<agent-name>", prompt: <task-specific dispatch prompt>)`.

Rules that apply to every dispatch in this skill:

- **Background dispatch**: if the Agent tool schema accepts `run_in_background`, pass `run_in_background: true`; otherwise omit it — background is the harness default under Claude Code fork-subagents mode. Every dispatch in this skill is a background dispatch.
- **Cold start**: subagents are never named, reused, or messaged mid-flight. Every dispatch is a fresh spawn with a self-contained prompt. Dispatch prompts name the relevant ticket IDs; subagents read the tickets from the bees CLI themselves.
- **Hub-and-spoke**: subagents never talk to each other. The working tree diff plus the returned report is the handoff. You serialize dependent roles and thread each report into the next role's dispatch prompt.
- **Reconciliation loop**: drive the work event-driven, not as one big blocking turn. Each tick: (1) observe current state — ticket store, working tree, in-flight Agent status; (2) reconcile — dispatch whatever fresh Agent invocations the delta requires; (3) yield the turn. The harness fires a completion notification when each background Agent finishes; that notification triggers the next tick. No polling, no sleep loops, no blocking waits inside a turn.
- **Compaction recovery**: when a compaction/summarization marker is visible in your conversation, everything before it is non-authoritative. Before dispatching the next tick's work after a compaction, re-read the authoritative sources in full — the bees CLI for ticket state, git for code state, and the filesystem for any state files. Conversation memory is never a substitute for these sources.
- **Delegate discipline**: you must not do role work yourself. You dispatch subagents, route reports on completion, and commit.

### 1. Find Bee to work on and validate

The user will either call without arguments, with a Bee id or with an Epic ID:
- If called without arguments, find all bees for this repo and ask the user which one they want to work on
- If called with a Bee id, find all Epics in the `pupa` state that are unblocked and ask which one they want to work on
- If called with an Epic id, find the Bee that is a parent of that Epic and use that Bee

You will ultimately get the Bee ID you need to work on.
Validate it is ready for work:
- Must have a status of `pupa` or `worker`
- If it has `up_dependencies` they must be in `pupa` state

#### Validate worktree
You should have been launched in a worktree with a name like "b_Wx7" for a bee called "b.Wx7".
If you are not launched in such a worktree, use AskUserQuestion to confirm they want to proceed. 
- Tell them what directory and branch they are on.

### 2. Find Epic to work on and validate

Find all Epics in the Bee and recommend the best one to work on first:
- Must have a status of `pupa` or `worker`
- If it has `up_dependencies` they must be in `finished` state


#### Check if stale
Be aware that the Epic was written before coding started. If the Epic has `up_dependencies` that have been completed then
you must review the work actually done in those Epics to see if this current Epic description is stale:

1. Review the git diff to understand what was actually implemented
2. Read the upcoming Epic and its Tasks/Subtasks
3. Update any Task or Subtask descriptions that are now stale given what the worker built (e.g., file paths changed, function signatures differ, new modules were created)

#### Mark status when ready to start work

If ready, mark the Epic status with `status=worker` to show work has started on the Epic

### 3. Dispatch Subagents to Execute Tasks

Before dispatching, load all Tasks and Subtasks for the Epic:
- Use `show_ticket()` on the Epic to get the `children` array (Task IDs)
- For each Task, fetch its full details including its own `children` array (Subtasks)
- Read every Subtask — these contain the detailed instructions (Context, What Needs to Change, Key Files, Acceptance Criteria) that the role subagents must follow
- Sort Tasks in dependency order (check each Task's `up_dependencies`) to ensure no Task is blocked when executed
- Verify at least 1 Task exists with at least 1 Subtask, and all are ready for work: `status!=larva`
- Mark the current Task with `status=worker` to show work has started
- Mark the Bee with `status=worker` to show work has started (if not already set)

Dispatch role subagents to work on an individual Task.
**IMPORTANT: You must not do role work yourself. Dispatch subagents, route their reports, and commit.**

Choose which roles are required.
- If source code is being modified or created, dispatch `apiary-engineer`.
  - **The Engineer is responsible for source code. It does *not* know how to update unit tests or docs!**
- If test code is being modified or created, dispatch one or more `apiary-test-writer` subagents.
    - **The Test Writer is responsible for unit tests. It does *not* know how to update source code or docs!**
    - If there are multiple files that need updating, the Task should have one Subtask per file
    - If so, dispatch multiple Test Writers to work on each Subtask and File in parallel
    - It might be necessary to order them so that the first one creates or modifies any shared fixtures
- If docs need to be modified or created, dispatch `apiary-doc-writer`.
- If this is the initial dispatch for a Task, **always** dispatch `apiary-product-manager`.
  - If this is a re-work dispatch after reviewer feedback you may **optionally** choose to not dispatch the Product Manager, if the work is minor enough and will not impact Product functionality

Parallel dispatch is bounded only by the rules above: one Test Writer per test-file Subtask, ordered when shared fixtures require it.

#### Sequencing across ticks

Coordination between roles is sequencing you own, spread across reconciliation ticks:
- On the Engineer's completion tick, dispatch the Test Writer(s) and Doc Writer with the Engineer's report threaded into their dispatch prompts — they review the Engineer's work as part of their instructions.
- Dispatch the Product Manager sequentially with whatever prior reports you want it to review — commonly at the end for a full review of the Task, but nothing prevents interim PM dispatches (e.g., a PM check on the Engineer's design before a Test Writer starts writing against it).
- One hard constraint: the PM must not run in parallel with a writer whose output it is supposed to review — concurrent subagents cannot see each other's in-flight work.

After dispatching this tick's work, yield — the harness fires a completion notification per Agent finish, which drives the next reconciliation tick.

Each role's Responsibilities and Instructions live in its agent definition (`.claude/agents/apiary-*.md`). Your dispatch prompt supplies the job-specific context: the Task and Subtask IDs to work, any threaded reports from prior roles, and any re-work items from reviewer feedback.

#### 4.1 After Each Task

When a Task and all its Subtasks are done (all reviewer feedback addressed or ignored):

1. Create one git commit for the Task. Use system or project defined guidance on git usage. **NEVER push to remote — committing only.**
2. Mark the Task as `status=finished` (Subtasks were marked finished by each subagent as they completed their work).
3. Output the summary below to the screen and continue to the next Task

```
## Task [N] of [total] Complete: [task-title]

**Task ID**: <task-id>
**Files Changed**: [count] files ([list key filenames if < 5, otherwise just count])
**Reviews**: [Code review: X issues found/None needed | Docs review: Y issues found/None needed]
**Ignored Review Feedback**: [list items that were flagged by code-review or doc-review but Director chose not to address, or "None"]
**Follow-up Tasks Created**: [count, if any] [list task-ids if created]
One of:
- Proceeding to next Task <task-id>
- Final Task, moving on to Final Reviews 
```

#### 4.2 Find next Epic or move to Final Review
If there are more Epics to work on, ask the user if they want to continue with the next logical one. If so, clear your context window and go back to step 2.
If not, move to final Bee review.

### 5. Final Bee-level Code, Doc and Eng reviews

Once all Epics in the Bee are done, dispatch the applicable reviewer subagents as parallel background dispatches (up to three, one per applicable reviewer) in a single reconciliation tick:

If you dispatched the Engineer during execution, dispatch `apiary-code-reviewer`.
If you dispatched the Test Writer during execution, dispatch `apiary-test-reviewer`.
If you dispatched the Doc Writer during execution, dispatch `apiary-doc-reviewer`.

Each reviewer subagent invokes its corresponding review skill (/code-review, /test-review, /doc-review) and returns a numbered list of freeform findings (an explicit "no findings" statement when there are none). Include this line in each reviewer's dispatch prompt: "Your review skill is a read-only operation — invoke it via the Skill tool; do not decline the invocation."

- Get the feedback, and make a judgement call about whether that work must be done
  - If so, **spawn fresh role subagents as needed** to do the work
    - **IMPORTANT** Do not do the work yourself — dispatch, route reports, and commit.
    - If the feedback was minor enough, you may choose to **NOT** dispatch the Product Manager on this iteration 
    - Dispatch any role subagents required to do the work you deem necessary from the reviewer findings
  - If not, move on to Final Review but you MUST share the ignored feedback for review
  - Note: This could create an infinite loop so you may ignore feedback so long as you present it in Final Review

  
### 6. Final Output

When **all** Epics in the Bee are done, you must show the User the full list of all Reviewer feedback you chose to ignore.
- Use the AskUserQuestion tool to ask the User if they want you to act on any of these, or just continue.

For each Acceptance Criteria, either demonstrate it directly (via test or script) or instruct the user how to validate it manually. Then use `AskUserQuestion` to get official sign-off on the Acceptance Criteria.

Then use `AskUserQuestion` with:
- Question: "Are you ready to mark this Bee as finished?"
- Options:
  - "Yes, mark as finished"
  - "No, we have more work to do"

### 7. Mark Bee Complete

Once the user approves the Bee as finished:

1. Mark all Epics in the Bee as `status=finished`:
```bees
update_ticket(ticket_id="<epic-id>", status="finished")
```

2. Verify all Epics are now `finished`, then mark the Bee itself:
```bees
update_ticket(ticket_id="<bee-id>", status="finished")
```

### 8. Output Final Summary

```markdown
## Bee Execution Complete: [bee-title]

**Bee ID**: <bee-id>
**Epics Completed**: [count]
**Tasks Completed**: [count]
**Bee Status**: Finished

All work has been synced to git.
```

### 9. Further testing and merging

Instruct the user to perform whatever further testing they want to do, then invoke the `teardown_worktree` skill to merge and teardown the worktree
