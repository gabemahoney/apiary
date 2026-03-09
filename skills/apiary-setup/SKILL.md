---
name: apiary-setup
description: Configure hives for Apiary workflow
---

## Overview

This skill should be run once to configure a repo for the Apiary workflow. It can also be run to fix hives that
have gotten misconfigured.

## Valid configuration

The repo must have the following hives available and configured with these child tiers and valid status values:

### Features Hive
Child tiers:
- t1 — Epic / Epics
- t2 — Task / Tasks
- t3 — Subtask / Subtasks

Status values:
- larva — not fully documented, not ready to work
- pupa — fully documented, ready to work
- worker — in progress
- finished — done

### Bugs Hive
Child tiers:
none

Status values:
- open — open bug
- finished — done

### Ideas Hive
Child tiers:
- t1 - Doc / Docs

Status values:
- larva — not fully documented, not ready to work
- pupa — fully documented, ready to work
- worker — in progress
- finished — done

## Instructions

Check for the existence of the above hives and validate their configs.
If any hives are missing:
- Ask the user if you can create them and if so where they should reside.
- Colonize the hive with no optional repo_root (unless the user instructs you to)
- Configure the hive with valid child tiers and status values

If a hive exists:
- Validate its child tiers and status values
- If they differ from above, ask user if you may change them to the values listed above.
