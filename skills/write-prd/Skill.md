---
name: write-prd
description: Write a Product Requirements Document (PRD) for a development effort. Adds or modifies a PRD Bee to a Bee in the Ideas hive that downstream skills (write-srd, req-review, hatch-epic) can consume.
---

# Write PRD

## Overview

The caller wants to write a PRD — a plain-english description of work to be done, stated as requirements. 
The PRD is the origin document for a development effort.
Downstream skills will consume it:

- `/write-srd` — turns the PRD into software requirements
- `/req-review` — reviews the PRD for completeness and executability
- `/hatch-epic` — breaks the work into Epics, Tasks, and Subtasks

The PRD must be thorough enough that an agent swarm can execute the work
autonomously, without needing to ask the user clarifying questions.

## Workflow

### 1. Determine the Idea Bee

The PRD will be a child of a Bee in the Ideas hive.
If the user did not provide a Bee id, ask it which Idea Bee you should build the PRD under.

### 2. Gather Requirements

You and the user may already have been chatting about and idea, if so, fill out the PRD body with as much
information as you already have,

Then, interview the user to understand the work. Ask focused questions to fill gaps so you can update the PRD.

Key questions to explore:
- What is the Problem Statement?
- What are the Acceptance Criteria?
- What is explicitly out of scope?
- Are there edge cases or error scenarios to handle?
- Are there UI/UX considerations?
- Are there performance or security constraints?
- Does this depend on or affect other work?

Use `AskUserQuestion` to gather information efficiently. Present options where
possible rather than open-ended questions.

#### Style Rules

- **Plain english.** No code, no pseudocode, no technical implementation details.
- **Concise and direct.** Less is more. Every sentence should add information.
- **Requirements, not solutions.** Describe what must be true, not how to build it.
- **Specific and testable.** Every requirement should be verifiable.
- **No user-story format required.** Just clear statements of what is needed.
- **Do not number product requirement numbers. Use plan conversational English to describe the problem**

#### Required Sections

```markdown
# [Feature/Project Name]

## Problem Statement
What problem exists today? Who has it? Why does it matter?
Keep this to 2-3 sentences max.

## Goals
What must be true when this work is complete?
Bulleted list of concrete outcomes.

## Non-Goals / Out of Scope
What this effort explicitly will NOT do.
Prevents scope creep during implementation.

## Requirements

### Functional Requirements
What the system must do. Each requirement should be:
- Specific enough to test
- Independent where possible

### Edge Cases and Error Handling
How the system should behave in non-happy-path scenarios.
Each edge case should describe the scenario and the expected behavior.

### Non-Functional Requirements (if applicable)
Performance, security, scalability, compatibility constraints.
Only include if relevant — do not pad with boilerplate.

### UI/UX Requirements (if applicable)
Layout, interaction, and presentation requirements.
Only include if the feature has a user-facing component.

## Acceptance Criteria
The checklist that must pass for this work to be considered done.
Each criterion should be observable and testable.
Format as a checklist:
- [ ] Criterion 1
- [ ] Criterion 2

## Assumptions
What is assumed to be true going into this work.
Dependencies on other systems, existing functionality, or prior work.

## Open Questions (if any)
Unresolved decisions flagged for user review.
Remove this section if there are none.
```

### 3. Present and Iterate

After writing the PRD, present a summary to the user:
- One-line description of each section
- Call out any assumptions made
- Highlight open questions

Use `AskUserQuestion` with:
- Question: "PRD is ready. How would you like to proceed?"
- Options:
  - "Looks good, save it"
  - "I have changes to suggest"
  - "Run /req-review on it"

If the user has changes, iterate until they approve.

## Quality Checklist

Before presenting the PRD, verify:

- [ ] Problem statement is clear and concise
- [ ] Goals are concrete and measurable
- [ ] Non-goals section exists and prevents scope creep
- [ ] Every functional requirement is testable
- [ ] Edge cases and error scenarios are covered
- [ ] Acceptance criteria are specific and observable
- [ ] Assumptions are explicitly stated
- [ ] No implementation details or code anywhere in the document
- [ ] No contradictions between sections
- [ ] An agent could implement this without making assumptions

## What NOT to Include

- Source code, pseudocode, or technical implementation details
- Architecture decisions or design proposals (that's the SRD's job)
- Time estimates or scheduling
- Specific file paths, function names, or line numbers
- "How to build it" — only "what must be true"

### 4. Write the PRD

Add the PRD as a child to the Idea bee.
Use the title "PRD".
Set its status to `larva`.

### 5. Next steps

Suggest the user run `/req-review <idea-bee-id>` to review and finalize the docs before they can be marked `pupa`.
