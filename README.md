# Apiary

A collection of Claude Code skills for structured software development workflows.

## Skills

| Skill | Description |
|-------|-------------|
| `apiary-setup` | Configure hives for Apiary workflow |
| `code-review` | Review changed files after task completion |
| `configure_worktree` | Launch Claude in isolated tmux session to work on a Bee asynchronously |
| `do-bee` | Proceed through each Epic in a Bee, doing the work described therein |
| `doc-review` | Review documentation completeness after task completes |
| `fix-bug` | Fix a bug described in a Bee ticket |
| `hatch-epic` | Break down a single Epic into Tasks |
| `hatch-feature` | Read Idea Bee and create a Feature Bee with Epics |
| `idea` | Create a persistent file describing a new idea |
| `req-review` | Review requirements for completeness |
| `teardown_worktree` | Merge and clean up a git worktree |
| `test-review` | Review test files for quality and coverage |
| `write-prd` | Write a Product Requirements Document for a development effort |
| `write-srd` | Write a System Requirements Document |

## Install

```bash
git clone https://github.com/gmahoney/apiary ~/projects/apiary
ln -s ~/projects/apiary/skills ~/.claude/skills
```

Or use the dotfiles `setup.sh` which handles this automatically.
