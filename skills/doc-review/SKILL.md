---
name: doc-review
description: Review documentation completeness after task completes. Checks user-facing docs updated (the README or the repo's named docs/ files, per its doc-writing guide) and architecture docs updated (the repo's named architecture docs per its doc-writing guide, else master_plan.md) with new functionality. Returns structured list of documentation work items.
---

## Overview

Review documentation completeness after Task completion. Verify user-facing docs and any architecture documents.
Concise is better than verbose. Value brevity.
User-facing docs are for human users that want to use the program.
Architecture docs should contain high level architecture and architecture and core technology.

## Mission

Analyze what changed, compare against current docs, return list of specific documentation gaps.

### User-facing docs
Check: user-facing docs updated (the README or the repo's named docs/ files, per its doc-writing guide).
The repo's doc-writing guide (CLAUDE.md "Documentation Locations") decides where docs go; the README is the fallback when the repo has no guide.
User-facing docs are for human users to understand how to install and run the project
- No implementation details
- No testing or unit testing details
  - This is IMPORTANT. Seriously. Don't talk about how to test the product in the user-facing docs. 
- No discussion of security implications or requirements
- No error cases, migration or recovery steps
- No history - just the current state of things
- Keep it short and simple - focused on how to install and how to use
- Don't describe how to use common tools (like screen, poetry, bash etc)
### Architecture docs
Check: architecture docs updated (the repo's named architecture docs per its doc-writing guide, else master_plan.md).
The repo's doc-writing guide (CLAUDE.md "Documentation Locations", "Internal architecture docs") names the architecture docs; master_plan.md is the fallback when the repo has no guide.
Architecture docs are for LLM codegen agents as a "cheat sheet" to understand the code base without reading all of it
- Dont brag about or rationalize the code
  - No performance details
  - Dont describe how comprehensive the tests are or the testing strategy
  - Dont describe design decisions, trade offs or other designs considered 
  - Dont describe what happened before - just the current state of things
  - No design patterns
- No code - that defeats the purpose, the LLM can read the code if it wants to
  - no functions, no methods
- Do add:
  - list of logical components
  - what the components do
  - how to components interact
  - how data flows through the components
  - use of resources  like databases or file storage
  - schemas or API endpoints

## Workflow

### 0. Understand Project Documentation Standards

Before working, read the engineering best practices guide referenced in the project's CLAUDE.md under "Documentation Locations". Where the guide conflicts with the PRD, SRD, plan or task, the guide wins. Reviewers must report any violation of the guide as a blocking finding, even when the spec asks for the violating behaviour.

Find any documentation standards, style guides, or writing conventions in the project (e.g. `CONTRIBUTING.md`, `docs/standards.md`, architecture docs, or references in `CLAUDE.md`).
Your job is to flag gaps where the work done deviates from whatever standards are defined.

### 1. Understand What Changed

Review all commits and changed files to understand the scope of work: new features, changed behavior, new commands/APIs, config/schema changes.

### 2. Review Current Documentation

Read the user-facing docs (where the repo's doc-writing guide puts them; README.md if it has none) and Architecture docs (the ones its guide names; master_plan.md if it has none) to understand current state.

### 3. Re-read Every Touched Doc in Full

For each changed README, or any doc the repo's doc-writing guide governs, read the WHOLE file, not just the diff.
Check it against the guide and the doc's own stated rules: length and scope, no implementation detail, no error cases, migration or recovery, no history, and anything else the guide forbids (or the User-facing docs rules above when the repo has no guide).
Flag every violation anywhere in the file, including text this change did not touch.

### 4. Find Documentation Gaps

**User-facing docs** (user-focused, concise):
- Install instructions correct?
- Updated setup/dependencies?
- Is the list of MCP server commands correct
- If yes, fix it. If no, LEAVE IT ALONE! (Step 3 findings still apply to the whole touched doc.)


**Architecture Docs** (cheat sheet for llms):
- Has the high level architecture changed?
- Have new components been introduce or old ones remove?
- Schema/API changes?
- Data flow still accurate?
- If yes, fix it. If no, LEAVE IT ALONE! (Step 3 findings still apply to the whole touched doc.)


### 5. Check for Inconsistencies
Look for docs that are now incorrect: outdated commands, deprecated features still shown, changed file paths, old config formats.

### 6. Find and reduce duplication and waste
- Look for sections of the docs that repeat information and suggest removing them.
- Look for sections of the doc that are too verbose and recommend ways to compact them without losing meaning
- Ensure docs serve the right purpose:
  - User-facing docs are a user manual
  - Architecture docs are a cheat sheet for LLMs to understand the architecture and core technology

### 7. Output Work Items

Return specific, actionable items as numbered list:

```markdown
## Documentation Review Work Items

1. Update README.md Quick Start - add `new-command` usage
2. Mark Component X as Implemented in master_plan.md:289
3. Remove deprecated `old-cmd` from README Commands section
```

Or if no issues:
```markdown
## Documentation Review Work Items

No documentation issues found. User-facing docs and architecture docs are up to date!
```

NOTE: It is OK to return "no issues found". Only return issues if they are very important. 
In fact, if you never return "no issues found" the workflow will go into an infinite loop which is very bad!
**Important**
- Docs are wrong
- User-facing docs are missing information the user needs to use the app
- A touched doc breaks its doc-writing guide or its own rules anywhere in the file (step 3)
**Not Important**
- Formatting issues
- Grammar

**Work item quality:**
- Be specific: include file, section, line number when possible
- Be actionable: "Update X section - add Y detail" not "docs need work"
- Focus on user-visible changes and breaking changes
- Skip trivial wording/style issues

**Priority guide:**
1. Critical: New commands, changed APIs, altered workflows
2. Important: Component status, schema changes, new features
3. Nice-to-have: Enhanced examples, clarifications

