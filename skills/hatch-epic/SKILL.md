---
name: hatch-epic
description: Break down a single Epic into Tasks. User can provide Epic ID or a Bee ID and skill finds Epics that are ready
---

# Epic to Tasks

Your job is to break down an Epic ticket into Tasks and Subtasks.

## Workflow

### 1. Determine Which Epic to Break Down

**If caller provides Epic ID**: Use that Epic ID directly.

**If caller provides a Bee ID**: Find workable Epics automatically by querying the `bees` MCP server for any
Epic children of that Bee in the `larva` state. These are Epics that are written but whose children (Tasks) have not been written yet.
If there are multiple, use `AskUserQuestion` with `multiSelect: false` to let user pick ONE Epic. Review the 
dependency chain and recommend the one that makes the most sense:
- Question: "Which Epic do you want to hatch?"
- Options:
  - Epic 1 (recommended)
  - Epic 2
  - etc

### 2. Fetch and Analyze Epic

Fetch full Epic details from the Bees server to understand scope of total work.

- Get that Epic from the Bees server and read it.
- Parse Epic title, description, and requirements.
- Read the parent Bee
- Read the egg source material linked in the parent Bee.
- Identify what implementation work is needed as a list of Tasks.
- Find any Epics this Epic depends on (check `up_dependencies` field) and use `show_ticket()` to read them
  - These Epics describe foundational work that will be complete before this Epic you are working on is done
  - So presume that foundational work is done and make a plan to build on top of it
- Check for sibling overlap: 
  - Read ALL sibling Epics under the same Bee. 
  - Before proposing any Task, verify it does not duplicate work scoped to another Epic. 
  - If a Task or Subtasks overlaps with a sibling Epic's scope, do NOT include it — that work belongs to the other Epic.


### 3. Break Epic into Tasks

#### Tasks
- Tasks should be discrete units of work - suitable for a single git commit.
- Do not include code snippets or file numbers. Code is going to change as execution proceeds. Assume the LLM working on the code will be capable of finding the code.
- Do not describe exactly how to implement the solution. The LLM working on the solution will be an expert. Just provide the scope of work and any requirements or acceptance criteria.

Example task:
```
Task 1: Implement CSV export functionality                                                                              
                                                                                                                                         
  Context: Users need to export data to CSV format for analysis in spreadsheet applications. Currently, only JSON export is supported.
                                                                                                                                         
  What Needs to Change:                                                                                                                  
  - Add export_to_csv() function to src/export.py using csv.DictWriter
  - Add CSV format option to export CLI command
  - Update export service to route CSV requests to new function
                                                                                                                                         
  Why: Users frequently request spreadsheet-compatible exports for data analysis and reporting workflows.
                                                                                                                                         
  Success Criteria:                                                                                                                   
  - Users can run export command with --format=csv flag
  - CSV output contains proper headers and quoted fields
  - Exported CSV files open correctly in Excel/Google Sheets
```

### 4. Create Task Team to Break Task into Subtasks

Form a team to write the Subtasks for this Epic. Your responsibilities are:
  - Surface design questions back to the Caller
    - If the team proposes different approaches to a problem, surface this back up to the caller with an AskUserQuestion
  - Responsible for coordinating the team and ensuring all work is complete, but the Product Manager has final authority on quality and completeness
- Instructions:
    - Carrying forward architectural decisions:
      - If the caller provides architectural decisions or constraints (e.g., "make parameter X optional with fallback Y"), explicitly reference it in every affected subtask description. 
      - Do not paraphrase or partially apply — use the caller's exact specification.

#### Team
If source code needs to be changed, include the Engineer. If not, the Engineer optional.
If unit test code need to be changed, include the Test Writer. If not, the Test Writer is optional.
If docs need to be changed, include the Doc Writer. If not, the Doc Writer is optional.
Always spawn the Product Manager.

**IMPORTANT**: You do not break Tasks into Subtasks. This is the job of the Team.

**CRITICAL — Subagent permissions**: Spawn ALL team members with `mode: "plan"`. Team members are read-only researchers. They must never create, update, or delete tickets. Only YOU (the team lead) call `create_ticket`, `update_ticket`, or `delete_ticket`.

When spawning team members, include the following restriction in each teammate's spawn prompt:

```prompt
You are a READ-ONLY researcher. You must NEVER call create_ticket, update_ticket, or delete_ticket.
Your job is to research the codebase and report your proposed subtasks back via SendMessage as text.
Only the team lead creates tickets.
```

Also include the following Subtasks guidance in each teammate's spawn prompt:

```prompt
Subtask represent discrete sets of work required to achieve the Task outcome.

- Do not include code snippets or file numbers. Code is going to change as execution proceeds. Assume the LLM working on the code will be capable of finding the code.
- Do not describe exactly how to implement the solution. The LLM working on the solution will be an expert. Just provide the scope of work and any requirements or acceptance criteria.
Examples include:
- Writing or updated a method
- Changing code to use a new method or method signature
- Updating a document
- Updating a test file

Sample subtask:
title: Modify test_api.py to include required changes
body: Update existing API test coverage to account for CSV export support. 
Ensure tests validate correct format selection, response structure, headers, and error handling without impacting existing JSON export behavior.
acceptance criteria:
- API tests cover successful CSV export responses.
- Tests validate presence of header row and correct row counts.
- Tests confirm JSON export behavior remains unchanged.
- All API tests pass after updates.
```


##### Team Composition

The team should consist of the following agents:

- Engineer
  - Model: Claude Sonnet
  - Responsibilities:
    - Writing implementation Subtasks for a task (if required)
      - Tasks that only involve research (no code or doc changes) may omit all of these subtasks.
  - Instructions:
    - Review any relevant docs at docs/architecture
    - Review the existing code to determine the current state
    - Review the engineering best practices guide at docs/engineering-best-practices.md
    - Write subtasks for each logical implementation step.
    - There may be one or many implementation subtasks
- Test Writer
  - Model: Claude Sonnet
  - Responsibilities:
    - Writing testing Subtasks for a task (if required)
  - Instructions:
    - Use docs/architecture/testing.md for reference
    - Use docs/guides/test_review_guide.md for reference
    - Write or modify any required unit tests
    - Write or modify any required Integration tests
    - Add a subtask **for each test file or logical group of test file** that needs to be modified based on the work described by the Engineer
      - The substask will provide high level instructions to: 
        - Update any tests that cover the work done in the parent Task
        - Delete any tests that are now made obsolete by work done in the parent Task
        - Add any tests to cover functionality that is currently not tested based on the work done in the parent Task
    - Add a final substask to run the full unit test suite and fix any failures. Integration tests will be handled by the calling function.
       - This subtask tells the agent to ensure 100% unit tests passing before completing, this means fixing broken tests
       - If for some reason the agent cannot get 100% unit tests passing it should report the failure to the Team Lead
- Doc Writer
  - Model: Claude Sonnet
  - Responsibilities:
    - Writing documentation Subtasks for a task (if required)
  - Instructions:
    - Use docs/guides/docs_guide.md for guidance
    - Readme:
      - If the Task modifies user-facing code or installation and setup:
        - Review the Readme
        - Write a subtask describing how the Readme should be updated based on the work done in this Task
    - Architecture Docs:
      - If the Task modifies source code:
        - Review the architecture docs at /docs/architecture
        - Review the development guides at /docs/guides
        - Write a subtask for each architecture doc that needs to be updated based on the work done in this Task
- Product Manager
  - Model: Claude Opus
  - Responsibilities:
    - Responsible for reviewing Tasks against the PRD and SRD
    - Ensures that the work being described meets the requirements
  - Instructions:
    - Read any source documents provided in the top level Bee
    - Review the Task and Subtasks to ensure that the work proposed: 
      - Aligns with the requirements
      - Does not introduce more functionality than asked for
        - e.g The PRD calls for no legacy support but the Engineers proposes a task for backwards compatibility.
        - Call this out as unacceptable
      - Review all Tasks once they are complete against the Epic to ensure that:
        - The work will meet the Acceptance Criteria
        - The work covers all functionality required by the Epic
        - The work does not introduce any functionality not required or explicitly disallowed in the Epic
    - Review the subtasks created by the Test Writer
      - Ensure they have done their best to create a subtask per test file that needs to be changed



#### Mandatory Subtask Description Template

Every subtask description MUST include all of the following sections. Do not omit any section. Do not use abbreviated or one-line descriptions.

```
## Context
Why this subtask exists and what preconditions are assumed.

## What Needs to Change
Specific files, functions, and changes required. Include line numbers where known.

## Key Files
- path/to/file.py — what changes here

## Acceptance Criteria
- Observable, testable conditions that confirm the subtask is complete
- Be specific: "function X returns Y" not "function works correctly"
```

#### Task Loop
Spawn one persistent team to handle all Tasks. Work through each Task sequentially with the same team, planning subtasks one Task at a time, **without asking the User for permission**. 
Only stop to review with the User once all Tasks are done.

### 5. Review Epic 

When all Tasks are complete,   
- Review quality of Task and Subtasks, make final decision when to present completed Task to caller
- You must defer to the Product Manager on whether a Task is final and complete

After Epic is complete, then create Tasks.
Each Task should be a Child of the Epic it is for (and the Epic should be marked as Parent).
If Tasks must be completed sequentially, add up and down dependencies to relevant tickets.

#### Set Status
- Set the Epic to `pupa` (it is now written and its children — the Tasks — are written)
- Set each Task to `pupa` (it is written and its children — the Subtasks — are written)
- Set each Subtask to `pupa` (it is written and has no children)

Show the Tasks you just created to the User in detail and ask them if they want to make modifications.


#### Checklist Before Returning

- [ ] All Subtasks have parent set to task-id
- [ ] If Task modifies code, all mandatory subtasks created (implementation steps, architecture docs review, unit test review, run full test suite)
- [ ] Documentation subtasks have up_dependencies on implementation (implementation must complete first)
- [ ] Testing subtasks have up_dependencies on implementation/add-tests (implementation and test creation must complete first)
- [ ] All descriptions follow the mandatory template (see below)
- [ ] NO git commit subtasks created (commits handled automatically by executors)
- [ ] Testing subtasks support maximum parallelization on execution by making one subtask per test file to be modified

