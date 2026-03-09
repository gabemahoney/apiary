---
name: hatch-feature
description: Read documents from Idea Bee and creates a Feature Bee with Epics to describe the work that needs to be done.
disable-model-invocation: false
---

# PRD/SRD to Bees

## Workflow

### Setup:
If there is no hive called Features, ask the user if you can create one somewhere. It must have these child tiers:
- t1 — Epic / Epics
- t2 — Task / Tasks
- t3 — Subtask / Subtasks

### 1. Read PRD/SRD and make top level Bee

The user should provide you a Bee in the Ideas hive. If they do not, list the Idea hive and ask them which they want to hatch.
First check if the Idea Bee is in the `pupa` state (which means its ready to be worked on). If not, warn the user and ask if they want to continue.
It will have children that represent the documents describing the work to be done.
These might take the form of Product Requirements Docs, Software Requirements docs, UI mockups.
 

- Get file paths, read documents, extract features/requirements/acceptance criteria
- Then expand beyond what the user sent. Look in the repo and read any architectural documents to understand design constraints.
- Before decomposition, align on the outcome. If you cannot clearly state what changes for the user or system when the Bee is complete, do not decompose yet.
  - From the PRD:
    - User or customer outcome
    - Business goal or KPI
    - Scope and constraints
  - From the SRD:
    - Non-negotiable requirements (security, latency, availability)
    - Architectural boundaries
    - External dependencies



### 2. Create the Bee

Goal: Create one top level Bee ticket in the Features hive to track the work
- Contains a brief summary of the goal and scope (2-3 sentences max).
- Set `up_deps` to the Idea Bee's ID so the Feature Bee depends on the Idea Bee being finished.

**Setting the `egg` field:**
1. Read `~/.bees/config.json` and find the `egg_resolver` configured for this scope (check scope-level, then global-level).
2. If an `egg_resolver` is configured, read that script file and locate its `## RESOLVER CONVENTION` block. Follow its instructions exactly.
3. If no resolver is configured, set `egg` to the absolute path of the source documents provided by the caller.

- Mark the Bee as `larva` (its children — the Epics — have not been written yet)

### 3. Break Feature Bee down into Epics

#### Every Epic Must Leave the Codebase Green
Every Epic must leave the codebase in a working state with all existing tests passing. This is the non-negotiable constraint for all Epics.

#### One Epic = One Outcome
- An Epic represents a single, coherent, user- or system-visible capability.
- Avoid Epics organized by system layers (e.g., backend, frontend).
- Prefer Epics defined by observable outcomes.
- An Epic may span multiple systems but must have one measurable success condition.

#### Decompose Vertically by Capability
Break Epics into stories that deliver end-to-end behavior.

Avoid technology layer stories:
- Database Epic ❌
- API Epic ❌
- UI Epic ❌
- Documentation Epic ❌
- Testing Epic ❌

Prefer capability slices:
- Epic: User performs action and receives feedback ✅
- Epic: System handles error and retry behavior ✅
- Epic: Metrics and logging are emitted ✅

Each Epic should be independently testable and demo-able.

##### Exception: Technical Refactors

For pure infrastructure or refactor work, strict vertical slicing may not apply. Pure-tech Epics are allowed provided they leave the codebase green (see above).

- **Go vertical as soon as possible.** After foundational Epics, each subsequent Epic should add a demonstrable capability. Bundle infrastructure each slice needs into that slice rather than separating into layer Epics.

**Anti-Patterns to Detect:**
- Epic chain where intermediate states are untestable
- Mixing pervasive refactor with feature work in one Epic

##### Granularity
Make Epics as granular as possible while adhering to the above constraints of one outcome and vertical decomposition.
Its OK to have a lot of Epics as long as: 
- logical outcomes and acceptance criteria are still contained in one Epic
- Epics still represent a vertical slice of end-to-end behavior (unless the technical refactor exception applies)
Imagine that we will celebrate the completion of each new Epic with a birthday party! Its ok to have a lot!

##### Acceptance Criteria
Provide clear actionable Acceptance Criteria that the user can use to objectively evaluate success.
Is there some artifact the User can interact with to test the Epic? If so, detail the steps they will take to do so.
If not, explain how the agent itself can demonstrate that the Epic was completed successfully.
The Acceptance Criteria should be a detailed description of what a "sprint demo" of the Epic would entail.

Good examples:
- Server starts on http://localhost:8000
  - Good because it explains how the user can validate
- Agent builds unit tests that validate the API endpoints respond to HTTP requests
  - Good because it explains how the agent will demonstrate success

Bad examples:
- Server is available for use
  - Bad because it does not explain how the user can validate
- API endpoints respond to HTTP requests
  - Bad because the user cannot validate themselves, and does not explain how the agent itself will demonstrate success

#### Present All Epics for User Review
When all Epics are complete, present them to the user for final review.
- Output as markdown: title, description, dependencies for each Epic
- **Use AskUserQuestion tool** to ask: proceed with creation, modify Epics, or cancel
- **Wait for approval.** Allow modifications if requested.

### 4. Create Shell Epics in Feature Bee

#### Creating Epics in the bees MCP server

Create T1 type child tickets in the Feature bee with status `larva` (their children — Tasks — have not been written yet).
Use the egg resolver to determine what to put in the `egg`.
**NOTE**: If the feature is small, there may only be one Epic. You dont need to make multiple.

##### Epic Viability Checklist
[ ] No testing Epic - testing is folded into the Epics where the work is done
[ ] No documentation Epic - documentation is folded into the Epics where the work is done


#### Setup dependencies
- After all Epics are created, analyze and set up blocking relationships.
- Common Dependency Patterns:
  - **Infrastructure blocks features**: Backend API must exist before frontend/features can use it
  - **Foundation blocks UI**: Data models/services block UI components that display them
  - **Data input blocks processing**: Upload/import features block features that process that data
  - **Auth blocks protected features**: Authentication blocks features requiring authorization

For each Epic, ask: "What must be completed before this Epic can be worked on?"

#### Set Status
- Set the top level Bee to `pupa` (it is written and its children — the Epics — are now written)
- Each Epic should already be `larva` from creation (they are written but their children — Tasks — are not yet written)

### 5. Report

Output markdown summary:
- Bee and Epics created
- Each Epic: ID, title, status, dependencies (if any)
- Dependency relationships created

### 6. Ask to continue to next step

The next optional step is described in the skill called `hatch-epic`. Ask the User if they want to proceed or stop here.
If they want to continue, load the `hatch-epic` skill and execute it with each epic until done 
- be mindful to hatch-epics in dependency order
- you do not have to ask permission from the user to hatch any specific epics, just go through all of them.
If not, you are done.

