---
name: project-setup
description: One-time configuration of a project for the Apiary workflow
---

# Overview

Run this skill once per project to bootstrap Apiary: detect the available
ticket-backend, create the three stores Apiary expects (Features, Plans, Bugs),
configure their status values, and write an `apiary.md` manifest at the
project root.

Re-running the skill in an already-configured project is safe — it detects
existing state, surfaces drift to the user, and changes nothing without
explicit approval.

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
user which to use with `AskUserQuestion`.

# Steps

## 1. Determine the project root

Ask the user for the absolute path of the project root (the directory that
will own `apiary.md`). Suggest the current working directory as the default
but require confirmation. Repo discovery and any automatic root inference
are out of scope for this skill — a later setup phase will add that.

## 2. Check for an existing apiary.md

If `<project_root>/apiary.md` already exists, read it and treat this run
as an idempotent re-configuration:

- Parse the existing `## Project Root`, `## New Feature`, and `## New Bug`
  sections.
- Compare the parsed values to whatever the user provides in step 4.
- For any drift, use `AskUserQuestion` to confirm before overwriting.
- If nothing differs, report "already configured" and exit cleanly.

Treat the level-2 (`##`) headings as section markers. Preserve any unknown
sections you encounter — future skills will append more sections, and this
skill must not clobber them.

## 3. Discover existing stores

Query the backend for stores named `Features`, `Plans`, and `Bugs` (case
sensitivity follows the backend's own rules). For each:

- If missing, you will create it in step 5.
- If present, fetch its tier configuration and status values, and compare
  to the target configuration in step 5. Record any drift.

## 4. Interview the user

Use `AskUserQuestion` for choice prompts. Collect the minimum needed to
populate `apiary.md`:

- **Project root** — confirm or correct the path from step 1.
- **New Feature → source_references** — one of:
  - `github resolver` — the backend will resolve GitHub URLs/IDs itself.
  - `none, interview user` — the skill that creates a feature will ask
    the user for source material instead.
- **New Bug → source_references** — same two choices as above.

Resolve each value to a concrete string. Do not write `ask user` or any
other placeholder into `apiary.md` — the answer is captured here, in this
interview.

This skill intentionally does NOT ask about repo discovery, build
commands, project templates, documentation locations, or decision-tree
logic for source references. Those belong to later setup phases and will
be added by future skills as additional sections in `apiary.md`.

## 5. Create or reconcile the three stores

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
- If it exists with different tiers or statuses, present the diff via
  `AskUserQuestion` and only change what the user approves.

Use whichever backend tools and CLI are available; the skill stays
backend-agnostic so the LLM running it picks the right calls.

## 6. Repo Discovery

Find the git repos that belong to this project. They drive the per-repo
build commands written in step 8.

1. Scan the project root for `.git` directories, walking 1–2 levels deep.
   Skip vendored directories: `node_modules`, `.venv`, `target`, `dist`,
   `build`, `.tox`.
2. Branch on what you found:
   - **One or more repos found.** List the relative paths to the user,
     then `AskUserQuestion`: "Does this project touch any other git
     repos not listed?" If yes, ask for additional paths (absolute or
     project-root-relative) and add them to the set.
   - **No repos found.** `AskUserQuestion`: "Does your project use one
     or more git repos?" If yes, ask for paths and add them. If no,
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
Commands` section in `apiary.md`. Use `AskUserQuestion` before adding
newly-found repos or removing repos that no longer exist.

## 7. Stack Detection

For each repo path collected in step 6, inspect manifest files at the
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
| (none of the above)                      | unknown           | Skip proposals — ask the user manually for each command in step 8 |

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
sensible default and let the user choose `omitted` in step 8.

The **Narrow test** slot must be expressed as a template that takes a
`<path>` argument. The **Full test** slot is the same idea without
`<path>`.

## 8. Build Commands

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
3. **Write `## Build Commands` to `apiary.md`** placed after `## New
   Bug` (and before any later sections future skills may add). Use this
   exact format:

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

## 9. Write apiary.md

Write `<project_root>/apiary.md` with these top-level sections, in this
order:

```
# Apiary Configuration

## Project Root
<absolute path>

## New Feature
- source_references: <resolved value from interview>

## New Bug
- source_references: <resolved value from interview>

## Build Commands
<per-repo blocks from step 8>
```

The `## Project Root` line is the only absolute path in the file. Any
other paths in the file — including the `### <relative-path>` headings
under `## Build Commands` — must be relative to that root.

If you are updating an existing `apiary.md`, edit only the sections this
skill owns (`## Project Root`, `## New Feature`, `## New Bug`, `## Build
Commands`) and preserve any other `##` sections verbatim — they belong
to skills that haven't been written yet.

# What comes next

Once setup is complete, point the user at `/new-feature` to start the
first feature in their project.
