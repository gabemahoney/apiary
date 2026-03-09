# Apiary 🐝 

Apiary is an opinionated agentic workflow for taking an idea from inception to working software.  
It uses the [bees](https://github.com/gabemahoney/bees) ticket management system. Install that first.

## Install

### Clone Repo

```bash
git clone https://github.com/gmahoney/apiary ~/projects/apiary
ln -s ~/projects/apiary/skills ~/.claude/skills
```

### Configure
Run `/apiary-setup`

## Workflow

### Idea
Run `/idea` to jot down a new idea.  
It will be stored as a bee in the Idea hive.

### Product Requirements Document
Run `/write-prd` with the Idea bee to flesh out a PRD.  
The PRD will be stored as a child of the Idea bee.  
> Tip: Run `/req-review` on the idea bee after the PRD is written multiple times until the feedback becomes trivial

### Software Requirements Document
Run `/write-srd` with the idea bee to flesh out an SRD  
The SRD will be stored as a child of the Idea bee.  
> Tip: Run `/req-review` on the idea bee after the SRD is written multiple times until the feedback becomes trivial

### Hatch Feature
Run `/hatch-feature` with the Idea bee to develop a plan for building the feature.  
The feature plan will be stored as a bee in the Features hive.

### Do Bee
Run `/do-bee` with the Feature bee to build the feature.  
It will create a full Claude Team to do the work.  
> Tip:
> - Run `/configure-worktree` with the Feature bee to do the work in a worktree
> - Run your own tests
> - Run `/teardown-worktree` to merge back to main

### Fix Bug
You can tell your LLM to file a bug in the Bugs hive, no skill needed.  
Run `/fix-bug` with the bug in the Bugs hive to fix.  
It will create a smaller Claude Team to do the work.  


## Advanced
The following skills are run by agents as part of the workflow above.  
- `hatch-epic`
- `code-review`
- `test-review`
- `doc-review`
