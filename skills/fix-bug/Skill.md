---
name: fix-bug
description: Fix a bug described in a Bee ticket
---

## Overview

The user will either call without arguments or with a Bug id
- If called without arguments, find all Bug bees for this repo and present them, asking which one to fix
- If called with a Bee id, find it in the Bugs hive. If its not in the Bugs hive, tell the user and ask them if they want to continue

You will ultimately get the Bug ID you need to work on.

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

### 3. Form Teams to Fix Bug

Analyze the bug, the source code, the tests and the docs. Understand the likely scope of the bug fix.
If the fix requires modifications to the source code, you will need to spawn and Engineer.
If the fix will require modifications to unit tests, you will need to spawn a Test Writer.
If the fix will require modifications to the Readme or Architecture docs, you will need to spawn a Doc Writer.

Use AskUserQuestion to confirm with User the scope of the problem and your proposed fix.
Once user agrees, form the appropriate team.

**IMPORTANT: You must stay in `delegate` mode. Do not take on work, delegate work to Team members.**

The team may consist of any of the following agents:
- Engineer
  - Model: Claude Sonnet
  - Responsibilities:
    - Executing implementation Subtasks for a task (if required)
  - Instructions:
    - Read the Bug description from the Bees server
    - Review any relevant internal architecture docs referenced in CLAUDE.md under "Documentation Locations"
    - Review the existing code to determine the current state
    - Review the engineering best practices guide referenced in CLAUDE.md under "Documentation Locations"
    - Modify any source code required to fix the bug
- Test Writer
  - Model: Claude Sonnet
  - Responsibilities:
    - Executing testing Subtasks for a task (if required)
  - Instructions:
    - Use the test writing guide referenced in CLAUDE.md under "Documentation Locations"
    - Use the test review guide referenced in CLAUDE.md under "Documentation Locations"
    - Review the work of the Engineer and see if any tests need to be added, deleted or updated based on that work
      - Review the work of the Engineer to find any gaps, then add, delete or updated required tests
- Doc Writer
  - Model: Claude Sonnet
  - Responsibilities:
    - Execute documentation Subtasks for a task (if required)
  - Instructions:
    - Use the doc writing guide referenced in CLAUDE.md under "Documentation Locations"
    - Review the customer-facing docs referenced in CLAUDE.md under "Documentation Locations" and see if they need any updates
    - Review the internal architecture docs referenced in CLAUDE.md under "Documentation Locations" and see if they need any updates
    - Review the work of the Engineer and see if any docs need to be updated based on that work
      - Review the work of the Engineer to find any gaps, then update docs
    - Update any docs that require updating


#### 4. Review Loop

Once the Team is done, form a review Team to check their work.
If you invoked the Engineer in the first team, invoke the Code Reviewer in this team.
If you invoked the Test Writer in the first team, invoke the Test Review in this team.
If you invoked the Doc Write in the first team, invoke the Doc Reviewer in this team.

- Code Reviewer
  - Model: Claude Sonnet
  - Responsibilities:
    - Review the output of the Engineer
    - Provide feedback where the work of the Engineer was not up to standards
  - Instructions:
    - Invoke the /code-review skill
- Test Reviewer
  - Model: Claude Sonnet
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
  - If so, **reform the first team*** to do the work
    - **IMPORTANT** Stay in delegate mode and do not do the work yourself.
    - If the feedback was minor enough, you may choose to **NOT** spawn the Product Manager on this iteration 
    - Spawn any team members required to do the work you deem necessary from the reviewer team
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

Use the AskUserQuestion tool to ask the User if they want you to make any more changes, or just continue.


