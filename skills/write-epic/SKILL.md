---
name: write-epic
description: Decompose a single Epic into Tasks and Subtasks. Caller can provide an Epic ID, or a Feature/Plan ID — the skill finds Epics in `draft` whose Tasks have not been written.
---

# Epic to Tasks and Subtasks

Your job is to decompose an Epic ticket into Tasks and Subtasks, recruit
a research team to flesh out the Subtasks, and write the result back to
the ticket backend.

## Status Ownership

This skill OWNS these transitions, and only these:

- Epic → `ready` (when all Tasks and Subtasks under it are written)
- Task → `ready` (Tasks are created in `ready`, or set to `ready` after
  creation, idempotently)
- Subtask → `ready` (Subtasks are created in `ready`, or set to `ready`
  after creation, idempotently)

This skill does NOT own and MUST NOT set:

- Epic → `active` or `done`
- Task → `active` or `done`
- Subtask → `active` or `done`
- Any Plan status transition (owned by the Plan-writing skill and the
  Feature-execution skill)
- Any Feature status transition (owned by Feature-level skills)

All status writes are idempotent — setting `ready` on something already
`ready` is a no-op.

## Workflow

### 0. Read `apiary.md`

Read `<project_root>/apiary.md` first. It supplies:

- **Documentation Locations** — `Reference` (engineering best practices,
  test guide, doc writing guide) and `Maintained` (contributor docs,
  customer-facing docs). These paths are passed through to subagents in
  their spawn prompts. Categories listed as `omitted` are skipped
  gracefully when referenced downstream.
- **`## New Feature` section** — declares the source-reference resolver
  used to fetch the Feature ticket's source content (e.g.,
  `github resolver`). This is needed when traversing the Plan → Feature
  chain in step 2.

If `apiary.md` is missing, emit a one-line notice ("apiary.md not found;
proceeding without project context") and continue. Documentation
Locations passed to subagents will be empty; subagents must still skip
any unspecified category gracefully. Source-reference resolution in
step 2 will fall back to ticket-body-only context.

### 1. Determine Which Epic to Break Down

**If caller provides an Epic ID**: Use that Epic ID directly.

**If caller provides a Feature ID or Plan ID**: Find decomposable Epics
automatically by listing Epic-tier children in the `draft` state under
that Feature/Plan. These are Epics that are written but whose children
(Tasks) have not been written yet.

If there are multiple, ask the user to pick ONE Epic. Review the
dependency chain and recommend the one that makes the most sense:

- Question: "Which Epic do you want to write?"
- Options:
  - Epic 1 (recommended)
  - Epic 2
  - etc

If the caller wants batch processing, work through Epics in dependency
order, one at a time, repeating steps 2–5 for each.

### 2. Fetch and Analyze Context

Pull full context from the ticket backend so you can decompose
intelligently.

- **Read the Epic.** Parse title, description, and requirements.
- **Read the parent Plan ticket** (the Epic's parent). The Plan frames
  the broader effort and its source reference points back at the
  Feature.
- **Follow the Plan's source reference to the Feature ticket.** The
  Plan's source reference IS the Feature ticket ID (set by the
  Plan-writing skill). Resolve it through the ticket backend to fetch
  the Feature ticket itself.
- **Read all t1 Doc children of the Feature ticket** — typically a PRD
  and an SRD, plus any mockups or supplementary docs. These supply the
  product- and software-level requirements that the Product Manager
  subagent will check Tasks against.
- **Read up-dependency Epics** (check the `up_dependencies` field on
  this Epic). These Epics describe foundational work that will be
  complete before this Epic is worked on. Presume that work is done and
  plan to build on top of it.
- **Check sibling overlap.** Read all sibling Epics under the same
  Plan. Before proposing any Task, verify it does not duplicate work
  scoped to another Epic. If a Task or Subtask overlaps with a sibling
  Epic's scope, do NOT include it — that work belongs to the other
  Epic.

If the source-reference resolver is not configured in `apiary.md`, the
Feature ticket can't be fetched, or any t1 Doc child fails to load,
emit a one-line notice ("Plan → Feature chain not fully resolved;
proceeding with available context") and continue with whatever context
did load. Pass whatever you have to the Product Manager subagent in
step 4.

### 3. Break Epic into Tasks

#### Tasks
- Tasks should be discrete units of work — suitable for a single git
  commit.
- Do not include code snippets or file numbers. Code will change as
  execution proceeds. Assume the LLM working on the code will be
  capable of finding the code.
- Do not describe exactly how to implement the solution. The LLM
  working on the solution will be an expert. Just provide the scope of
  work and any requirements or acceptance criteria.

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

### 4. Dispatch Subagents to Break Tasks into Subtasks

Dispatch role-specific subagents via the Agent tool to research and
propose the Subtasks for this Epic. Your responsibilities are:

- Surface design questions back to the caller.
  - If subagents propose different approaches to a problem, surface
    this back up to the caller and ask the user to decide.
- Coordinate the subagents and ensure all work is complete. The
  Product Manager has final authority on quality and completeness.
- Carry forward architectural decisions:
  - If the caller provides architectural decisions or constraints
    (e.g., "make parameter X optional with fallback Y"), explicitly
    reference it in every affected Subtask description.
  - Do not paraphrase or partially apply — use the caller's exact
    specification.

#### Subagent Composition

If source code needs to be changed, dispatch the Engineer
(`subagent_type: engineer`). Otherwise the Engineer is optional.
If unit test code needs to be changed, dispatch the Test Writer
(`subagent_type: test-writer`). Otherwise the Test Writer is optional.
If docs need to be changed, dispatch the Doc Writer
(`subagent_type: doc-writer`). Otherwise the Doc Writer is optional.
Always dispatch the Product Manager (`subagent_type: pm`).

**IMPORTANT**: You do not break Tasks into Subtasks yourself. That is
the job of the dispatched subagents.

#### CRITICAL — Subagent permissions

For the purposes of this skill, every dispatched subagent is a
**read-only researcher**. Subagents MUST NEVER create, update, or
delete tickets. Each subagent returns its proposed Subtask titles and
bodies as text in the Agent tool's response. Only this skill (the
calling skill) writes tickets.

Dispatch each role via the Agent tool with the appropriate
`subagent_type` literal — `engineer`, `test-writer`, `doc-writer`, or
`pm`. Do not use any Agent-Teams API for role dispatch in this skill.

Include the following restriction at the top of every dispatch prompt:

```prompt
You are a READ-ONLY researcher for the write-epic skill. You must NEVER
create, update, or delete tickets. Do not call any bees ticket-mutation
commands. Your job is to research the codebase and the supplied context,
then report your proposed Subtasks back as the text body of your final
response (the Agent tool's tool-result). Only the calling skill writes
tickets.
```

Include the following Subtasks guidance in every dispatch prompt:

```prompt
Subtasks represent discrete sets of work required to achieve the parent Task's outcome.

- Do not include code snippets or file numbers. Code will change as execution proceeds. Assume the LLM working on the code will be capable of finding the code.
- Do not describe exactly how to implement the solution. The LLM working on the solution will be an expert. Just provide the scope of work and any requirements or acceptance criteria.

Examples include:
- Writing or updating a method
- Changing code to use a new method or method signature
- Updating a document
- Updating a test file

Sample Subtask:
title: Modify test_api.py to include required changes
body: Update existing API test coverage to account for CSV export support.
Ensure tests validate correct format selection, response structure, headers, and error handling without impacting existing JSON export behavior.
acceptance criteria:
- API tests cover successful CSV export responses.
- Tests validate presence of header row and correct row counts.
- Tests confirm JSON export behavior remains unchanged.
- All API tests pass after updates.
```

Each subagent's response text is the source of truth for its proposed
Subtasks; this skill is responsible for taking those proposals and
writing them to the ticket backend in step 5.

##### Roles

- **Engineer** (`subagent_type: engineer`)
  - Model: Claude Sonnet
  - Responsibilities:
    - Proposing implementation Subtasks for a Task (if required).
      Tasks that only involve research (no code or doc changes) may
      omit all of these Subtasks.
  - Instructions (embed in dispatch prompt):
    - Review any relevant internal architecture docs referenced in
      `apiary.md` under "Documentation Locations" (Maintained →
      contributor docs). Skip categories marked `omitted`.
    - Review the existing code to determine the current state.
    - Review the engineering best practices guide referenced in
      `apiary.md` under "Documentation Locations" (Reference →
      engineering best practices). Skip if `omitted`.
    - Propose Subtasks for each logical implementation step. There may
      be one or many implementation Subtasks. Return them as text.
- **Test Writer** (`subagent_type: test-writer`)
  - Model: Claude Sonnet
  - Responsibilities:
    - Proposing testing Subtasks for a Task (if required).
  - Instructions (embed in dispatch prompt):
    - Use the test writing guide referenced in `apiary.md` under
      "Documentation Locations" (Reference → test guide). Skip if
      `omitted`.
    - Use the test review guide referenced in `apiary.md` under
      "Documentation Locations" (Reference → test guide). Skip if
      `omitted`.
    - Propose Subtasks covering any required unit tests.
    - Propose Subtasks covering any required integration tests.
    - Propose a Subtask **for each test file or logical group of test
      files** that needs to be modified based on the work described by
      the Engineer. The Subtask provides high-level instructions to:
      - Update any tests that cover the work done in the parent Task.
      - Delete any tests that are now made obsolete by work done in
        the parent Task.
      - Add any tests to cover functionality that is currently not
        tested based on the work done in the parent Task.
    - Propose a final Subtask to run the full unit test suite and fix
      any failures. Integration tests are handled by the calling
      skill.
      - This Subtask tells the agent to ensure 100% unit tests passing
        before completing — i.e., fix broken tests.
      - If the agent cannot get 100% unit tests passing, it should
        report the failure to the calling skill.
- **Doc Writer** (`subagent_type: doc-writer`)
  - Model: Claude Sonnet
  - Responsibilities:
    - Proposing documentation Subtasks for a Task (if required).
  - Instructions (embed in dispatch prompt):
    - Use the doc writing guide referenced in `apiary.md` under
      "Documentation Locations" (Reference → doc writing guide). Skip
      if `omitted`.
    - Customer-facing docs:
      - If the Task modifies user-facing code or installation/setup:
        - Review the customer-facing docs referenced in `apiary.md`
          under "Documentation Locations" (Maintained → customer-facing
          docs). Skip if `omitted`.
        - Propose a Subtask describing how the customer-facing docs
          should be updated based on the work done in this Task.
    - Architecture docs:
      - If the Task modifies source code:
        - Review the internal architecture docs referenced in
          `apiary.md` under "Documentation Locations" (Maintained →
          contributor docs). Skip if `omitted`.
        - Propose a Subtask for each architecture doc that needs to be
          updated based on the work done in this Task.
- **Product Manager** (`subagent_type: pm`)
  - Model: Claude Opus
  - Responsibilities:
    - Reviewing proposed Tasks against the PRD and SRD attached to the
      parent Feature ticket.
    - Ensuring that the work being described meets the requirements.
  - Instructions (embed in dispatch prompt):
    - This skill will provide the PRD and SRD content (and any other
      Feature-level docs) fetched in step 2 via the Plan → Feature
      source-reference chain. Read them. Do NOT assume the docs are
      attached to the parent Plan or Epic — they live on the Feature
      ticket and have already been resolved for you.
    - Review each proposed Task and its proposed Subtasks to ensure
      that the work:
      - Aligns with the requirements in the PRD/SRD.
      - Does not introduce more functionality than asked for.
        - e.g., the PRD calls for no legacy support but the Engineer
          proposes a Subtask for backwards compatibility — call this
          out as unacceptable.
      - Once all Tasks are complete, review them against the Epic to
        ensure that:
        - The work will meet the Epic's Acceptance Criteria.
        - The work covers all functionality required by the Epic.
        - The work does not introduce any functionality not required
          or explicitly disallowed in the Epic.
    - Review the Subtasks proposed by the Test Writer.
      - Ensure they have done their best to propose one Subtask per
        test file that needs to be changed.

#### Mandatory Subtask Description Template

Every Subtask description MUST include all of the following sections.
Do not omit any section. Do not use abbreviated or one-line
descriptions.

```
## Context
Why this Subtask exists and what preconditions are assumed.

## What Needs to Change
Specific files, functions, and changes required. Include line numbers where known.

## Key Files
- path/to/file.py — what changes here

## Acceptance Criteria
- Observable, testable conditions that confirm the Subtask is complete
- Be specific: "function X returns Y" not "function works correctly"
```

#### Task Loop
Work through each Task sequentially. For each Task, dispatch the
required role subagents via the Agent tool, collect their proposed
Subtasks from the response text, and run the PM subagent to review the
proposals. Plan Subtasks one Task at a time, **without asking the user
for permission**. Only stop to review with the user once all Tasks are
done.

### 5. Review and Write

When all Tasks are complete:

- Review the quality of the Tasks and Subtasks; make the final
  decision when to present completed Tasks to the caller.
- You must defer to the Product Manager on whether a Task is final
  and complete.

After the Epic decomposition is complete, write the Tasks. Each Task
should be a child of the Epic it belongs to (and the Epic should be
the parent). If Tasks must be completed sequentially, set up and down
dependencies on the relevant tickets.

#### Set Status

Apply these writes when all Tasks and their Subtasks are written.
Every write is idempotent — if the ticket is already at the target
status, the write is a no-op.

- Set the Epic → `ready` (it is now written and its children — the
  Tasks — are written).
- Each Task → `ready` (create at `ready`, or set to `ready` after
  creation; the Task is written and its children — the Subtasks — are
  written).
- Each Subtask → `ready` (create at `ready`, or set to `ready` after
  creation; the Subtask is written and has no children).

##### Status Ownership reminder

This skill MUST NOT, under any circumstance:

- Set the Epic to `active` or `done`.
- Set any Task to `active` or `done`.
- Set any Subtask to `active` or `done`.
- Touch the parent Plan's status — that's owned by the Plan-writing
  skill and the Feature-execution skill.
- Touch the Feature ticket's status — that's owned by Feature-level
  skills.

If you find yourself reaching for a non-`ready` status, stop. That
transition belongs to a downstream skill (e.g., the Epic-execution
skill in a later phase).

Show the Tasks you just wrote to the user in detail and ask whether
they want modifications.

#### Checklist Before Returning

- [ ] All Subtasks have parent set to a Task ID
- [ ] If a Task modifies code, all mandatory Subtasks are written
  (implementation steps, architecture docs review, unit test review,
  run full test suite)
- [ ] Documentation Subtasks have up-dependencies on implementation
  (implementation must complete first)
- [ ] Testing Subtasks have up-dependencies on implementation/add-tests
  (implementation and test creation must complete first)
- [ ] All descriptions follow the mandatory Subtask template
- [ ] NO git commit Subtasks created (commits are handled
  automatically by executors)
- [ ] Testing Subtasks support maximum parallelization on execution by
  making one Subtask per test file to be modified
- [ ] All Tasks and Subtasks are at status `ready`; the Epic is at
  status `ready`; nothing is at `active` or `done`
