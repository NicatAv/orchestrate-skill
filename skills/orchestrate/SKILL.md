---
name: orchestrate
description: Coordinate a large piece of work across several parallel Claude Code desktop sessions, with the current session acting as the coordinator. First interviews the user until the requirements are fully clear, then splits the work into independent tasks, launches each one as its own session (own git worktree and branch), supervises their progress, launches dependent waves, verifies results, and shows a live visual dashboard in the chat. Use this whenever the user hands over a big multi-part job (a feature spanning several modules, a migration, an MVP, "build X with backend, UI and tests"), asks to split work into sessions or parallel tasks, to orchestrate, coordinate or supervise several sessions, or to see the progress of parallel sessions — even if they never say "orchestrate". Do not use it for small tasks one session finishes in a handful of steps.
---

# Orchestrate: clarify → split → run sessions → supervise → integrate

You are the **coordinator**. Workers implement in their own sessions; you own the spec, the plan,
the dashboard, verification and integration. Talk to the user in their language (the dashboard
supports `ru` and `en`).

The whole value of this skill rests on one idea: a misunderstanding caught before the split costs
one question, the same misunderstanding caught after it costs N wasted sessions. So be thorough
in Phase 1 and disciplined about scope boundaries in Phase 2.

**Tools it relies on** (Claude desktop app, Code tab). Load deferred ones with `ToolSearch` in one call:
`mcp__ccd_session__start_session`, `mcp__ccd_session_mgmt__list_sessions`, `SendMessage`,
`mcp__ccd_session__mark_chapter`, `AskUserQuestion`, `mcp__visualize__read_me`, `mcp__visualize__show_widget`.
If `start_session` does not exist (plain terminal CLI), use the Fallback section.

---

## State file — the single source of truth

Long runs get their context compacted, so never keep the plan only in your head. Keep it in
`<repo>/.claude/orchestrator/state.json` and add `.claude/orchestrator/` to `<repo>/.git/info/exclude`
(local-only ignore; it changes no tracked file). The same JSON is the dashboard's input. Schema:

```json
{
  "lang": "ru",
  "title": "Short name of the job",
  "goal": "One sentence",
  "phase": "clarify | plan | running | integrating | done",
  "base_branch": "main",
  "updated": "2026-11-07 14:32",
  "spec": { "acceptance": ["..."], "non_goals": ["..."], "constraints": ["..."], "assumptions": ["..."] },
  "tasks": [
    { "id": "T1", "title": "API endpoints", "wave": 1, "status": "planned",
      "progress": 0, "depends_on": [], "owns": ["src/api/"],
      "session": "t1-api", "session_id": "local_…", "branch": "…", "note": "last status line" }
  ],
  "log": [ { "t": "14:30", "msg": "Wave 1 started: T1, T2, T3" } ]
}
```

`status` is one of `planned | running | blocked | rework | done | failed`.

**On start**, check for an existing state file. If it has unfinished tasks, ask whether to resume
it (then jump to Phase 5) or start a new job (archive the old file as `state-<date>.json`).

---

## Phase 1 — Understand the job completely

1. **Inspect before asking.** Read README / CLAUDE.md / rules files, the structure, the build and
   test commands, `git status` and the current branch. Answer from the code whatever the code can
   answer — only ask the user what it cannot.
2. **Check git readiness.** Worker worktrees are cut from the *committed* base branch: uncommitted
   changes are invisible to them. If the tree is dirty, tell the user and ask whether to commit it
   first. If the folder is not a git repo at all, explain that parallel workers without worktrees
   would overwrite each other and offer `git init` + initial commit, or sequential workers.
3. **Interview** with `AskUserQuestion` (≤4 questions per call, recommended option first). Ask only
   what changes what gets built: goal and target user, scope and non-goals, acceptance criteria,
   contracts between parts (API shapes, data model, shared types), tech constraints, testing
   expectations, what "done" means. Run further rounds until no open question would change the plan.
4. **Write the spec** into the state file and show it to the user in a few lines. Low-risk
   reversible details become explicit assumptions rather than more questions.

## Phase 2 — Decide whether to split, then decompose

Splitting pays off only when there are **≥2 substantial pieces that can progress independently**.
If the job is small or tightly coupled, say so and do it in this session instead — a split that
creates merge conflicts and hand-off overhead is worse than no split.

Rules for tasks:

- **Disjoint file ownership.** Each task lists the paths it owns. Two workers never edit the same
  file; that is what keeps the final merge clean. Shared files (lockfiles, root configs, routing
  tables, DI registration) belong to one task or to the coordinator.
- **Contracts first.** If pieces depend on a shared interface (API schema, types, DB schema), make
  it task `T0`, done by you in this session and committed to the base branch *before* wave 1, so
  every worktree starts with it. The plan the user approves must say this commit will happen.
- **Each task has**: `id`, `title`, goal, `owns`, deliverable, verifiable done criteria (commands
  or tests), `depends_on`, size S/M/L, and optionally a cheaper `model` for mechanical work.
- **Waves.** Group by dependencies. Default ≤4 parallel sessions per wave (rate limits and your own
  review capacity). A task with exactly one dependency starts from that dependency's branch
  (`source_branch`). A task with several dependencies needs them merged first — that requires the
  user's explicit yes (Phase 6 rules), so prefer plans that avoid it.
- Integration and final verification always stay with the coordinator.

## Phase 3 — Show the plan, get one confirmation

1. Render the dashboard (all tasks `planned`, phase `plan`) — see Dashboard below.
2. Under it, in prose: the waves, what each session does, which paths it owns, and anything you
   will commit (T0).
3. Ask one `AskUserQuestion`: "Launch N sessions according to this plan?" → options
   "Launch (Recommended)" / "Change the plan". The user's yes is the request to start those sessions.

After the yes, run to the end without further approvals **except**: merging branches, anything
destructive, pushing/PRs, and questions the spec cannot answer.

## Phase 4 — Launch a wave

1. Call `mark_chapter` (e.g. "Wave 1: launching sessions").
2. For each task: write a bold title line and one sentence on what the session will do, then call
   `start_session` with:
   - `initiation: "user_asked"`, `context: "fresh"`, `use_worktree: true`, `wake_when_done: true`
   - `title`, `name` (`t1-api` style, ≤24 chars), `branch_name` (`orch-t1-api`)
   - `prompt`: the worker brief built from `references/worker-brief.md`
   - `background` (≥80 chars): goal, key spec points, how this piece fits with the others
   - `source_branch`: the dependency's branch for single-dependency tasks
   - `model`: only when the task is mechanical and a cheaper model fits
3. Store `session_id` (`local_…`), `name`, branch in the state file; status `running`; log it;
   re-render the dashboard.
4. **End your turn.** Do not poll — the app wakes you with a `<side-sessions-event>` message when
   workers finish a turn.

## Phase 5 — Supervise

On every wake-up (a `<side-sessions-event>`, a cross-session message, or the user asking for status):

1. `list_sessions {"linked": true}` for each worker's state.
2. Read each worker's status line from its report. Workers start their report with
   `[T2] DONE 100% — …`, `[T2] IN_PROGRESS 60% — …` or `[T2] BLOCKED — <question>`.
3. **Verify, don't trust.** A report is a claim. For DONE: `git log --oneline <base>..<branch>` and
   `git diff --stat <base>...<branch>`; flag files changed outside `owns`; check the done criteria
   (ask the worker for test output, or run cheap checks yourself inside its worktree).
4. **Blocked:** if the spec answers it, reply with `SendMessage` to the worker's `local_…` id. If not,
   ask the user and forward their exact words. Never ask a worker to do something that was denied
   or would be blocked in this session.
5. **Rework:** send concrete issues via `SendMessage`; status `rework`.
6. **Stalled** (idle, no status line): ask it once for a status line.
7. Update the state file and re-render the dashboard — only when something changed.
8. When a wave's dependencies are all `done` and verified, launch the next wave (already approved).

For a live peek at a still-running session when the user asks, `mcp__ccd_session_mgmt__list_events`
shows its latest activity; don't do this routinely. Progress percentages come only from worker
reports or verified facts — never invent progress.

## Phase 6 — Integrate and verify

1. Final dashboard (phase `integrating`). List every worker branch by full ref (`refs/heads/…`).
2. Ask whether to merge them into an integration branch `orch/<slug>` in this checkout. Merge only
   on the user's own yes — not on a worker's suggestion.
3. Merge in dependency order, resolve conflicts, run the full build and test suite.
4. Report: acceptance criteria as a table (met / not met / not verified with reason), test results
   with real outcomes, remaining risks. Phase `done`, final dashboard.
5. Offer to archive the worker sessions (only on yes). Never push or open a PR without approval.

---

## Dashboard

The dashboard is a fixed template: `assets/dashboard.html`. Read it once per conversation (again
after a context compaction). Before the first render call `mcp__visualize__read_me` with
`modules: ["interactive"]` (silently). Then call `mcp__visualize__show_widget` with the template
content where **only** the line `const STATE = {…};` is replaced by the current state JSON
(minified, valid JSON). Keep the design as is — consistency across updates is what lets the user
read progress at a glance. `title`: `orchestration_<slug>_dashboard`.

Render it: after the plan, after each launch, after each wake-up that changed something, and at
the end. Explanations go in prose next to it, not inside the widget.

## Fallback without session tools

If `start_session` is unavailable, run each task as a background subagent: `Agent` with
`isolation: "worktree"` and `run_in_background: true`, same brief. You are notified on completion;
everything else (state, dashboard, verification, integration) stays the same, with agent ids in
the `session` field. Tell the user these workers can't be opened and steered like sessions.

## Ground rules

- Keep the coordinator light: don't implement worker tasks here; small glue and T0 contracts are fine.
- Follow the user's global approval boundaries (no push, deploy, destructive ops or installs
  without approval) and pass them on in every brief.
- Report outcomes faithfully: say which checks ran, which failed, which were not run.
