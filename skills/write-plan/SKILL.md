---
name: write-plan
description: Read documents from a Feature ticket and create a Plan in the Plans store, decomposed into Epics. Auto-chains into write-epic for each Epic.
disable-model-invocation: false
---

# Feature Docs to Plan

This skill operates against a **Feature ticket** in the Features store. The Features
store is prescriptive: every t1 child of a Feature ticket is a document (PRD, SRD, or
any other doc type the project uses). This skill reads **all** t1 children of the
given Feature, creates a Plan ticket in the Plans store, decomposes it into Epics, and
then auto-chains into `write-epic` for each Epic.

The skill stays backend-agnostic — speak generically about "tickets" and "stores";
the LLM picks whichever ticket-backend tools are available at runtime.

# Inputs

- A **Feature ticket ID**. If the user did not provide one, list the Features store
  and ask them to pick.

# Status Ownership

This skill explicitly OWNS the following status transitions:

- Plan ticket: `→ draft` (on creation)
- Plan ticket: `→ ready` (after all Epics are created)
- Epic ticket: `→ draft` (on Epic creation)

This skill explicitly does NOT set:

- Plan ticket → `active` or `done` (those belong to `develop-feature` / `develop-epic`)
- Epic ticket → `ready`, `active`, or `done` (those belong to `write-epic`,
  `develop-epic`, etc.)
- Any Task or Subtask status (those belong to `write-epic`, `develop-epic`, and
  subagents).

All status writes are **idempotent**: if a ticket already carries the target status,
do not re-write it. Re-running this skill on a partially-built Plan is a valid resume
flow.

The Plan status flow this skill drives is `draft → ready`. Downstream, Plan moves
`ready → active → done` under other skills' ownership.

# Workflow

## Step 0: Read `apiary.md`

Before anything else, read `<project_root>/apiary.md` from the project root (the
current working directory, unless the user has indicated otherwise).

Extract:

- **Documentation Locations** — for general project context (optional; pass through
  to downstream skills if useful).
- **`## New Feature` section** — note whether it declares a source-reference
  resolver (e.g., `github resolver`). This is used later in Step 3 if the Feature
  ticket has a source reference recorded.

If `apiary.md` is missing, emit a one-line notice ("apiary.md not found; proceeding
without project context") and continue. The skill remains functional without it.

## Step 1: Setup — confirm the Plans store exists

The Plans store must exist before a Plan ticket can be created. Query the ticket
backend for a store named `Plans`.

- If it exists, continue.
- If it does not exist, **stop** with a clear message telling the user to run
  `/project-setup` to configure stores. This skill does NOT create stores.

The Plans store is prescriptive. It has these child tiers:

- t1 — Epic / Epics
- t2 — Task / Tasks
- t3 — Subtask / Subtasks

## Step 2: Hard-gate — Feature must be `ready`

Identify the parent Feature ticket (from caller-supplied ID, or by asking the user
to pick from the Features store).

Read the Feature ticket's status:

- **`draft`** — **Hard refuse.** This is a hard-gate. Output a clear remediation
  message:

  > The Feature ticket `<id>` is in `draft`. Plans cannot be written against a Feature
  > whose docs are not yet reviewed. Run `/req-review <feature-id>` to drive the docs
  > and the Feature to `ready`, or manually mark the Feature `ready` if you want to
  > skip review.

  Then **stop** the skill cleanly. Do not create anything.

- **`ready`** — proceed normally (this is the standard start case).
- **`active`** — proceed (resume case; the Feature has already been promoted past
  `ready` and a Plan may already exist or be partially built).
- **`done`** — proceed (resume case; treat as already-past-ready).
- **Any other value** — refuse with a one-line explanation that the Feature is in an
  unrecognized status and stop.

## Step 3: Read the Feature's t1 docs

Fetch **all t1 children** of the Feature ticket. The Features store is prescriptive
— every t1 child of a Feature is a document. Read each one in full.

These docs (PRD, SRD, mockups, any other doc type the project uses) are the
**primary input** for planning. Do not hardcode "PRD and SRD only" — load whatever
t1 docs the Feature has.

For each doc:

- Extract features, requirements, and acceptance criteria.
- Note non-negotiable requirements (security, latency, availability), architectural
  boundaries, and external dependencies.

After reading the docs, expand beyond what they say: look in the repo and read any
architectural documents (referenced via `apiary.md`'s Documentation Locations, or
discovered manually) to understand design constraints.

Before decomposition, align on the outcome. If you cannot clearly state what changes
for the user or system when the Feature is complete, do not decompose yet. Instead,
surface the gap to the user and stop.

### Resolve source references on the Feature (optional)

If the Feature ticket has a source reference recorded **and** `apiary.md`'s
`## New Feature` section declares a resolver:

- Use the configured resolver to fetch the referenced content (for example,
  `gh issue view <id>` for a `github resolver`). Add the fetched content to the
  working context as additional planning input.
- If the reference is absent, the resolver is not configured, or the fetch fails,
  emit a one-line notice ("source reference not resolved; proceeding with t1 docs
  only") and continue.

## Step 4: Create the Plan ticket

Goal: Create one Plan ticket in the Plans store to track the work.

- **Title** — short, derived from the Feature's title and primary outcome.
- **Body** — a brief summary of the goal and scope (2-3 sentences max).
- **Status** — `draft` (its children, the Epics, have not been written yet). This
  step is idempotent.
- **Source reference** — set the Plan's source reference to point at the **Feature
  ticket ID** (so downstream skills can trace Plan → Feature). Use the source-reference
  mechanism appropriate to the ticket backend in use:

  - For a `bees` backend, set resolver = `bees`, value = `<feature-id>` (e.g.,
    `b.xyz`).
  - For a `beads` backend, set resolver = `beads`, value = `<feature-id>` (e.g.,
    `bd-xxx`).
  - For other backends, use the equivalent built-in cross-store reference resolver.

  Use generic phrasing in any user-facing output: "store the Feature's ticket ID as
  the Plan's source reference". Do **not** describe this back-pointer as an
  up-dependency and do **not** name the storage field directly.

This back-pointer is **load-bearing** — `develop-feature` uses it to find the Plan
from the Feature.

## Step 5: Decompose the Plan into Epics

### Every Epic Must Leave the Codebase Green

Every Epic must leave the codebase in a working state with all existing tests
passing. This is the non-negotiable constraint for all Epics.

### One Epic = One Outcome

- An Epic represents a single, coherent, user- or system-visible capability.
- Avoid Epics organized by system layers (e.g., backend, frontend).
- Prefer Epics defined by observable outcomes.
- An Epic may span multiple systems but must have one measurable success condition.

### Decompose Vertically by Capability

Break the Plan into Epics that deliver end-to-end behavior.

Avoid technology layer Epics:

- Database Epic
- API Epic
- UI Epic
- Documentation Epic
- Testing Epic

Prefer capability slices:

- Epic: User performs action and receives feedback
- Epic: System handles error and retry behavior
- Epic: Metrics and logging are emitted

Each Epic should be independently testable and demo-able.

#### Exception: Technical Refactors

For pure infrastructure or refactor work, strict vertical slicing may not apply.
Pure-tech Epics are allowed provided they leave the codebase green (see above).

- **Go vertical as soon as possible.** After foundational Epics, each subsequent
  Epic should add a demonstrable capability. Bundle infrastructure each slice needs
  into that slice rather than separating into layer Epics.

**Anti-Patterns to Detect:**

- Epic chain where intermediate states are untestable
- Mixing pervasive refactor with feature work in one Epic

### Granularity

Make Epics as granular as possible while adhering to the above constraints of one
outcome and vertical decomposition. It is OK to have a lot of Epics as long as:

- logical outcomes and acceptance criteria are still contained in one Epic
- Epics still represent a vertical slice of end-to-end behavior (unless the
  technical refactor exception applies)

Imagine that we will celebrate the completion of each new Epic with a birthday
party — it is OK to have a lot.

### Acceptance Criteria

Provide clear actionable Acceptance Criteria that the user can use to objectively
evaluate success.

Is there some artifact the user can interact with to test the Epic? If so, detail
the steps they will take to do so. If not, explain how the agent itself can
demonstrate that the Epic was completed successfully. The Acceptance Criteria
should be a detailed description of what a "sprint demo" of the Epic would entail.

Good examples:

- Server starts on http://localhost:8000
  - Good because it explains how the user can validate
- Agent builds unit tests that validate the API endpoints respond to HTTP requests
  - Good because it explains how the agent will demonstrate success

Bad examples:

- Server is available for use
  - Bad because it does not explain how the user can validate
- API endpoints respond to HTTP requests
  - Bad because the user cannot validate themselves, and does not explain how the
    agent itself will demonstrate success

### Present All Epics for User Review

When all Epics are designed, present them to the user for final review.

- Output as markdown: title, description, dependencies for each Epic.
- **Use `AskUserQuestion`** to ask: proceed with creation, modify Epics, or cancel.
- **Wait for approval.** Allow modifications if requested.

## Step 6: Create Epic tickets and wire dependencies

### Create Epics

Create t1 Epic children under the Plan ticket with status `draft` (their children,
the Tasks, have not been written yet).

**NOTE**: If the Plan is small, there may only be one Epic. You do not need to make
multiple.

#### Epic Viability Checklist

- [ ] No testing Epic — testing is folded into the Epics where the work is done.
- [ ] No documentation Epic — documentation is folded into the Epics where the work
      is done.

### Set up dependencies

After all Epics are created, analyze and set up the Epic dependency graph. Use
the `up_dependencies` and `down_dependencies` fields on each Epic ticket to wire
the graph (these are the only legitimate uses of the dependency fields here — they
are NOT used to back-point at the Feature; that is the Plan's source reference).

For each Epic, ask: "What must be completed before this Epic can be worked on?"
Then set the up-dependency / down-dependency relationships across the Epic graph
accordingly.

Common dependency patterns:

- **Infrastructure blocks features**: backend API must exist before frontend or
  features can use it.
- **Foundation blocks UI**: data models / services block UI components that display
  them.
- **Data input blocks processing**: upload / import features block features that
  process that data.
- **Auth blocks protected features**: authentication blocks features requiring
  authorization.

## Step 7: Promote Plan to `ready`

Once all Epics are created and the dependency graph is wired, set the Plan ticket's
status to `ready` (idempotent — if it is already `ready`, no-op).

Each Epic remains in `draft` at this point. This skill MUST NOT set Epics to
`ready`, `active`, or `done`. This skill MUST NOT set the Plan to `active` or
`done`. Those transitions belong to downstream skills.

## Step 8: Report

Output a markdown summary:

- Plan ticket: ID, title, status (`ready`).
- Epics created: each with ID, title, status (`draft`), dependencies (if any).
- Dependency relationships created.

## Step 9: Auto-chain into `write-epic`

After the Plan is `ready`, walk the Epic dependency graph in topological order and
invoke `/write-epic <epic-id>` for each Epic.

- **Do NOT ask the user for permission.** The user ran one command and expects a
  fully decomposed Plan; auto-chain through every Epic.
- Continue until all Epics have been processed.
- If `/write-epic` reports failure for any Epic, surface the error to the user and
  stop the chain. The remaining Epics can be picked up by re-running this skill or
  by invoking `/write-epic` directly later.

This auto-chain preserves the original one-command UX: the user invokes `write-plan`
once and receives a Plan with every Epic decomposed into Tasks (and Subtasks, per
`write-epic`'s behavior).
