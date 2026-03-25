---
name: write-srd
description: Write a Software Requirements Document (SRD) from a Product Requirements Document (PRD). Explores the codebase, then defines what must be true about the software without solving the problem.
disable-model-invocation: true
---

# Write SRD from PRD

## Overview

Given a PRD, write a companion Software Requirements Document (SRD) that defines
what must be true about the software for the PRD to be satisfied. The SRD does not
solve the problem — it provides guidance on constraints the implementation must meet.

Be definitive in the SRD. Do not present options unless you believe further research is needed
- Flag such comments as **RESEARCH NEEDED**

## Input

The user will provide either a Bee in the Idea hive, a PRD bee child of such a bee or nothing.
If nothing, as them for the Bee in the Idea hive that has the PRD as a child.

## Process

### 1. Read the PRD

Read the PRD ticket thoroughly. Understand:
- What is being built
- What components are affected
- What edge cases exist
- What acceptance criteria are defined

### 2. Explore the Codebase

Before writing anything, explore the codebase to understand:
- **Architecture**: Top-level directory structure, module organization
- **Affected modules**: Which files/modules will need changes based on the PRD
- **Existing patterns**: How similar features are currently implemented (config loading, validation, error handling, serialization, etc.)
- **Data models**: Current model definitions that will be extended
- **Test fixtures**: How tests are structured, what helpers/factories exist, what constants are centralized
- **Configuration**: How config is loaded, validated, and accessed

Use the Explore agent for this research. Be thorough — the SRD must reference
real modules, patterns, and conventions from the actual codebase.

#### Style Rules

- **No source code.** No code examples, no function signatures, no pseudocode.
- **No line numbers.** You can mention files but not lines, these are going to change during execution.
- **Define what must be true**, not how to implement it.
- **Be specific.** Reference actual module names and patterns from the codebase.
- **Be concise.** Each requirement should be one clear statement.
- **Be testable.** Every requirement should be verifiable.

#### Structure

Organize requirements by domain/concern, not by PRD section. Use numbered requirement
groups with subsections:

```
## SR-1: [Domain Name]

### SR-1.1: [Specific Concern]
- Requirement statement
- Requirement statement

### SR-1.2: [Another Concern]
- Requirement statement
```

#### Required Sections

Derive these from the PRD. Common patterns include:

- **Data Model** — Fields, types, constraints, serialization, which entities are affected
- **Configuration** — Schema changes, validation rules, resolution/fallback logic
- **API / Commands** — New endpoints or tools, parameters, return types, registration
- **Execution / Processing** — Runtime behavior, lifecycle, external process management
- **Validation / Linting** — New rules, integration with existing validation, corruption marking
- **Existing Behavior** — What must NOT change, what must be preserved
- **Test Fixtures** — How to extend existing test infrastructure (see below)
- **Documentation** — What docs must be written after implementation

#### Test Fixtures Section (Required)

Every SRD must include a test fixtures section. This prevents implementors from
hardcoding values and breaking test conventions. The section must:

1. **Identify the test architecture** — Where fixtures, helpers, constants, and docs live
2. **Mandate use of existing helpers** — Name the specific factory functions and fixtures
   that must be used instead of hardcoding
3. **Specify helper extensions** — What parameters need to be added to existing helpers
   for the new feature
4. **Specify new fixtures** — What new fixtures are needed and where they should be defined
5. **Specify test organization** — Which existing test files new tests belong in vs.
   new test files needed

The goal: an implementor who reads only the SRD test section should know exactly which
helpers to call, which fixtures to use, and where to put new tests — without hardcoding
any YAML, JSON, or string literals in test files.

### 4. Present to User

After writing the SRD, present a summary of the requirement groups to the user.
List each SR group with a one-line description.

## What NOT to Include

- Source code or pseudocode
- Implementation details (which functions to call, which libraries to use)
- Step-by-step implementation plans
- Time estimates
- Architectural proposals or design alternatives

## Quality Checklist

Before presenting the SRD, verify:

- [ ] Every PRD acceptance criterion maps to at least one SR
- [ ] Every SR is testable and specific
- [ ] No contradictions between SRD and PRD
- [ ] Codebase references are accurate (real module names, real fixture names)
- [ ] Test fixtures section names specific helpers, factories, and constants files
- [ ] No source code anywhere in the document
- [ ] Requirements define "what must be true" not "how to implement"

### 3. Write the SRD

Write the SRD as a child of the Idea Bee.
Use the title "SRD".
Set its status to `larva`.

### 4. Next steps

Suggest the user run `/req-review <idea-bee-id>` to review and finalize the docs before they can be marked `pupa`.
