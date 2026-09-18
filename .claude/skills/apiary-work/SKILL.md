---
name: apiary-work
description: Apiary self-modification workflow — work an apiary ticket, fix an apiary bug, or implement an apiary bee as a single agent. Replaces do-bee/fix-bug when the target repo is apiary itself. Takes a bees ticket ID, dispatches one worker, and gates push behind operator sign-off.
---

# apiary-work

Single-agent execution of a bees ticket against the apiary repo itself. No
engineer/reviewer/doc-writer team — one worker implements end-to-end, and you
(the calling agent) run a review loop with the operator before anything is
pushed.

## Non-goals

- No multi-agent subagent team (engineer, reviewers, doc writer).
- No pre-dispatch model validation — the general-purpose Agent inherits
  harness defaults.
- No PRD/SRD generation — that stays in `write-prd` / `write-srd`.

## Steps

### 1. Input

Take a bees ticket ID (a bee ID like `b.448` or a child-tier ID). If the user
didn't supply one, ask.

### 2. Gather context

- Fetch the ticket: `bees show-ticket --ids <id>`.
- Scan the ticket body for linked PRD/SRD/design-doc ticket IDs and fetch
  those too.
- Read the repo state relevant to the ticket so the dispatch prompt reflects
  reality, not just the ticket's claims.

### 3. Dispatch one worker

Spawn ONE foreground `Agent` (no `subagent_type` — general-purpose) with a
self-contained prompt containing:

- the ticket body verbatim
- any PRD/SRD contents verbatim
- relevant repo conventions
- instructions to: implement the change, run any local checks (grep-based
  acceptance criteria, etc.), commit locally on `master`, and STOP before
  pushing
- required report format: files changed, commit SHA, one-paragraph summary,
  any open questions or concerns it chose to set aside

### 4. Review loop (calling agent = operator-facing)

On worker return:

1. Show the operator `git show --stat <sha>` and `git diff <sha>^..<sha>`.
2. Present the worker's summary and open questions.
3. Wait for explicit operator sign-off — do not proceed without it.
4. If the operator requests changes: `SendMessage` back to the SAME worker
   (its context is preserved) with the change request. The worker adds fixup
   commits and reports again. Repeat until sign-off.

### 5. Publish

On sign-off, instruct the worker to:

- `git push origin master`
- `bees update-ticket --ids <id> --status finished`

Confirm both to the operator.
