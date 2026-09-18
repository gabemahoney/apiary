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
- Read the reference_materials linked in the parent Bee.
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

### 4. Dispatch Planner Subagents to Break Task into Subtasks

You are the **calling agent**. Planner role players are **subagents** — named agent definitions installed in `.claude/agents/`, spawned via the Agent tool: `Agent(subagent_type: "<agent-name>", prompt: <task-specific dispatch prompt>)`. Your responsibilities are:
  - Surface design questions back to the Caller
    - If the planner subagents propose different approaches to a problem, surface this back up to the caller with an AskUserQuestion
  - Responsible for coordinating the planner subagents and ensuring all work is complete, but the Product Manager has final authority on quality and completeness
- Instructions:
    - Carrying forward architectural decisions:
      - If the caller provides architectural decisions or constraints (e.g., "make parameter X optional with fallback Y"), explicitly reference it in every affected subtask description. 
      - Do not paraphrase or partially apply — use the caller's exact specification.

Rules that apply to every dispatch in this skill:

- **Pre-dispatch validation**: before dispatching any subagent, read the target agent definition file from the installed `.claude/agents/` directory and confirm the frontmatter's model field is present and not the placeholder value. If it is missing or still the placeholder, abort with: "Cannot dispatch <agent-name>: model is not configured. Run /apiary-setup to select a model for each role."
- **Background dispatch**: if the Agent tool schema accepts `run_in_background`, pass `run_in_background: true`; otherwise omit it — background is the harness default under Claude Code fork-subagents mode. Every dispatch in this skill is a background dispatch.
- **Cold start**: subagents are never named, reused, or messaged mid-flight. Every dispatch is a fresh spawn with a self-contained prompt. Dispatch prompts name the relevant ticket IDs; planner subagents read the tickets from the bees CLI themselves.
- **Hub-and-spoke**: subagents never talk to each other. Each planner returns its proposals in its report; you serialize dependent roles and thread each report into the next role's dispatch prompt.
- **Reconciliation loop**: drive the work event-driven, not as one big blocking turn. Each tick: (1) observe current state — ticket store, in-flight Agent status; (2) reconcile — dispatch whatever fresh Agent invocations the delta requires; (3) yield the turn. The harness fires a completion notification when each background Agent finishes; that notification triggers the next tick. No polling, no sleep loops, no blocking waits inside a turn.
- **Compaction recovery**: when a compaction/summarization marker is visible in your conversation, everything before it is non-authoritative. Before dispatching the next tick's work after a compaction, re-read the authoritative sources in full — the bees CLI for ticket state, git for code state, and the filesystem for any state files. Conversation memory is never a substitute for these sources.

#### Roles
If source code needs to be changed, include `apiary-engineer-planner`. If not, the Engineer is optional.
If unit test code need to be changed, include `apiary-test-writer-planner`. If not, the Test Writer is optional.
If docs need to be changed, include `apiary-doc-writer-planner`. If not, the Doc Writer is optional.
Always dispatch `apiary-product-manager-planner`.

**IMPORTANT**: You do not break Tasks into Subtasks. This is the job of the planner subagents.

**CRITICAL — Subagent permissions**: Planner subagents are read-only researchers: their agent definitions grant no `Edit`/`Write` tools, and their prompts prohibit ticket mutations. They must never create, update, or delete tickets. Only YOU (the calling agent) call `create_ticket`, `update_ticket`, or `delete_ticket`.

When dispatching planner subagents, include the following restriction in each dispatch prompt:

```prompt
You are a READ-ONLY researcher. You must NEVER call create_ticket, update_ticket, or delete_ticket.
Your job is to research the codebase and return your proposed subtasks as text in your report.
Only the calling agent creates tickets.
```

Also include the following Subtasks guidance in each dispatch prompt:

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

Each planner's Responsibilities and Instructions live in its agent definition (`.claude/agents/apiary-*-planner.md`).

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
Spawn fresh planner subagents per Task via background dispatch, threading prior-Task reports into subsequent dispatch prompts. Work through each Task sequentially, planning subtasks one Task at a time, **without asking the User for permission** — reconcile each Task's completion notifications before dispatching the next Task's planners. Within a single Task you may dispatch multiple planners in parallel (Engineer/Test Writer/Doc Writer research on disjoint parts before the Product Manager's synthesis).
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
