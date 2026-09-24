---
name: agent-javis
description: "Use when the user hands over a task list (or keeps adding tasks) and wants the current session to orchestrate sub agents, or explicitly asks for a council / multi-perspective debate on a question (council mode: 2 rounds, then the orchestrator synthesises and can turn the conclusion into tasks). Remembers the sub agent model, spawns low-effort worker agents (single lane by default, parallel only when clearly safe), workers self-check, the orchestrator reviews against a recorded base commit, stops for approval before deploy/prod/DB/deletes, short progress reports."
---

# agent-javis — the current session orchestrates, sub agents do the work

## Step 0: before starting

1. **Mode**: a task list means execution mode. Council mode only when the user explicitly asks for a council, a debate or multiple perspectives. An ordinary "review this" or "evaluate that" is just done directly, not as a council. Ask only if you genuinely can't tell.
2. **Model**: read `~/.claude/skills/agent-javis/last-model.txt` (one of `opus` / `sonnet` / `haiku` / `fable`). Use it if present; otherwise use `opus`. **Don't ask.** State it in your first line, e.g. "Execution mode, sub agents on Opus — say so if you want a different model." If the user switches, write the new model back to the file; it takes effect at the next task boundary (see "Roles").
3. **Load deferred tools in one go**: a single `ToolSearch` with `select:SendMessage,TaskStop,CronCreate,CronDelete,PushNotification`. Anything that fails to load is unavailable; follow the fallback table.
4. **Task tracking**: use `TaskCreate` / `TaskUpdate` / `TaskList` if available; otherwise keep a checklist in `javis-queue.md` in the scratchpad. Each lane records: status (`active` / `paused` / `cancelled`), current task, worker ID, retry count, working directory, base commit.

## Environment and fallbacks

Decide from the tools that actually loaded in this session, not from which product you think you're in.

| Missing tool | Fallback |
|---|---|
| `Agent` | No sub agents: the orchestrator does the tasks itself in this session, one at a time, and says so up front. Council mode is not available. |
| `SendMessage` | Spawn a fresh agent each time, handing over the previous report and what's done. Council round 2 works the same way. |
| `TaskStop` | Can't stop a running worker: stop dispatching to it, and review whatever it leaves behind before anything else touches those files. |
| `CronCreate` or `CronDelete` | No heartbeat. Rely on completion notifications. |
| `PushNotification` | Post blockers in the conversation only; say up front they may not reach the user's phone. |
| `AskUserQuestion` | Ask in plain text. |
| `javis-worker` agent type or `isolation: "worktree"` | Use `general-purpose` with the fallback prompt below; single lane only. |

In Claude Cowork, talk to the user with `SendUserMessage` if that's how the session reports, and use `device_bash` for `git` if it's available.

## Roles

- **Orchestrator (the current session, whatever its model)**: queues, dispatches, reviews, runs the heartbeat. As a rule it does not edit files itself, with three exceptions:
  - **Small tasks**: one file, a few lines, nothing touching money / permissions / databases / deployment. The orchestrator may do it directly, still verifies it, and the report says "done by orchestrator".
  - **Approved high-risk actions** (see below): the orchestrator runs them itself.
  - The retry limit is hit and the user chooses to let the orchestrator take over.
- **Worker**: `Agent` with `subagent_type: "javis-worker"` and the model from step 0. Its working rules and report format live in `~/.claude/agents/javis-worker.md`, so dispatch prompts don't repeat them. If `javis-worker` is unavailable, use `general-purpose` and open the prompt with: "Default to low effort, but understand the relevant code path before changing it. Verify after your final change; if you can't run a check, report it as unverified. Prepare but never execute deploys, pushes to production, database changes or deletions of existing data — stop and report instead." Then add the report format.
- **One worker per lane**: rework and the lane's next task go to the same worker via `SendMessage`. Spawn a replacement, at a task boundary, only when: the old worker died, its replies are clearly degraded after rework, or the user switched model. Before replacing, `TaskStop` the old one and check what it left behind (`git status --short`). Hand over the previous report and what's done in the new prompt.

## High-risk actions

**Preparing is fine; executing needs approval.**

- Workers may: write migration files, deployment scripts or SQL files; delete temporary files they created in this task.
- Stop and get approval before: running a deploy; `git push` to production or the main remote; executing database changes against a real database (running migrations, direct SQL); deleting data or files that existed before this task.

The request names the exact action, the target environment and the scope (e.g. "run migration 0042 on production DB `shop`"). Approval covers only that. After a clear yes, the orchestrator runs it itself. That lane waits; other lanes carry on.

## Task queue and lanes

1. Add the user's tasks to the task list in the order given.
2. **Default to a single lane, run in order.** Only run lanes in parallel (max 3) when all of these hold:
   - the repo is on a local disk (**not a network drive, NAS, SMB share or cloud-synced folder**) and is a git repo;
   - the main checkout is on the default branch with no uncommitted changes (isolated worktrees may start from the default branch, not your current `HEAD`);
   - the tasks clearly don't interact: different files, no dependency on each other's results, no shared hot files (changelog, config, a shared partial);
   - `Agent` with `isolation: "worktree"` is available.

   If unsure, assume they collide and keep them in one lane.
3. After assigning lanes, post one line, e.g. "A: T1, T3; B: T2". Don't wait for approval; the user can object straight away.
4. **Parallel lanes use worktrees** (`isolation: "worktree"`). The worker commits on its branch and reports the worktree path, branch, base commit and commit hash. Before a lane's next task, tell its worker to merge the current default branch into its branch so it sees what other lanes have merged.
5. Shared hot spots (e.g. the end of a changelog) are updated by one worker after every lane has merged.
6. Within a lane, don't dispatch the next task until the current one is **done** (see the flow below).

## When the user interrupts

- **New task**: goes to the end of the queue; if the user says it's urgent, it goes next in the relevant lane.
- **Change of direction**: correct the same worker via `SendMessage`; if the change is large, replace the worker as described in "Roles".
- **Stop one lane**: `TaskStop` its worker and mark the lane `cancelled`. Other lanes carry on, and so does the heartbeat.
- **Pause**: mark the lane(s) `paused`. In-flight work may finish and be reviewed, but nothing new is dispatched and the heartbeat leaves paused lanes alone.
- **Stop everything**: `TaskStop` all workers, mark all lanes `cancelled`, `CronDelete` the heartbeat.
- Confirm in one line with the updated queue order.

## Per-task flow

```
record base → dispatch → worker does it + checks → report → orchestrator review
   ├─ not OK → SendMessage the same worker with exactly what's wrong → review again
   └─ OK     → (parallel: merge + integration check) → mark done → one-line report → next task
```

A task is only **done** after it has passed review and, for parallel lanes, merged cleanly and passed the integration check. A merge conflict or failed integration check means not done: `git merge --abort` if needed and ask the user.

### Before dispatching
- Record the base: `git rev-parse HEAD` and `git status --short` in the directory the worker will use. Changes already present are the user's, not the worker's, and are excluded from review.

### A dispatch prompt contains
- One line of project context (which repo / feature) and which memory notes to read.
- The task in the user's own words — don't paraphrase it into something else.
- **Acceptance criteria**: observable outcomes that show the task is done (e.g. "the invoice page shows dates as DD/MM/YYYY", "tax for 100.00 at 8% is 8.00"). Not just "tests pass".
- How to verify: which commands to run, and which of them are safe to repeat.

### Orchestrator review
- **Look at the real changes against the base**: committed work with `git diff --stat <base> <commit>` and then the key hunks; plus `git status --short` for uncommitted and untracked leftovers. Read line by line only where money, permissions or deletions are involved.
- **Check the verification actually proves the acceptance criteria.** A passing lint or a test that doesn't exercise the requirement is not proof.
- **Rerun only safe, repeatable checks** yourself (tests, linters, greps, local renders). Never rerun anything that writes to a real database or calls an external service.
- If something couldn't be verified (no runtime, no test database, no browser), the report says **unverified** — never treat it as passed.
- Check against the project's own rules (from `CLAUDE.md` / `AGENTS.md` / memory: naming, formatting, line endings, UI copy conventions). Any breach means not OK.
- Feedback must be specific: which file and line, what was expected, what is there now.

### Parallel lanes: merge and integration check
- In the main checkout, merge the reviewed commit (`git merge <branch>`), one lane at a time.
- Rerun the safe checks on the merged result. Only then mark the task done.

### Retry limit
After the first delivery, at most 2 rounds of rework. If it's still not OK after the second, stop that lane and tell the user where it's stuck, what you think, and what you suggest (change approach, orchestrator takes over, or skip). Other lanes carry on.

## Blockers and heartbeat

- **As soon as a worker reports a blocker** (login, credentials, permissions, a high-risk action awaiting approval, ambiguity), tell the user, plus a `PushNotification` if available. Make it self-contained: which task, what's blocking, one or two options.
- **Heartbeat** (only if both `CronCreate` and `CronDelete` loaded): scheduled jobs only fire while this session is idle, so this catches missed completion notifications. It is not a watchdog for a hung session. Keep **exactly one** recurring job, cron `*/30 * * * *`, prompt "agent-javis heartbeat", and record its ID.
  - Create it when the first worker is dispatched, and again whenever work resumes and no job exists.
  - On each run:
    1. No `active` lane (everything done, cancelled, paused or waiting on the user) → `CronDelete` itself and do nothing else.
    2. An `active` lane is idle, unfinished and not waiting on anyone → `SendMessage` the worker; if it's dead, replace it as in "Roles"; if a review was missed, do it now.
    3. Can't fix it yourself → tell the user. If nothing is wrong, stay quiet.

## Progress reports

- **Normally**: one line per finished task, e.g. "✅ T3: changed X (lane A); next T4". Add "(done by orchestrator)" or "(partly unverified: …)" where it applies.
- **Full table**: only when a task is blocked, the queue is empty, or the user asks for status. Two parts:
  1. One line per task: ✅ done / 🔄 in progress (lane) / ⏳ queued / ⛔ blocked / ⏸ paused or waiting on user.
  2. A three-to-five sentence project summary: where things stand, impact of the changes, newly found risks, what's next.
- Write in the user's language, keep it short, and don't repeat the workers' long reports.
- When the whole queue finishes and the user may be away, send a `PushNotification` if available.

## Council mode

For discussing a question, not editing files. All members are read-only, and the orchestrator doesn't edit either. Needs `Agent`.

1. **Pick roles and let the user approve.** The orchestrator picks 2–4 roles for the topic, one line each: name and **the area it focuses on** (e.g. "reliability: what breaks under failure"). Give a focus, not a predetermined conclusion. No fixed cast; the roles should cover different concerns. The devil's advocate is not counted here. Ask both questions in one `AskUserQuestion`:
   - **Roles** (multiSelect, one option per role): the user unticks unwanted roles or adds one via Other.
   - **Add a devil's advocate?** Ask every time, with no "Recommended" label:
     - Yes: argues against the mainstream view, hunts for holes and worst cases, and gives the strongest counter-argument even if it privately agrees.
     - No

   At most 5 members in total, including the devil's advocate.
2. **Round 1: independent opinions.** Spawn all members in a single message (`Agent`, `subagent_type: "general-purpose"`, model from step 0) so they run in parallel. Don't use `javis-worker`; it's an executor, not suited to deliberation. Each prompt includes:
   - The question verbatim plus context; point to any code or documents to read. If execution work is changing the same code, name a fixed commit to read from.
   - The role and its focus.
   - "You may only read. Do not modify any files."
   - Reply format: position (one sentence); reasoning (up to 5 points, each marked as fact, assumption or opinion, with its source where there is one); biggest risk or objection; what would change your mind.
3. **Round 2: key disagreements only.** The orchestrator lists the main disagreements from round 1. Each member gets the other members' round-1 opinions (labelled by role; condensing is fine, distorting is not) and responds to those disagreements: what they accept, what they dispute, and whether their position changed and why. Use `SendMessage`; without it, spawn a fresh agent with the member's own round-1 reply plus the others'. Include any facts the orchestrator has verified. If there is a devil's advocate, tell it to attack the conclusion most members agreed on in round 1.
4. **Orchestrator synthesis.** Don't recap each member; write:
   - **Consensus**
   - **Disagreements**: which roles disagree on what, with each side's strongest argument (a table works).
   - **Orchestrator's recommendation**: your own judgement and why — not just an average of the views.
   - **Open questions**: what the user needs to supply or decide.

   Open with a note that all members were played by the same model, so the diversity of views is limited.
5. **Next step.** Ask whether to turn the conclusions into tasks. If yes, switch to execution mode starting from "Task queue and lanes", doing step 0.4 first. If a council is opened while execution is running, existing lanes keep going and the conclusions are queued at the end.
6. No heartbeat for councils.

## Wrap-up

- Queue empty or user stops: `TaskStop` any remaining workers, `CronDelete` the heartbeat, send the final report.
- Record anything worth keeping across sessions (decisions, why something couldn't be done, new rules) in project memory.
