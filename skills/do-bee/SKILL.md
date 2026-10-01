---
name: do-bee
description: Proceed through each Epic in a Bee, doing the work described therin. Report questions and status back to caller.
---

## Overview

This skill orchestrates the work for a complete Bee ticket by:
1. Finding the Bee to work on and validating it is ready (all child Epics hatched — none `larva`)
2. Finding the best Epic to work on
   2.1. Validating the Epic is unblocked
   2.2. Reconciling the Epic and downstream plan against work completed in previous Epics (re-plan, not hatch)
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
- If it has `up_dependencies` they must be `pupa` or later (not `larva`). (This leniency is deliberate: Bee-level dependencies express planning order, not "code already written".)
- All child Epics must be `pupa` or later — none `larva`. A `larva` Epic means hatching is incomplete: stop and tell the caller to finish `hatch-epic` for every remaining `larva` Epic under this Bee. Never hatch Epics yourself; never execute a partially hatched Bee.

#### Validate worktree
You should have been launched in a worktree with a name like "b_Wx7" for a bee called "b.Wx7".
If you are not launched in such a worktree, use AskUserQuestion to confirm they want to proceed. 
- Tell them what directory and branch they are on.

### 2. Find Epic to work on and validate

Find all Epics in the Bee and recommend the best one to work on first:
- Must have a status of `pupa` or `worker`
- If it has `up_dependencies` they must be in `finished` state
- If ANY Epic under the Bee is still `larva`, that is an error — the Bee was handed off before hatching completed. Stop and report it to the caller; do not hatch it yourself and do not execute around it.


#### Reconcile the plan against completed work (re-plan, not hatch)
All Epics were hatched up front, before any coding started, so each completed Epic can make the remaining plan stale. Before starting work on each Epic (skip this pass only when no Epic in the Bee is `finished` yet — nothing has been built):

1. Review the git diff of the Epics completed so far to understand what was actually built
2. Read this upcoming Epic and its Tasks/Subtasks; update any descriptions now stale given what was built (file paths changed, function signatures differ, new modules created)
3. Scan downstream not-yet-started Epics for descriptions the completed work has outright invalidated (removed files, renamed modules, changed contracts) and fix those; leave cosmetic drift for that Epic's own reconcile pass when it comes up

This is RE-PLANNING: you are editing Tasks/Subtasks of Epics that are already hatched (`pupa`). It NEVER creates Tasks for a `larva` Epic — that is hatching, which finished before this skill started. This reconcile pass is exactly why all hatching can and must happen up front.

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
- If test code is being modified or created, dispatch `apiary-test-writer`.
    - **The Test Writer is responsible for unit tests. It does *not* know how to update source code or docs!**
    - By default dispatch **one** Test Writer per Task, assigned all of the Task's testing Subtasks, so it sees the Task's whole test surface and does not duplicate cases, helpers or test matrices across files.
    - Split into more than one Test Writer only when the test work divides into genuinely independent areas: disjoint test files, no overlapping behaviour under test, and no fixture or helper that more than one lane creates or changes. Give each Test Writer a lane of Subtasks, keep the number of lanes small, and name the other lanes' scope in each dispatch prompt so lanes do not duplicate each other. If a lane creates or changes a fixture another lane uses, dispatch that lane first.
    - To decide whether to split, read the planner's note on independent groups and shared fixtures or helpers in the Context section of the testing Subtasks. If the note is missing or unclear, dispatch one Test Writer.
    - When you split, keep the final "run the full unit test suite and fix failures" Subtask out of the lanes. Once all lanes finish, dispatch a single Test Writer for it, with the lanes' reports threaded into its prompt.
    - Trade-off: fewer Test Writers make a Task more serial, but create far less duplication for the reviews to clean up.
- If docs need to be modified or created, dispatch `apiary-doc-writer` for **one** initial pass per Task, after code and tests have settled (see Sequencing across ticks). After that it is dispatched only for the batched doc pass of each review round (and the final-review fix-up in step 5), never to re-sync docs after each code or test change.
- If this is the initial dispatch for a Task, **always** dispatch `apiary-product-manager`.
  - If this is a re-work dispatch after reviewer feedback you may **optionally** choose to not dispatch the Product Manager, if the work is minor enough and will not impact Product functionality

Parallelism within a Task is the exception: the Engineer, then the Test Writer(s), then the Doc Writer, with parallel Test Writers only for genuinely independent areas.

#### Sequencing across ticks

Coordination between roles is sequencing you own, spread across reconciliation ticks:
- On the Engineer's completion tick, dispatch the Test Writer(s) with the Engineer's report threaded into their dispatch prompts — they review the Engineer's work as part of their instructions.
- Dispatch the Doc Writer's initial pass after code and tests have settled: on the completion tick of the last Test Writer (or of the Engineer, when the Task has no test work), with the Engineer and Test Writer reports threaded into its dispatch prompt.
- Dispatch the Product Manager sequentially with whatever prior reports you want it to review — commonly at the end for a full review of the Task, but nothing prevents interim PM dispatches (e.g., a PM check on the Engineer's design before a Test Writer starts writing against it).
- One hard constraint: the PM must not run in parallel with a writer whose output it is supposed to review — concurrent subagents cannot see each other's in-flight work.
- Batch re-work within a Task (e.g. after PM findings) in the same order as the initial pass: the Engineer, then the Test Writer(s), then the Doc Writer, each dispatched after the previous role's re-work completes, with the earlier re-work reports threaded into its prompt. Skip any role with no findings and no upstream changes to follow. Make at most **one** Doc Writer dispatch for that review round, carrying all doc-affecting findings batched together. Never dispatch one Doc Writer pass per finding, or a doc pass while that round's code or tests are still changing. Skip the doc pass entirely if nothing doc-affecting changed.

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
If there are more Epics to work on, continue automatically with the next logical one — it must already be hatched (`pupa` or `worker`, never `larva`); execution never triggers hatching. Clear your context window and go back to step 2, which includes the reconcile pass.
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
    - Dispatch any role subagents required to do the work you deem necessary from the reviewer findings, keeping the fix-up lean and in the same order as the initial pass: code findings to the Engineer, then all test findings to a single Test Writer (split only per the independent-areas rule in step 3), then all doc findings batched into a single Doc Writer. Dispatch each role after the previous role's fix-up completes, with the earlier fix-up reports threaded into its prompt; skip any role with no findings and no upstream changes to follow.
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
