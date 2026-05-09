---
name: project-setup
description: One-time configuration of a project for the Apiary workflow
---

# Overview

Run this skill once per project to bootstrap Apiary: detect the available
ticket-backend, create the three stores Apiary expects (Features, Plans, Bugs),
configure their status values, and write an `apiary.md` manifest at the
project root.

The skill can be invoked in two ways:

- **With a template name** (e.g., `/project-setup public-github`) — the named
  template under `~/.claude/apiary-templates/` drives the New Feature, New Bug,
  and (optionally) Documentation Locations sections.
- **Interactively** — with no template name, the skill either offers the user
  any installed templates to pick from or proceeds in pure interview mode.

Re-running the skill in an already-configured project is safe — it detects
existing state, surfaces drift to the user, and changes nothing without
explicit approval. On rerun, environment factors recorded in the existing
apiary.md are re-detected and the user is prompted before any value changes
are written back.

Apiary supports two known ticket backends — **bees**
(https://github.com/gabemahoney/bees) and **beads**
(https://github.com/steveyegge/beads). This skill speaks generically about
"stores", "tickets", "tiers", and "status values"; map those to whichever
backend's tools are available at runtime.

# Entry gate

Before doing anything else, verify a backend is reachable — either as a
configured MCP server or as a CLI on PATH. If neither is present, tell the
user how to install one and exit:

- bees: https://github.com/gabemahoney/bees
- beads: https://github.com/steveyegge/beads

If exactly one backend is available, use it. If both are available, ask the
user which to use.

# Steps

## 1. Determine the project root

Ask the user for the absolute path of the project root (the directory that
will own `apiary.md`). Suggest the current working directory as the default
but require confirmation. Repo discovery and any automatic root inference
are out of scope for this skill — a later setup phase will add that.

## 2. Check for an existing apiary.md

If `<project_root>/apiary.md` already exists, read it and treat this run
as an idempotent re-configuration:

- Parse the existing `## Project Root`, `## New Feature`, `## New Bug`,
  and `## Documentation Locations` sections.
- Within `## New Feature` and `## New Bug`, parse the decision-tree
  shape: the plain-english rationale paragraph (if present), the bullet
  list of factor lines (`- <factor_name>: <current value>`), and the
  `When <factor_name> = <value>:` branches with their nested
  `- source_references: <resolver>` lines.
- Within `## Documentation Locations`, parse the `### Reference` and
  `### Maintained` subsections (each bullet line is
  `- **<Category>**: <comma-separated paths or "omitted">`).
- Compare the parsed values to whatever the user provides in later
  steps (interview answers, template-resolved values, or re-detected
  environment factors).
- For any drift, ask the user to confirm before overwriting.
- If nothing differs, report "already configured" and exit cleanly.

**Factor-change detection on rerun.** For each factor listed in the
existing `## New Feature` / `## New Bug` decision tree:

1. Re-detect the factor's current environment value using the same
   generic-language detection used when a template is applied (the LLM
   picks the appropriate tool — for example, for `github_visibility`,
   run `gh repo view --json visibility` from a repo path discovered in
   step 8, or fall through to "none" if no GitHub repo is configured).
2. Compare the re-detected value against the persisted value on the
   factor line.
3. If different, surface `<factor_name>: <old> → <new>` to the user and
   ask whether to update. Ask per-factor — the
   user may accept some changes and reject others.
4. On confirmation for a given factor, re-evaluate which `When ... =
   ...:` branch matches the new value, then rewrite both the factor
   line and the matched branch's `- source_references: ...` line in
   the `## New Feature` and `## New Bug` sections (a factor may appear
   in both — update wherever it is referenced).
5. If no factor changed, this concern is a no-op; the other idempotency
   paths in this step (drift on Documentation Locations, build commands
   in step 10, etc.) continue normally.
6. This factor-change detection reuses the same compare-and-confirm
   pattern that drives the rest of step 2's idempotency — it does not
   introduce a parallel control flow.

Treat the level-2 (`##`) headings as section markers. Preserve any unknown
sections you encounter — future skills will append more sections, and this
skill must not clobber them.

## 3. Discover existing stores

Query the backend for stores named `Features`, `Plans`, and `Bugs` (case
sensitivity follows the backend's own rules). For each:

- If missing, you will create it in step 7.
- If present, fetch its tier configuration and status values, and compare
  to the target configuration in step 7. Record any drift.

## 4. Resolve the template (if any)

Templates live in `~/.claude/apiary-templates/` (a global user-level
directory, OUTSIDE the project worktree). Each template is a plain
markdown file — no special syntax, no YAML, no annotations. The LLM
reads the template and follows its instructions.

**Selection.**

- If the skill was invoked with an argument (e.g.,
  `/project-setup public-github`), treat that argument as the template
  name and load `~/.claude/apiary-templates/<name>.md`.
- If no argument was provided and the templates directory contains one
  or more `*.md` files, list them and ask the user to pick one or pick
  "no template (interview)".
- If the templates directory is empty or does not exist, fall through
  to interview mode — step 5 will collect the values directly from the
  user.

**Error paths.**

- Templates directory missing → graceful fallback to interview mode
  (step 5). Tell the user where templates are expected to live so they
  can add some later.
- Named template not found → list the available templates and ask the
  user again.
- Template fails to parse (no `## New Feature Intake` heading, no `##
  New Bug Intake` heading, malformed `When ...:` branch, etc.) →
  report the parse error to the user and fall back to interview mode.

**Resolution pipeline.** Once a template loads cleanly:

1. Parse the template's `## New Feature Intake` and `## New Bug Intake`
   sections. Each section may carry a plain-english rationale
   paragraph, zero or more factor lines (`- <factor_name>:
   <description>`), and one or more `When <factor_name> = <value>:`
   branches whose body contains a `- source_references: <resolver>`
   line.
2. Parse the optional `## Documentation Locations` section. Each
   bullet under `### Reference Documentation` or `### Maintained
   Documentation` is either a hardcoded path (or comma-separated list
   of paths), the literal `omitted`, or — when the template says "Ask
   the user for each (record 'omitted' if skipped)" — a category that
   step 6 will interview.
3. For every factor listed in either intake section, detect the
   current environment value. The skill does not prescribe a specific
   tool — pick whatever fits the factor name; the factor's textual
   description in the template hints at the appropriate detection
   approach. (Illustrative example only: a `github_visibility` factor
   would call out to a GitHub CLI on a discovered repo path, falling
   through to `"none"` when no GitHub repo is configured. Substitute
   whatever tool the factor implies on your system.)
4. For each factor's detected value, match the appropriate `When
   <factor_name> = <value>:` branch. If multiple branches match (e.g.,
   `private or none` covers two values), pick the one whose value list
   contains the detected value. If no branch matches, fall back to
   asking the user.
5. Carry the resolved factor-value pairs and matched branch contents
   into step 11 — they become the rendered decision-tree shape in
   `## New Feature` and `## New Bug` (factor lines + matched
   `When ...:` branch with its `source_references` line).

Stores, status values, repo discovery, and build commands are NOT
touched by templates — those remain owned by `project-setup` directly
(steps 3, 5, 6, 7, 8) and behave the same regardless of which template
(or no template) was selected.

## 5. Interview the user

Ask the user with choice prompts. Collect the minimum needed to
populate `apiary.md`:

- **Project root** — confirm or correct the path from step 1.

If a template was applied in step 4, the New Feature / New Bug values
are already resolved — skip those prompts. Otherwise, the interview
covers them directly:

- **New Feature → source_references** — one of:
  - `github resolver` — the backend will resolve GitHub URLs/IDs itself.
  - `none, interview user` — the skill that creates a feature will ask
    the user for source material instead.
- **New Bug → source_references** — same two choices as above.

Resolve each value to a concrete string. Do not write `ask user` or any
other placeholder into `apiary.md` — the answer is captured here, in
this interview (or, when a template applies, in step 4).

In pure interview mode (no template), there are zero factors. The
resulting `## New Feature` / `## New Bug` sections are written in their
degenerate decision-tree form (no factor list, no `When ...:`
branches — just the resolved `- source_references:` line). See step 11
for the exact shapes.

## 6. Documentation Locations

Capture two categories of project documentation, each supporting a
list of paths (relative to the project root):

- **`### Reference`** — engineering best practices, test guide, doc
  writing guide.
- **`### Maintained`** — contributor docs, customer-facing docs.

For each category:

- If the template hardcoded a value, use it as-is.
- If the template said "ask the user" for that category (or no
  template applied), ask the user. Accept a single path, a
  comma-separated list of paths, or skip.
- If the template did not list the category at all, default to the
  literal word `omitted` (no backticks).
- If the user skips a category during interview, record `omitted`.

Multiple paths per category are written on the bullet line as a single
comma-separated value (matching the PRD example).

The Documentation Locations section sits between `## New Bug` and
`## Build Commands` in the rendered apiary.md (step 11).

Idempotency: parse any existing `## Documentation Locations` section
in apiary.md (handled in step 2). Present per-category drift to the
user and only change what the user approves.

## 7. Create or reconcile the three stores

Target configuration:

| Store    | Child tiers                                     | Status values                                  |
|----------|-------------------------------------------------|------------------------------------------------|
| Features | t1: Doc / Docs                                  | `draft`, `ready`, `active`, `done`, `published` |
| Plans    | t1: Epic / Epics; t2: Task / Tasks; t3: Subtask / Subtasks | `draft`, `ready`, `active`, `done`             |
| Bugs    | (none)                                          | `open`, `active`, `done`, `published`          |

For each store:

- If it does not exist, create it with the listed tiers and configure its
  status values.
- If it exists with the listed configuration, leave it alone.
- If it exists with different tiers or statuses, present the diff to
  the user and only change what the user approves.

Use whichever backend tools and CLI are available; the skill stays
backend-agnostic so the LLM running it picks the right calls.

## 8. Repo Discovery

Find the git repos that belong to this project. They drive the per-repo
build commands written in step 10.

1. Scan the project root for `.git` directories, walking 1–2 levels deep.
   Skip vendored directories: `node_modules`, `.venv`, `target`, `dist`,
   `build`, `.tox`.
2. Branch on what you found:
   - **One or more repos found.** List the relative paths to the user,
     then ask the user: "Does this project touch any other git repos
     not listed?" If yes, ask for additional paths (absolute or
     project-root-relative) and add them to the set.
   - **No repos found.** Ask the user: "Does your project use one or
     more git repos?" If yes, ask for paths and add them. If no,
     record an empty list and continue — the project has no build
     commands to configure.
3. Normalize every path to be relative to the declared project root —
   e.g., `./`, `./backend`, `./frontend`, `./packages/api`.
   Deduplicate.
4. Hold the resulting list in working memory; the next two steps consume
   it. Do **not** add a `## Repos` section to `apiary.md` — repos appear
   implicitly via `### <relative-path>` headings under `## Build
   Commands`.

**Idempotency.** On rerun, compare the freshly-discovered set against
the `### <relative-path>` headings under any existing `## Build
Commands` section in `apiary.md`. Ask the user before adding
newly-found repos or removing repos that no longer exist.

## 9. Stack Detection

For each repo path collected in step 8, inspect manifest files at the
repo root and resolve to a stack:

| Manifest                                 | Stack             | Notes                                                              |
|------------------------------------------|-------------------|--------------------------------------------------------------------|
| `Cargo.toml`                             | Rust              | Note `[workspace]` (workspace vs single crate)                     |
| `package.json` + `tsconfig.json`         | Node/TypeScript   | Detect test runner from `jest.config.*`, `vitest.config.*`, `package.json` |
| `package.json` (no `tsconfig.json`)      | Node/JavaScript   | Same test-runner detection                                         |
| `pyproject.toml` or `setup.py`           | Python            | If `poetry.lock` is present, prefix commands with `poetry run`     |
| `go.mod`                                 | Go                | Standard go toolchain                                              |
| `pom.xml`                                | Java/Maven        |                                                                    |
| `build.gradle` or `build.gradle.kts`     | Java/Gradle       |                                                                    |
| (none of the above)                      | unknown           | Skip proposals — ask the user manually for each command in step 10 |

For each detected stack, propose sensible defaults across these five
fixed slots: **Compile/type-check**, **Format**, **Lint**, **Narrow
test**, **Full test**. Anchors (one short-form per stack):

- **Python+poetry** — `poetry run mypy .`, `poetry run black .`, `poetry run ruff check .`, `poetry run pytest <path>`, `poetry run pytest`
- **Rust** — `cargo check`, `cargo fmt`, `cargo clippy`, `cargo test <path>`, `cargo test`
- **Node/TS** — `tsc --noEmit`, `prettier --write .`, `eslint .`, `vitest run <path>` or `jest <path>`, `vitest run` or `jest`
- **Go** — `go vet ./...` or `go build ./...`, `gofmt -w .`, `go vet ./...`, `go test <path>`, `go test ./...`
- **Java/Maven** — `mvn compile`, `mvn spotless:apply` (or omitted), `mvn checkstyle:check` (or omitted), `mvn test -Dtest=<path>`, `mvn test`
- **Java/Gradle** — equivalent idiomatic commands (`gradle compileJava`, `gradle spotlessApply`, `gradle check`, `gradle test --tests <path>`, `gradle test`)

Compile/type-check may legitimately be `omitted` for languages without
static analysis (e.g., plain JS without `tsc --noEmit`). Still propose a
sensible default and let the user choose `omitted` in step 10.

The **Narrow test** slot must be expressed as a template that takes a
`<path>` argument. The **Full test** slot is the same idea without
`<path>`.

## 10. Build Commands

Confirm and write per-repo build commands.

1. **Confirm each command with the user.** For each repo, present the
   five proposed commands. The five fixed slot labels are
   `Compile/type-check`, `Format`, `Lint`, `Narrow test`, `Full test`.
   The user can accept, override, or set any slot to `omitted`.
2. **Validation rules.**
   - `Compile/type-check` may be `omitted` without comment.
   - The other four are expected, but the user may still omit any of
     them.
   - If `Full test` is set to `omitted`, emit a non-blocking warning
     explaining that this disables a key quality gate consumed by
     `develop-epic` and `fix-bug`. Do not block.
3. **Write `## Build Commands` to `apiary.md`** placed after
   `## Documentation Locations` (and before any later sections future
   skills may add). Use this exact format:

   ```
   ## Build Commands

   ### <relative-path>
   - **Compile/type-check**: `<command>` | omitted
   - **Format**: `<command>` | omitted
   - **Lint**: `<command>` | omitted
   - **Narrow test**: `<command-with-<path>>` | omitted
   - **Full test**: `<command>` | omitted
   ```

   Repeat the H3 block per repo in discovery order. Use the literal word
   `omitted` (no backticks) when omitted; backtick-quote real commands.
   Preserve every other `##` section verbatim.

4. **Idempotency on rerun.** Reuse the same compare-and-confirm pattern
   the skill uses for stores. If a `## Build Commands` section already
   exists, parse the existing per-repo blocks and use them as the
   proposal baseline (rather than the freshly-detected stack defaults).
   Per repo, per slot: ask only on differences or explicit re-review.
   Setting a slot to its current value is a no-op — don't rewrite the
   file with no semantic change.

5. **Generic language.** Don't hard-code backend-specific tool names;
   pick whatever file edit tool is available to update `apiary.md`.

## 11. Write apiary.md

Write `<project_root>/apiary.md` with these top-level sections, in this
order:

```
# Apiary Configuration

## Project Root
<absolute path>

## New Feature
<decision-tree shape, possibly degenerate — see below>

## New Bug
<decision-tree shape, possibly degenerate — see below>

## Documentation Locations
<Reference + Maintained subsections, "omitted" where skipped>

## Build Commands
<per-repo blocks from step 10>
```

### Decision-tree shape for `## New Feature` and `## New Bug`

The full shape (when a template was applied with one or more factors):

```
## New Feature

<plain-english rationale paragraph (optional)>

- <factor_name>: <current value>

When <factor_name> = <value>:
  - source_references: <resolver>
When <factor_name> = <other_value>:
  - source_references: none, interview user
```

The factor list captures the re-detected current value of each factor
(from step 4's resolution pipeline, or the persisted value when no
factor changed on rerun). The `When ...:` branches are copied verbatim
from the template. Both the matched branch and the alternate branches
are written — they document how the value would change if the
environment changed.

The degenerate shape (interview mode, no factors): the section
collapses to a single resolved branch, no factor list, no `When ...:`
lines. This satisfies the rule that no "ask user" placeholders persist
in apiary.md:

```
## New Feature

- source_references: <resolved value, e.g., "none, interview user" or "github resolver">
```

Both shapes are valid apiary.md content. The skill MUST handle both
cases when reading and writing.

### Documentation Locations shape

```
## Documentation Locations

### Reference
- **Engineering best practices**: docs/best-practices.md, docs/coding-standards.md
- **Test guide**: docs/testing-guide.md
- **Doc writing guide**: omitted

### Maintained
- **Contributor docs**: docs/architecture/
- **Customer-facing docs**: README.md, docs/user-guide.md
```

Multiple paths per category are written as comma-separated values on
the bullet line. Skipped categories are recorded as the literal word
`omitted` (no backticks).

### Path rules and idempotency

The `## Project Root` line is the only absolute path in the file. Any
other paths in the file — including the `### <relative-path>` headings
under `## Build Commands` and any paths under `## Documentation
Locations` — must be relative to that root.

If you are updating an existing `apiary.md`, edit only the sections
this skill owns (`## Project Root`, `## New Feature`, `## New Bug`,
`## Documentation Locations`, `## Build Commands`) and preserve any
other `##` sections verbatim — they belong to skills that haven't
been written yet.

# What comes next

Once setup is complete, point the user at `/new-feature` to start the
first feature in their project.
