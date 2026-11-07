# orchestrate — a Claude Code skill for multi-session work

Hand Claude Code a big job and let the current session become a **coordinator**:

1. **Clarify** — it inspects the repo, then interviews you until nothing that would change the result is left open, and writes a short spec (goal, acceptance criteria, non-goals).
2. **Split** — it decomposes the work into tasks with disjoint file ownership, groups them into dependency waves (≤4 parallel by default), and puts shared contracts (API, types, schema) first.
3. **Confirm once** — you see the plan as a dashboard and answer one question: launch?
4. **Run** — every task starts as its own Claude Code desktop session in its own git worktree and branch, with a self-contained brief and a fixed status protocol (`[T2] DONE 100% — …`).
5. **Supervise** — the coordinator is woken when workers finish a turn (no polling), verifies their branches with `git log` / `git diff`, answers blockers from the spec or asks you, launches the next wave and redraws the dashboard.
6. **Integrate** — merges only with your explicit yes, runs the build and tests, reports every acceptance criterion.

The dashboard renders inline in the chat: summary metrics, overall progress, task rows per wave with status, progress, session and branch, and recent events. Labels in Russian or English.

## Requirements

- The **Claude desktop app, Code tab** — the skill uses its session tools (`start_session`, `list_sessions`, cross-session messages) and inline widgets. In the plain terminal CLI it falls back to background subagents in worktrees.
- A **git repository** with the base work committed (worker worktrees are cut from committed history).

## Install

**As a plugin (marketplace):**

```bash
claude plugin marketplace add NicatAv/orchestrate-skill
claude plugin install orchestrate@orchestrate-skill
```

**Manually:** copy `skills/orchestrate` to `~/.claude/skills/orchestrate` (all projects) or to `<project>/.claude/skills/orchestrate` (one project).

## Use

```text
/orchestrate Add an orders module: REST API, Compose list screen, DB migration and offline sync
```

Or just describe a large multi-part task — the skill's description lets Claude pick it up on its own.

## Layout

```
skills/orchestrate/
├── SKILL.md                    # the workflow
├── references/worker-brief.md  # brief template sent to each worker session
└── assets/dashboard.html       # dashboard template; only the STATE line changes
```

Run state is kept in `<repo>/.claude/orchestrator/state.json` (excluded locally via `.git/info/exclude`), so a long run survives context compaction and can be resumed.

## Safety

The coordinator never merges, pushes, opens PRs, deletes or installs anything without your approval, and passes the same boundaries to every worker.

## License

MIT
