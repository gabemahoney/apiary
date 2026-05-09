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

In either case, Claude will copy each skill directory (`project-setup`, `new-feature`, `write-prd`, `write-srd`, `make-plan`, `hatch-epic`, `do-bee`, `new-bug`, `fix-bug`, etc.) into the chosen skills directory without disturbing any other skills already installed there.

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
The plan is stored in the Plans store.
> Note: `/write-plan` is not yet shipped — a later Epic delivers it. The README documents the shipping flow.

### Do Bee
Run `/do-bee` with the Feature bee to build the feature.
It will create a full Claude Team to do the work.
> Tip:
> - Run `/configure-worktree` first to do the work in an isolated worktree
> - Run your own tests
> - Run `/teardown-worktree` to merge back to main

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

