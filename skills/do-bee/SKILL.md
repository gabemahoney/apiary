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
3. Forming a Team to complete the work described in the Epic
   3.1. Sending questions and requests for clarification or guidance to the caller
   3.2. Creating one git commit per Task that includes all changes for that Task
4. Looping 2-3 until all Epics are done, then:
   4.1. Disbanding the execution Team
   4.2. Forming a new Review Team
   4.3. Addressing issues found by the Review Team
   4.4. Getting User approval
   4.5. Marking Bee and all child tickets as closed
   4.6. Outputting a final summary

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

### 3. Form Team to Execute Tasks

Before forming the Team, load all Tasks and Subtasks for the Epic:
- Use `show_ticket()` on the Epic to get the `children` array (Task IDs)
- For each Task, fetch its full details including its own `children` array (Subtasks)
- Read every Subtask — these contain the detailed instructions (Context, What Needs to Change, Key Files, Acceptance Criteria) that the team must follow
- Sort Tasks in dependency order (check each Task's `up_dependencies`) to ensure no Task is blocked when executed
- Verify at least 1 Task exists with at least 1 Subtask, and all are ready for work: `status!=larva`
- Mark the current Task with `status=worker` to show work has started
- Mark the Bee with `status=worker` to show work has started (if not already set)

Create an Agent Team to work on an individual Task.
**IMPORTANT: You must stay in `delegate` mode. Do not take on work, delegate work to Team members.**

Choose which Team members are required.
- If source code is being modified or created, spawn the Engineer.
  - **The Engineer is responsible for source code. It does *not* know how to update unit tests or docs!**
- If test code is being modified or created, spawn one or more Test Writers.
    - **The Test Writer is responsible for unit tests. It does *not* know how to update source doe or docs!**
    - If there are multiple files that need updating, the Task should have one Subtask per file
    - If so, spawn multiple Test Writers to work on each Subtask and File in parallel
    - It might be necessary to order them so that the first one creates or modifies any shared fixtures
- If docs need to be modified or created, spawn the Doc Writer.
- If this is the first time forming the team, **always** spawn the Product Manager.
  - If you are re-forming the team to address Code, Doc and Test Reviewer feedback you may **optionally** choose to not spawn the Product Manager, if the work is minor enough and will not impact Product functionality 

The team may consist of any of the following agents, but the Product Manager must always be spawned:
- Engineer
  - Model: Claude Sonnet
  - Responsibilities:
    - Executing implementation Subtasks for a task (if required)
      - Tasks that only involve research (no code or doc changes) may omit all of these subtasks.
  - Instructions:
    - Read the Subtask description from the Bees server — it contains Context, What Needs to Change, Key Files, and Acceptance Criteria
    - Review any relevant internal architecture docs referenced in CLAUDE.md under "Documentation Locations"
    - Review the existing code to determine the current state
    - Review the engineering best practices guide referenced in CLAUDE.md under "Documentation Locations"
    - Execute each implementation Subtask following the instructions in its description
    - There may be one or many implementation subtasks
    - Mark each Subtask as `status=worker` when starting it and `status=finished` when done
- Test Writer
  - Model: Claude Sonnet
  - Responsibilities:
    - Executing testing Subtasks for a task (if required)
      - Tasks that only involve research (no code or doc changes) may omit all of these subtasks.
  - Instructions:
    - Use the test writing guide referenced in CLAUDE.md under "Documentation Locations"
    - Use the test review guide referenced in CLAUDE.md under "Documentation Locations"
    - Execute all test subtasks to change, add or delete tests
    - Review the work of the Engineer and see if any tests need to be added, deleted or updated based on that work
      - It is possible the testing subtasks were incomplete
      - Review the work of the Engineer to find any gaps, then add, delete or updated required tests
    - Mark each Subtask as `status=worker` when starting it and `status=finished` when done
- Doc Writer
  - Model: Claude Sonnet
  - Responsibilities:
    - Execute documentation Subtasks for a task (if required)
      - Tasks that only involve research (no code or doc changes) may omit all of these subtasks.
  - Instructions:
    - Use the doc writing guide referenced in CLAUDE.md under "Documentation Locations"
    - Execute any customer-facing docs subtasks
    - Execute any internal architecture docs subtasks
    - Review the work of the Engineer and see if any docs need to be updated based on that work
      - It is possible the doc subtasks were incomplete
      - Review the work of the Engineer to find any gaps, then update docs
    - Mark each Subtask as `status=worker` when starting it and `status=finished` when done
- Product Manager
  - Model: Claude Sonnet
  - Responsibilities:
    - Responsible for reviewing Task work against the PRD, SRD and Grandparent Bee
    - Ensures that the work that was done meets the requirements
    - Surface design questions back to the Caller
      - If the team proposes different approaches to a problem, surface this back up to the caller with an AskUserQuestion
    - Responsible for providing report to share back up to calling Agent
    - Ultimately responsible for the quality of the Task work and correctness of the output of the Team
  - Instructions:
    - Get the Task from the Bees server and read it.
    - Read all Subtasks (children of the Task) — these contain the detailed work instructions.
    - Read the Parent Epic.
    - Read the Grandparent Bee.
    - Read the source material linked in the Grandparent Bee.
    - Make sure the Test Writer and Doc Writer review the work of the Engineer
      - The Engineer's output needs review by the rest of the team
    - Review quality of Task and Subtasks efforts, make final decision when to present completed Task to caller
    - Review the Task and Subtasks execution to ensure that the work: 
      - Aligns with the requirements
      - Does not introduce more functionality than asked for
        - e.g The PRD calls for no legacy support but the Engineers proposes a task for backwards compatibility.
        - Call this out as unacceptable
      - Review all Tasks once they are complete against the Epic to ensure that:
        - The work will meet the Acceptance Criteria
        - The work covers all functionality required by the Epic
        - The work does not introduce any functionality not required or explicitly disallowed in the Epic
    - Uses the code-review and doc-review skill after work has been done for quality control
      - NOTE: These skills could infinitely return work items
      - Product Manager must use judgement when deciding whether to ask the Team to make the improvements or not
      - If the Product Manager decides to ignore code-review or doc-review feedback, this MUST be included in the end of task summary report for review
    - Provide report when done. Must include:
      - Any ignored reviewer feedback
      - Any contentious topics between team members
      - Any design decisions that were made that conflicted with work described in tickets
      - Any incomplete work


#### 4.1 After Each Task

When a Task and all its Subtasks are done (all reviewer feedback addressed or ignored):

1. Create one git commit for the Task. Use system or project defined guidance on git usage. **NEVER push to remote — committing only.**
2. Mark the Task as `status=finished` (Subtasks were marked finished by each agent as they completed their work).
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

Once all Epics in the Bee are done:
- delete the Team 
- form a new review Team to check their work.

If you invoked the Engineer in the first team, invoke the Code Reviewer in this team.
If you invoked the Test Writer in the first team, invoke the Test Review in this team.
If you invoked the Doc Writer in the first team, invoke the Doc Reviewer in this team.

- Code Reviewer
  - Model: Claude Opus
  - Responsibilities:
    - Review the output of the Engineer
    - Provide feedback where the work of the Engineer was not up to standards
  - Instructions:
    - Invoke the /code-review skill
- Test Reviewer
  - Model: Claude Opus
  - Responsibilities:
    - Review the output of the Test Writer
    - Provide feedback where the work of the Test Writer was not up to standards
  - Instructions:
    - Invoke the /test-review skill
- Doc Reviewer
  - Model: Claude Sonnet
  - Responsibilities:
    - Review the output of the Doc Writer
    - Provide feedback where the work of the Doc Writer was not up to standards
  - Instructions:
    - Invoke the /doc-review skill
- Get the feedback, and make a judgement call about whether that work must be done
  - If so, **reform or re-use the first team** to do the work
    - **IMPORTANT** Stay in delegate mode and do not do the work yourself.
    - If the feedback was minor enough, you may choose to **NOT** spawn the Product Manager on this iteration 
    - Spawn any team members required to do the work you deem necessary from the reviewer team
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