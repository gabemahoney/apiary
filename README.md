# Apiary 🐝

Apiary is an opinionated agentic workflow for taking an idea from inception to working software.
It uses the [bees](https://github.com/gabemahoney/bees) ticket management system. Install that first.

## Install

### 1. Clone the repo

```bash
git clone https://github.com/gabemahoney/apiary ~/projects/apiary
```

### 2. Install the skills and agents

Ask Claude Code to install the Apiary skills and agent definitions. The two directories are a paired copy — always install both to the same scope. You have two options — pick one.

**Option A — Global install (recommended for single-user machines).** Installs into your user-level `.claude` directories so every repo you work in can use them.

> "Install the Apiary skills from `~/projects/apiary/skills` globally into `~/.claude/skills`, and the Apiary agents from `~/projects/apiary/agents` into `~/.claude/agents`."

**Option B — Single-repo install.** Installs into a specific project's `.claude` directories so only that repo sees them. Useful if you want to try Apiary on one project without affecting anything else.

> "Install the Apiary skills from `~/projects/apiary/skills` into `<absolute path to target repo>/.claude/skills`, and the Apiary agents from `~/projects/apiary/agents` into `<absolute path to target repo>/.claude/agents`."

In either case, Claude will copy each skill directory (`apiary-setup`, `idea`, `write-prd`, `write-srd`, `make-plan`, `hatch-epic`, `do-bee`, `fix-bug`, etc.) and each agent definition file (`apiary-engineer.md`, `apiary-code-reviewer.md`, etc.) into the chosen directories without disturbing any other skills or agents already installed there.

### Execution model

Apiary's execution skills spawn role players (Engineer, Test Writer, Doc Writer, Product Manager, and reviewers) as **named subagents** via the Agent tool. The document and planning skills (`write-prd`, `write-srd`, `req-review`, `make-plan`) likewise delegate their heavy work to named subagents (PRD Writer, SRD Writer, Requirements Reviewer, Plan Writer) — the calling agent handles user interaction and reports the results. All definitions live in the install's `.claude/agents/` directory (user-global or per-repo, mirroring the skill install). Subagents run in-process and inherit the calling agent's permission surface — no experimental feature flags required.

### 3. Configure
Run `/apiary-setup`


## Workflow

### Idea
Run `/idea` to jot down a new idea.
It will be stored as a bee in the Ideas Hive.

### Product Requirements Document
Run `/write-prd` with the Idea bee to flesh out a PRD.
The PRD will be stored as a child of the Idea bee.
> Tip: Run `/req-review` on the Idea bee after the PRD is written multiple times until the feedback becomes trivial

### Software Requirements Document
Run `/write-srd` with the Idea bee to flesh out an SRD.
The SRD will be stored as a child of the Idea bee.
> Tip: Run `/req-review` on the Idea bee after the SRD is written multiple times until the feedback becomes trivial

### Make Plan
Run `/make-plan` with the Idea bee to develop a plan for building the feature.
The plan will be stored as a bee in the Plans Hive.

### Do Bee
Run `/do-bee` with the Feature bee to build the feature.
It will spawn a full set of role subagents via the Agent tool to do the work.
> Tip:
> - Run `/configure-worktree` first to do the work in an isolated worktree
> - Run your own tests
> - Run `/teardown-worktree` to merge back to main

### Fix Bug
You can tell your LLM to file a bug in the Bugs Hive, no skill needed.
Run `/fix-bug` with the bug in the Bugs Hive to fix.
It will spawn a smaller set of role subagents via the Agent tool to do the work.


## Advanced Configuration
Apiary provides default `/code-review`, `/test-review` and `doc-review` skills. Feel free to replace this in part 
or wholesale with your own guidelines.

**Note:** These guidelines are enforced across all repos.
Documents describing repo-specific guidelines should be defined in a repo-specific Claude.md file. 
`/apiary-setup` configures these repo-specific entries but does not modify the above-listed skills.

