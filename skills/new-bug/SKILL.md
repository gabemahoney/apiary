---
name: new-bug
description: Capture a new bug as a ticket in the Bugs store
---

# Overview

The user has a new bug to file. This skill reads `apiary.md`, evaluates the
**New Bug** section to decide how to gather context, ensures the Bugs store
exists, and creates a Bug ticket with status `open`.

The skill stays backend-agnostic — speak generically about "tickets" and
"stores"; the LLM picks whichever ticket-backend tools are available at
runtime.

# Steps

## 1. Read `apiary.md`

Read `<project_root>/apiary.md` from the project root (the current working
directory, unless the user has indicated otherwise).

If `apiary.md` does **not** exist, jump to the **Fallback: no apiary.md**
section below.

## 2. Evaluate the New Bug section

Locate the `## New Bug` section in `apiary.md`. Two valid shapes:

- **Decision-tree shape** — a (possibly empty) plain-english rationale
  paragraph, followed by a bullet list of factor lines
  (`- <factor_name>: <current value>`), followed by one or more
  `When <factor_name> = <value>:` branches. Each branch contains a
  `- source_references: <resolver>` line (for example, `github resolver`)
  or `- source_references: none, interview user`.

  Match the **current value** on each factor line against the `When ... =
  ...:` branches and select the branch whose value list contains the
  current value. If no branch matches, fall back to interview (treat it
  as `source_references: none, interview user`).

- **Simple (degenerate) shape** — a single bullet
  `- source_references: <resolved value>` with no factor list and no
  `When ...:` branches. Use the value as-is.

Carry the resolved `source_references` value forward to step 4.

## 3. Confirm the Bugs store exists

The Bugs store must exist before a Bug ticket can be created. Query the
ticket backend for a store named `Bugs`.

- If it exists, continue.
- If it does not exist, ask the user where to create one and create it
  (no child tiers; status values `open`, `active`, `done`, `published`).
  If the user prefers a full setup pass, suggest running `/project-setup`
  instead.

## 4. Gather Bug context

Branch on the resolved `source_references` value from step 2:

- **`github resolver`** (or another configured resolver) — ask the user
  for the input value the resolver expects (for example, a GitHub issue
  URL or issue number). Record the input value as a source reference on
  the Bug, tagged with the resolver type.

  Optional pre-population: if the resolved content is fetchable cheaply
  (e.g., `gh issue view <id>` for a GitHub issue), use the title and
  body to pre-fill the Bug title and body. If the fetch fails for any
  reason, gracefully skip pre-population and continue with an interview
  for the title, description, and reproduction steps — do not error out.
- **`none, interview user`** — ask the user for a short title, a
  description, and any reproduction steps. Use those answers to populate
  the Bug ticket. No source reference is recorded.

If the user wants to add anything beyond the resolver-fetched content
(extra context, repro notes, links), capture that and append it to the
Bug body.

## 5. Create the Bug ticket

Create a new ticket in the Bugs store with:

- **Title** — from the resolved source content or from the user.
- **Body** — the bug description, repro steps, and any resolver-fetched
  content (e.g., the GitHub issue body).
- **Status** — `open`.
- **Source reference** — the resolver input value from step 4, when a
  resolver was used.

## 6. Next-step prompt

After the Bug ticket is created, ask the user about next steps. The
question text should be along the lines of:
"Bug ticket created in `open`. How would you like to proceed?"

Options (use these labels exactly):

- `fix-bug` — run the `/fix-bug` skill now to start working on this Bug.
- `leave open` — escape hatch. The Bug stays in `open` for later. The
  user can return to `/fix-bug <bug-id>` whenever they want; until then
  no skill touches this Bug. Selecting this option simply hands control
  back to the user.

# Fallback: no apiary.md

If `apiary.md` is missing, do a short interview instead:

1. Ask the user for the bug title, a description, and reproduction steps.
2. Confirm the Bugs store exists (step 3 above) — if not, ask where to
   create one and create it.
3. Create the Bug ticket with status `open` and the interview content as
   the body. No source reference is recorded.
4. Surface a one-line suggestion: "Run `/project-setup` to configure
   `apiary.md` for richer bug intake next time."
5. Run the next-step prompt from step 6.
