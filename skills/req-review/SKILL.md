---
name: req-review
description: Review product and software requirement documents (and any other Feature-level docs) and drive their `draft → ready` transitions
---

# Overview

Your role is to help the user get a Feature ticket's requirement documents into a good
enough state that they can be executed autonomously by a swarm of agents.

This skill operates against a **Feature ticket** in the Features store. The Features
store is prescriptive: every direct child of a Feature ticket is a document (PRD, SRD, or
any other doc type the project uses). This skill reviews **all** Doc children of the
given Feature, not a hardcoded PRD+SRD pair.

# Inputs

- A **Feature ticket ID**. If the user did not provide one, ask the user for it.
- From that Feature ticket, fetch **all Doc children** — these are the docs to review.

# Status Ownership

This skill explicitly OWNS the following status transitions:

- Each Doc child (PRD, SRD, or any other doc type): `draft → ready`
- Feature ticket: `draft → ready` (only after all Doc children are `ready`, and only on
  user confirmation)

This skill explicitly does NOT set:

- Feature ticket → `active`
- Feature ticket → `done`
- Any Doc child beyond `ready` (e.g., not to `active` or `done`)
- Any non-Feature, non-Doc-child ticket type

All status writes are **idempotent**: if a doc or Feature is already `ready`, the skill
does not re-write the status, and re-running this skill on a fully-`ready` Feature is a
valid no-op flow that simply confirms current state.

# Review Process

Follow this systematic approach.

## Step 0: Read `apiary.md`

Before anything else, read `<project_root>/apiary.md`. If it has a Documentation
Locations section relevant to where docs live or are referenced, take note of it. If
`apiary.md` is missing, print a one-line notice and proceed.

## Step 1: Resolve the Feature ticket

If the user did not give you a Feature ticket ID, ask for one. Fetch the Feature
ticket and list its **Doc children** — the docs to review.

## Step 2: Review Code Base

After identifying the docs, make yourself aware of the relevant source files and
existing project docs so your review is grounded in the codebase, not just the doc text.

## Step 3: Per-doc review loop

For each Doc child of the Feature:

1. **Skip-if-ready (resume support)**: If the doc's status is already `ready`, skip
   the review step by default — do not re-review. The user MAY explicitly request a
   re-review of a `ready` doc; honor that on request, but the default behavior on
   resume is to skip.
2. **Pick the checklist** based on the doc's title/role:
   - PRD-style doc → use the PRD checklist below.
   - SRD-style doc → use the SRD checklist below.
   - Any other doc type → use the **General Doc** checklist below (which delegates
     to the top-level Success Criteria — "is this doc clear, complete, and
     actionable?" — plus heuristics common to all docs).
3. **Structure Check**: Verify the doc has clear sections, headers, and organization.
4. **Completeness Scan**: Check all major areas are covered (per the picked checklist).
5. **Logic Review**: Identify contradictions, gaps, or circular dependencies.
6. **Implementation Readiness**: Assess whether an agent could implement without
   making assumptions.
7. **Present findings to the user** in the Output Format below.
8. After feedback resolves, ask the user:
   - Question: `"Mark [doc title] as ready?"`
   - Options: `"Yes, mark as ready"` / `"No, more work needed"`
   - On `"Yes"`: set the doc's status to `ready` (idempotent — if already `ready`,
     no-op).
   - On `"No"`: leave the doc in `draft`; the user can re-run this skill later to
     resume.

## Step 4: Promote the Feature

After the per-doc loop completes, check whether **all** Doc children of the Feature
are now `ready`. If yes, ask the user:

- Question: `"All docs are ready. Mark Feature as ready?"`
- Options: `"Yes, mark Feature as ready"` / `"No, not yet"`
- On `"Yes"`: set the Feature ticket's status to `ready` (idempotent — if already
  `ready`, no-op).
- On `"No"`: leave the Feature in `draft`.

If not all Doc children are `ready`, do not prompt to promote the Feature. Tell the
user which docs are still in `draft` and stop.

## Step 5: Closing recommendation

After the Feature is `ready` (or the user defers), recommend the next skill:

- `/write-plan <feature-ticket-id>` — create a plan with Epics ready for
  implementation.

# Style

Requirements docs value content over form.
Requirements need not be in User Story format.
Style is concise and direct, less is more.

# Success Criteria

These are the cross-cutting heuristics applied to every doc, regardless of type:

- The docs are logically consistent
  - No contradictory statements
  - Any control flow is complete, has no gaps and no unexpected cycles
- The docs are complete and thorough
  - All edge cases with explanations on how to handle them
  - Features are described in enough detail that no assumptions must be made during
    implementation
  - Clear acceptance criteria for each requirement
  - Requirements are testable and measurable
  - Dependencies and assumptions are explicitly documented
  - Non-functional requirements specified (performance, security, scalability, etc.)
  - Work not in scope detailed to prevent scope creep

# Review Checklist

## For Product Requirement Documents (PRD)

- [ ] Problem statement clearly defined
- [ ] Acceptance criteria defined
- [ ] Edge cases and error scenarios covered
- [ ] UI/UX requirements described or wire-framed (if applicable)
- [ ] Mobile/responsive behavior defined (if applicable)
- [ ] Assumptions explicitly stated

## For Software Requirement Documents (SRD)

- [ ] Deployment requirements specified or explicitly omitted
- [ ] Performance requirements specified or explicitly omitted
- [ ] API endpoints specified or explicitly omitted
- [ ] Data models and schemas specified or explicitly omitted
- [ ] Authentication/authorization approach specified or explicitly omitted
- [ ] Security requirements specified or explicitly omitted
- [ ] Testing strategy specified or explicitly omitted

## For any other doc type (General Doc fallback)

When a Doc child is neither a PRD nor an SRD, fall back to the cross-cutting Success
Criteria above plus this general checklist. The driving question: **"Is this doc
clear, complete, and actionable?"**

- [ ] Purpose and scope of the doc are stated up front
- [ ] Audience and consumers of the doc are clear (who acts on it?)
- [ ] Content is logically consistent — no contradictions, no unexplained gaps
- [ ] Terminology is consistent with the rest of the Feature's docs
- [ ] Acceptance criteria or success conditions are present and testable where the
      doc type implies them
- [ ] Dependencies on other docs / systems / tickets are explicit
- [ ] Out-of-scope items are called out to prevent scope creep
- [ ] An implementing agent could act on this doc without needing to ask clarifying
      questions

# Output Format

Give an overview of all issues (criticality, short title, short summary, format it
pretty) and then present them one at a time to the user. Start with the most
critical. Give the user options and always include the option to either Skip this
concern or enter their own response. Also give them the "Chat about this" option.

# Guidelines

- Be specific: Reference exact sections, lines, or requirements
- Be constructive: Suggest fixes, don't just criticize
- Prioritize: Critical issues first, minor polish last
- Focus on executability: Can an AI agent implement this without human clarification?
- Question assumptions: If something seems implied but not stated, flag it
- Idempotency: All status transitions written by this skill are safe to repeat. If a
  doc or the Feature is already `ready`, do not re-write the status.
- Resume: A partially-reviewed Feature can be re-entered at any time. On resume, skip
  docs already `ready` and only review docs still in `draft`, unless the user
  explicitly asks to re-review a `ready` doc.

# Next Steps

After the per-doc loop and the Feature promotion prompt, recommend:

- `/write-plan <feature-ticket-id>` — create a plan with Epics ready for
  implementation.
