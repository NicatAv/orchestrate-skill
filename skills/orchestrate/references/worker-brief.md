# Worker brief template

Fill this in and pass it as the `prompt` of `start_session`. The worker has not seen the
conversation, so the brief must stand on its own: include the spec excerpts and contracts it
needs, not "see the plan". Keep it under ~60 lines; put file paths and identifiers in backticks.

```
You are worker {ID} in a coordinated multi-session job "{TITLE}". A coordinator session started
you and will collect your branch. Other workers handle other parts in parallel.

## Your task
{GOAL — 2-5 sentences, concrete}

## Scope you own
You may create and edit only: {OWNED PATHS}.
If you need a change anywhere else (shared types, root configs, lockfiles, other modules), don't
make it — report BLOCKED with the exact change you need. The coordinator decides.

## Context
{SPEC EXCERPTS: relevant acceptance criteria, constraints, decisions}
{CONTRACTS: API shapes / types / schema this task must follow, or where T0 committed them}
{CONVENTIONS: build and test commands, code style notes worth repeating}

## Shared resources (other workers run at the same time)
{RULES FROM shared_resources THAT CONCERN THIS TASK, e.g.:
 - Migration numbers: use only V15–V17.
 - Don't commit generated `{FILE}`; the coordinator regenerates it.
 - Dev server: port {PORT} only.
 - Run module tests `{CMD}`; don't run the full build — the coordinator does.}

## Done means
- {VERIFIABLE CRITERION 1}
- {VERIFIABLE CRITERION 2}
- Run: `{TEST / BUILD COMMANDS}` — all pass.

## Rules
- Work only in your own worktree and branch. When finished, commit your work to your branch
  with clear messages; that is how the coordinator collects it. Don't push, open PRs, or merge
  other branches. Don't rename or switch your branch.
- You are a leaf worker: don't start, propose or hand off other sessions, and don't split this
  task further. If it turns out too big for one session, report BLOCKED with a proposed split.
- Don't add dependencies unless this brief lists them.
- Don't ask the user for decisions this brief already covers. If truly blocked, stop and report
  BLOCKED with one precise question.
- Follow the user's approval boundaries: nothing destructive, no deploys, no installs of
  system-wide tools.

## Status report (the coordinator parses this — keep it exact)
Start your end-of-turn recap AND your final message with exactly one status line:
  [{ID}] DONE 100% — <one-line result>
  [{ID}] IN_PROGRESS <n>% — <done so far / next step>
  [{ID}] BLOCKED — <one precise question>
For DONE, follow with: your branch name (`git branch --show-current`), files changed, commands
run with pass/fail, known gaps.
The status line must come first: the coordinator only sees the first ~400 characters.
```
