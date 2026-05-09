---
name: configure-worktree
description: Launch a Claude session in an isolated worktree for a Feature
---

# Overview

Set up isolated git worktree(s) for a Feature ticket and launch a Claude
session in each, so work proceeds asynchronously without disturbing the
user's main checkouts. The skill is multi-repo aware: it reads
`apiary.md` to discover candidate repos, lets the LLM (with user
confirmation) pick the subset relevant to the Feature, and creates one
worktree per affected repo.

# Usage

**Invocation patterns**:
- `/configure-worktree <feature-id>`
- "Launch async session for <feature-id>"
- "Make a workspace for <feature-id>"

# Steps

## Step 0: Read apiary.md

Locate `apiary.md` at the Project Root (the directory the user invoked
the skill from, walking upward until `apiary.md` is found, or use the
known Project Root if already established in context).

If `apiary.md` is missing, abort with:

```
apiary.md not found. Run `/project-setup` first to configure this project.
```

Otherwise, read `apiary.md` and extract:

- **Project Root** — base for path operations.
- **`## Build Commands` section** — every `### <relative-path>` heading
  under this section is a CANDIDATE repo. Resolve each relative path to
  an absolute path against the Project Root.

The candidate repo list drives Step 1.

## Step 1: Determine affected repos

The CANDIDATE list from `apiary.md` Build Commands is the universe of
possible repos. Do NOT auto-create worktrees in every candidate.

The LLM running this skill selects the SUBSET of candidates relevant to
this Feature, drawing on:

- The Feature ticket description (title + body).
- The parent Plan body, if reachable from the Feature.
- The Feature's PRD/SRD references (the t1 Doc children of the Feature
  ticket reachable via the Plan → Feature source-ref chain).
- Epic / Task descriptions if already loaded into context.

Use these signals to nominate a subset. If signals are ambiguous,
present the full candidate list and let the user pick.

**Always confirm with the user** before creating worktrees. Present
the proposed subset (formatted as a list of absolute paths) and let
the user confirm or adjust.

For single-repo projects (one `### <relative-path>` heading), still
confirm but default the subset to that single repo.

Persist the user-confirmed list of affected repo absolute paths in
working memory; subsequent steps iterate over it.

## Step 2: Per-repo worktree creation

Iterate over the user-confirmed affected-repos list. For each repo:

### Step 2a: Per-repo uncommitted-changes check

Run `git status` inside each affected repo. Collect the dirty repos.

If any repo has uncommitted changes, present a single per-repo summary
to the user:

```
The following repos have uncommitted changes:
- <repo-path-1>: <short summary>
- <repo-path-2>: <short summary>

Commit them before creating worktrees?
```

Options: "Yes, commit all" / "No, abort" / "Decide per repo".

If the user picks "Decide per repo", ask once per dirty repo. Apply the
chosen action; the worktree for each repo is created from that repo's
current HEAD afterward.

### Step 2b: Create the worktree

Normalize the ticket ID by replacing dots with underscores (e.g.,
`<feature-id>` `b3.hz` → directory name `b3_hz`).

For each affected repo, the worktree is a sibling directory to that
repo, sharing the same normalized-ticket-id directory name across repos:

```
worktree_path = {affected_repo_root}/../{normalized_ticket_id}
```

Determine the per-repo base branch — each repo may have its own default
(check `git remote show origin` HEAD or the repo's HEAD-tracking config;
do NOT assume `main` or `master` globally).

Create the worktree from the per-repo base branch:

```
git -C {affected_repo_root} worktree add {affected_repo_root}/../{normalized_ticket_id} <per-repo-base-branch>
```

Multi-repo example, two cases:

- **Repos share a parent.** If `/proj/repo-a` and `/proj/repo-b` are
  both affected and the normalized ticket id is `b3_hz`, the
  `{repo}/../{ticket_id}` rule places both worktrees at the same
  path `/proj/b3_hz` — a collision. In this case, suffix the
  worktree path with the repo name to disambiguate:
  `/proj/b3_hz-repo-a` and `/proj/b3_hz-repo-b`.
- **Repos live under different parents.** Each worktree sits
  beside its own source repo (`{repo}/../{ticket_id}`); the shared
  directory name `{normalized_ticket_id}` is fine because the
  parents differ.

Detect the collision case before creating worktrees and pick the
disambiguating form when needed.

### Step 2c: Propagate auto_approve.sh per repo

`auto_approve.sh` is gitignored and must be copied into each new
worktree. For each affected repo, look up the source-of-truth
`auto_approve.sh`:

1. First, try the sibling "main checkout for this repo" — the
   conventional sibling directory holding the user's primary working
   copy of this repo. If a directory matching that convention exists
   beside the repo and has an `auto_approve.sh`, copy it.
2. Otherwise, fall back to the affected repo root itself:
   `cp {affected_repo_root}/auto_approve.sh {worktree_path}/auto_approve.sh`.

If neither source has the file, skip silently for that repo — the
worktree just won't have one.

`docker/`, `.claude/`, and any other tracked support files are present
automatically because they live in git.

## Step 3: Ask the user for the agent's prompt

Ask the user for the full instruction string the spawned Claude session
will receive on launch. This can be a slash-command
invocation (typically `/develop-feature <feature-id>`), a freeform task
description, or any valid prompt.

## Step 4: Ask the user for model preference

Ask which Claude model to use for the session(s) — Opus or Sonnet (or
whichever models the host environment offers). Pass the user's choice
through to whichever launch mechanism is selected in Step 5.

## Step 5: Launch a Claude session in each affected worktree

Launch a Claude session in the affected worktree(s) to begin work on
this Feature. **Pick the available mechanism** — for example, the
waggle MCP server if installed, a direct `tmux new-session -d`
invocation otherwise, or any equivalent backgrounding mechanism the
host environment provides. The skill body is deliberately silent on
which specific tool to call; the LLM picks based on what's available.

Inputs the LLM must wire through to whichever mechanism it picks:

- **Worktree absolute path** — the per-repo worktree dir from Step 2b.
- **Session label** — the normalized ticket ID. If launching one
  session per repo, suffix with the repo identifier
  (`<ticket-id>-<repo-name>`).
- **Agent identity** — Claude.
- **Model** — the user's choice from Step 4.
- **Initial prompt / command** — the prompt from Step 3 (typically
  `/develop-feature <feature-id>`).

### Single-session vs one-per-repo

Default: launch a single primary session in one designated **primary
worktree**. The LLM picks the primary repo based on Feature signals
(the repo most touched by the Feature description / Plan / PRD). If the
signals are ambiguous, ask the user. The spawned session is expected to
navigate between sibling worktrees as Tasks dictate.

Alternative: if the user prefers, launch one session per affected
worktree. Ask the user to confirm when more than one repo is affected.

## Step 6: Report session status

After launch, report the status of each session using whichever
mechanism was used to launch it (waggle's session-list tool if waggle
was used; `tmux list-sessions` if tmux was used; the equivalent for any
other mechanism). Confirm each launched session is alive.

## Step 7: Per-repo summary report

Provide a final per-repo summary:

```
Worktrees created:
- {affected_repo_1}: {worktree_path_1}
- {affected_repo_2}: {worktree_path_2}
  ...

Sessions launched:
- {session_label_1} in {worktree_path_1}
  Attach hint: <mechanism-specific> (e.g., `tmux attach -t <session>`
  if launched via tmux; consult waggle's docs if launched via waggle;
  otherwise consult the chosen mechanism's docs)
- ...

Ticket: <feature-id>
Initial prompt: <user-prompt>

Note: when this Feature reaches `done`, run `/teardown-worktree` to
merge and clean up across all of these repos.
```
