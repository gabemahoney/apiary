---
name: new-feature
description: Create a new Feature ticket in the Features store, optionally seeded by a source reference
---

# Overview

The user wants to file a new feature. This skill reads `apiary.md`,
evaluates the **New Feature** section to decide how to gather context,
ensures the Features store exists, and creates a Feature ticket with
status `draft`.

The skill stays backend-agnostic — speak generically about "tickets" and
"stores"; the LLM picks whichever ticket-backend tools are available at
runtime.

# Steps

## 1. Read `apiary.md`

Read `<project_root>/apiary.md` from the project root (the current
working directory, unless the user has indicated otherwise).

If `apiary.md` does **not** exist, jump to the **Fallback: no apiary.md**
section below.

## 2. Evaluate the New Feature section

Locate the `## New Feature` section in `apiary.md`. Two valid shapes:

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

## 3. Confirm the Features store exists

The Features store must exist before a Feature ticket can be created.
Query the ticket backend for a store named `Features`.

- If it exists, continue.
- If it does not exist, ask the user where to create one and create it.
  If the user prefers a full setup pass, suggest running `/project-setup`
  instead.

## 4. Gather Feature context

Branch on the resolved `source_references` value from step 2:

- **`github resolver`** (or another configured resolver) — ask the user
  for the input value the resolver expects (for example, a GitHub issue
  URL or issue number). Record the input value as a source reference on
  the Feature, tagged with the resolver type.

  Optional pre-population: if the resolved content is fetchable cheaply
  (e.g., `gh issue view <id>` for a GitHub issue), use the title and
  body to pre-fill the Feature title and body. If the fetch fails for
  any reason, gracefully skip pre-population and continue with an
  interview — do not error out.
- **`none, interview user`** — ask the user for a short title and a
  description. Use those answers to populate the Feature ticket. No
  source reference is recorded.

If the user wants to add anything beyond the resolver-fetched content
(extra context, links, acceptance notes), capture that and append it to
the Feature body.

## 5. Create the Feature ticket

Create a new ticket in the Features store with:

- **Title** — from the resolved source content or from the user.
- **Body** — the feature description plus any resolver-fetched content.
- **Status** — `draft`. This step is idempotent: if the ticket somehow
  already carries `draft`, do not error.
- **Source reference** — the resolver input value from step 4, when a
  resolver was used.

This skill **only** sets the Feature to `draft`. It must never write
`ready`, `active`, or `done` — those transitions belong to other
skills (notably `req-review` for `draft -> ready`).

## 6. Next-step prompt

After the Feature ticket is created, ask the user about the canonical
next-step options. The question text should be along the lines of:
"Feature ticket created in `draft`. How would you like to proceed?"

Options (use these labels exactly):

- `write-prd` — run the `/write-prd` skill to author a Product
  Requirements Document for this feature.
- `write-srd` — run the `/write-srd` skill to author a Software
  Requirements Document for this feature.
- `mark as ready` — escape hatch. The user takes responsibility for
  promoting the Feature to `ready` themselves. **This skill does NOT
  set the Feature to `ready`** — automatic `draft -> ready` promotion
  is owned by `req-review`. Selecting this option simply hands control
  back to the user.

# Fallback: no apiary.md

If `apiary.md` is missing, do a short interview instead:

1. Ask the user for the feature title and a short description.
2. Confirm the Features store exists (step 3 above) — if not, ask where
   to create one and create it.
3. Create the Feature ticket with status `draft` and the interview
   content as the body. No source reference is recorded.
4. Surface a one-line suggestion: "Run `/project-setup` to configure
   `apiary.md` for richer feature intake next time."
5. Run the next-step prompt from step 6.

# What comes next

Depending on the user's selection in step 6, the natural follow-ons are
`/write-prd`, `/write-srd`, or a manual user-driven promotion to
`ready` (typically after running `/req-review`).
