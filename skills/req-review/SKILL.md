---
name: req-review
description: Review product and software requirement documents and provide suggestions
disable-model-invocation: true
---

# Overview

Your role is to help the user get Product Requirement and Software Requirement documents
into a good enough state that they can be executed autonomously by a swarm of agents.

You will be provided a directory which contains one or more documents. You must read and review them all.
If you are not provided a directory, as the user to provide one.

# Style

Requirements docs value content of form. 
Requirements need not be in User Story format.
Style is concise and direct, less is more.

# Success Criteria

- The docs are logically consistent
  - No contradictory statements
  - Any control is complete, has no gaps and no unexpected cycles
- The docs are complete and thorough
  - All edge cases with explanations on how to handle them
  - Features are described in enough detail that no assumptions must be made during implementation
  - Clear acceptance criteria for each requirement
  - Requirements are testable and measurable
  - Dependencies and assumptions are explicitly documented
  - Non-functional requirements specified (performance, security, scalability, etc.)
  - Work not in scope detailed to prevent scope creep

# Review Process

Follow this systematic approach:

1. **Review Code Base**: After reading the documents, make yourself aware of the relevant source files and docs.
2. **Structure Check**: Verify document has clear sections, headers, and organization
2. **Completeness Scan**: Check all major areas are covered (see checklist below)
3. **Logic Review**: Identify contradictions, gaps, or circular dependencies
4. **Implementation Readiness**: Assess if an agent could implement without making assumptions

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

# Output Format

Give an overview of all issues (criticality, short title, short summary, format it pretty) and then present them one at a time to the user.
Start with the most critical. Give the user options and always include the option
to either Skip this concern or enter their own response. Also give them the "Chat about this" option. 

# Guidelines

- Be specific: Reference exact sections, lines, or requirements
- Be constructive: Suggest fixes, don't just criticize
- Prioritize: Critical issues first, minor polish last
- Focus on executability: Can an AI agent implement this without human clarification?
- Question assumptions: If something seems implied but not stated, flag it

# Next Steps

After all feedback is complete, use AskUserQuestion to ask the user for each doc reviewed:
- "Mark [PRD/SRD] as `pupa`?"
  - Options: "Yes, mark as pupa" / "No, more work needed"
- If yes, update the doc's ticket status to `pupa`.

After marking docs, check if **both** the PRD and SRD are now `pupa`. If so, use AskUserQuestion to ask:
- "Both PRD and SRD are pupa. Mark the Idea Bee as `pupa` too?"
  - Options: "Yes, mark as pupa" / "No, not yet"
- If yes, update the Idea Bee's status to `pupa`.

Then recommend:
- `/make-plan <bee-id>` — create a Plan Bee with Epics ready for implementation