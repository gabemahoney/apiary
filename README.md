# Apiary 🐝

Apiary is an opinionated agentic workflow for taking an idea from inception to working software.
It uses the [bees](https://github.com/gabemahoney/bees) ticket management system. Install that first.

## Install

### 1. Clone the repo

```bash
git clone https://github.com/gabemahoney/apiary ~/projects/apiary
```

### 2. Install the skills

Ask Claude Code to install the Apiary skills. You have two options — pick one.

**Option A — Global install (recommended for single-user machines).** Installs the skills into your user-level skills directory so every repo you work in can use them.

> "Install the Apiary skills from `~/projects/apiary/skills` globally into `~/.claude/skills`."

**Option B — Single-repo install.** Installs the skills into a specific project's `.claude/skills` directory so only that repo sees them. Useful if you want to try Apiary on one project without affecting anything else.

> "Install the Apiary skills from `~/projects/apiary/skills` into `<absolute path to target repo>/.claude/skills`."

In either case, Claude will copy each skill directory (`project-setup`, `new-feature`, `write-prd`, `write-srd`, `write-plan`, `write-epic`, `develop-feature`, `develop-epic`, `new-bug`, `fix-bug`, etc.) into the chosen skills directory without disturbing any other skills already installed there.

### 2b. Install the subagents

Apiary also ships custom subagents under `~/projects/apiary/agents/` that the execution skills (e.g. `/develop-epic`) dispatch to do code, test, doc, review, and PM work. Pick the same scope you picked for the skills.

**Option A — Global install (recommended for single-user machines).** Copy `~/projects/apiary/agents/*` to `~/.claude/agents/`.

> "Install the Apiary subagents from `~/projects/apiary/agents` globally into `~/.claude/agents`."

**Option B — Single-repo install.** Copy `~/projects/apiary/agents/*` to `<absolute path to target repo>/.claude/agents/`.

> "Install the Apiary subagents from `~/projects/apiary/agents` into `<absolute path to target repo>/.claude/agents`."

**Note:** Custom subagents load at session start. After installing or updating subagent definitions, run `/agents` in Claude Code (or restart the session) for the new subagents to be available — execution skills will hard-fail with an actionable message if any of the seven required subagent types (`engineer`, `test-writer`, `doc-writer`, `pm`, `code-reviewer`, `test-reviewer`, `doc-reviewer`) are missing at dispatch time.

The repo also ships project-setup templates under `~/projects/apiary/apiary-templates/`. To make them available globally, copy that directory to `~/.claude/apiary-templates/` (or copy individual files into an existing directory there). Templates are plain markdown — author your own or edit the shipped ones to fit your team.

### 3. Configure
Run `/project-setup` to bootstrap a project for the Apiary workflow. The command is interview-driven: it asks you a small set of questions and uses your answers to set things up. It is safe to re-run on an already-configured project — it detects what already exists, surfaces any drift, and only changes things you explicitly approve.

When the command finishes, you'll have:

- An `apiary.md` file at the project root that records your answers and acts as the manifest later skills read from.
- Three stores ready to hold work:
  - **Features** — tracks feature ideas through the requirements lifecycle, from initial draft to published.
  - **Plans** — tracks the planned work that delivers features, broken into nested levels of granularity, from draft to done.
  - **Bugs** — tracks defects from the moment they're filed through to resolution.

Each store is configured up front with status values appropriate to its lifecycle, so you don't have to think about state machines while you're working.

`/project-setup` also discovers the git repos involved in your project, handling both single-repo and multi-repo layouts. For each repo it detects the stack and proposes build commands (compile/type-check, format, lint, narrow test, full test) for you to confirm or override. The confirmed commands are stored per-repo in `apiary.md` so later skills can run them without guessing.

A few additional conveniences: `/project-setup` accepts a named template from `~/.claude/apiary-templates/` (e.g., `public-github`) to preseed the interview — templates are plain markdown you can author or customize. The interview also captures Documentation Locations, split into Reference docs (consulted by skills as guides) and Maintained docs (skills update these); categories you skip are recorded as `omitted`. Re-running `/project-setup` re-detects environment factors (e.g., GitHub visibility) and prompts before updating saved configuration, with no changes when nothing has drifted.


## Workflow

Every Apiary skill reads `apiary.md` first to discover the project's stores, repos, configured commands, and intake preferences.

### Feature
Run `/new-feature` to capture a new feature.
It is stored as a Feature ticket in the Features store with status `draft`.
`apiary.md` configures whether intake requests a source reference (e.g., a GitHub issue URL) so the ticket links back to the original report.

### Product Requirements Document
Run `/write-prd` with the Feature ticket id to flesh out a PRD.
The PRD is stored as a t1 Doc child of the Feature ticket.
> Tip: Run `/req-review` on the Feature ticket after the PRD is written, repeating until the feedback becomes trivial. `/req-review` walks through review feedback and promotes the docs and the Feature toward `ready`.

### Software Requirements Document
Run `/write-srd` with the Feature ticket id to flesh out an SRD.
The SRD is stored as a t1 Doc child of the Feature ticket.
> Tip: Run `/req-review` on the Feature ticket after the SRD is written, repeating until the feedback becomes trivial. `/req-review` walks through review feedback and promotes the docs and the Feature toward `ready`.

### Write Plan
Run `/write-plan` with the Feature ticket id to develop a plan for building the feature.

- The skill hard-gates on the Feature being `ready`. If the Feature is still `draft`, you'll be sent back to `/req-review` (or asked to manually mark the Feature `ready`) before planning can proceed.
- The Plan is stored in the Plans store, with its source reference pointing back at the Feature so `develop-feature` can find the Plan from the Feature later.
- `/write-plan` automatically chains into `/write-epic` for each Epic in dependency order, so a single command produces a fully decomposed Plan with Epics, Tasks, and Subtasks.

### Develop Feature
Run `/develop-feature` against a Feature ticket id to build the feature end-to-end.

- The Feature must already have a Plan. If it doesn't, run `/write-plan` first.
- The skill walks the Plan's Epics in dependency order, dispatching `/develop-epic` (see the `### Develop Epic` subsection below) for each one.
- Between Epics, you'll be prompted to continue or pause so you can inspect progress before the next Epic starts.
- At the end, the combined deferred review feedback from all Epics is presented to you in one place.
- Once the run is complete, the skill suggests marking the Feature `done` (user-driven) and running `/teardown-worktree` to merge.

> Tip:
> - Run `/configure-worktree` first to do the work in an isolated worktree (see Worktrees below)
> - Run your own tests
> - Run `/teardown-worktree` to merge back to main when the Feature is `done`

### Develop Epic
Run `/develop-epic` against an Epic in status `ready` to execute it end-to-end.

- Subagents (Engineer, Test Writer, Doc Writer, PM) execute the Epic's Tasks per the plan.
- After Tasks complete, a review cycle (`code-review`, `test-review`, `doc-review`) runs and any feedback is folded back in.
- Close-out lands a single commit for the whole Epic.

> Tip: Re-running `/develop-epic` on the same Epic resumes interrupted execution — the Epic stays `active` until close-out, so you can pick up where you left off.
> Tip: The `agents/` directory must be installed (see Install section "2b") and Claude Code must be reloaded (`/agents` or restart the session) before subagents are dispatchable.

### Worktrees
Apiary's worktree skills isolate Feature work in a git worktree (or a set of them) so you can iterate without disturbing the main checkout.

- **Multi-repo aware**: `/configure-worktree` reads `apiary.md` to discover the repos involved in the project and creates a worktree in each repo the Feature touches (you confirm the affected subset). `/teardown-worktree` merges and cleans up across all of them.
- **Feature-done gate**: `/teardown-worktree` refuses to merge until the Feature is `done`. If the Feature is still `active`, the skill asks you to promote it first — typically via `/develop-feature` close-out, or by manually marking it `done` after validating.
- **Generic spawning**: `/configure-worktree` describes intent; the LLM picks the actual launch mechanism — waggle MCP if installed, tmux otherwise, or any equivalent. There is no hard waggle dependency.

### Bugs
Bugs follow a two-step flow:

- Run `/new-bug` to file a Bug ticket in the Bugs store. The skill reads `apiary.md` and, when the New Bug section configures a source reference (e.g., a GitHub issue URL), prompts you for it so the ticket links back to the original report.
- Run `/fix-bug` with the Bug ticket to work it. It supports resume — interrupting and re-invoking the skill picks up from where it left off — and ends with a confirm-fix prompt so you verify the bug is actually resolved before the ticket is closed out.


## Advanced Configuration
Apiary provides default `/code-review`, `/test-review` and `doc-review` skills. Feel free to replace this in part 
or wholesale with your own guidelines.

**Note:** These guidelines are enforced across all repos.
Documents describing repo-specific guidelines should be defined in a repo-specific Claude.md file. 
`/project-setup` configures these repo-specific entries but does not modify the above-listed skills.

