# Apiary 🐝

Apiary is an opinionated agentic workflow for taking an idea from inception to working software.
It uses the [bees](https://github.com/gabemahoney/bees) ticket management system. Install that first.

## Install

### 1. Clone the repo

```bash
git clone https://github.com/gabemahoney/apiary ~/projects/apiary
```

### 2. Install the skills

Ask Claude Code to install the Apiary skills into your user skills directory. For example:

> "Install the Apiary skills from `~/projects/apiary/skills` into `~/.claude/skills`."

Claude will copy each skill directory (`apiary-setup`, `idea`, `write-prd`, `write-srd`, `make-plan`, `hatch-epic`, `do-bee`, `fix-bug`, etc.) into `~/.claude/skills` without disturbing any other skills already installed there.

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
It will create a full Claude Team to do the work.
> Tip:
> - Run `/configure-worktree` first to do the work in an isolated worktree
> - Run your own tests
> - Run `/teardown-worktree` to merge back to main

### Fix Bug
You can tell your LLM to file a bug in the Bugs Hive, no skill needed.
Run `/fix-bug` with the bug in the Bugs Hive to fix.
It will create a smaller Claude Team to do the work.


## Advanced Configuration
Apiary provides default `/code-review`, `/test-review` and `doc-review` skills. Feel free to replace this in part 
or wholesale with your own guidelines.

**Note:** These guidelines are enforced across all repos.
Documents describing repo-specific guidelines should be defined in a repo-specific Claude.md file. 
`/apiary-setup` configures these repo-specific entries but does not modify the above-listed skills.

