---
name: agent-javis
description: "Use when the user hands over a task list (or keeps adding tasks) and wants the current session to orchestrate sub agents, or wants a council of role-playing sub agents to debate a question (council mode: 2 rounds, then the orchestrator synthesises and can turn the conclusion into tasks). Remembers the sub agent model, spawns low-effort worker agents (single lane by default, parallel only when clearly safe), workers self-check, the orchestrator reviews, stops for approval before deploy/prod/DB/deletes, short progress reports."
---

# agent-javis — the current session orchestrates, sub agents do the work

## Step 0: before starting

1. **Mode**: if the user gives a task list, use execution mode. If they ask to discuss, debate or evaluate something, use council mode. Only ask when you genuinely can't tell.
2. **Model**: read `~/.claude/skills/agent-javis/last-model.txt` (one of `opus` / `sonnet` / `haiku` / `fable`). Use it if present; otherwise use `opus`. **Don't ask.** State it in your first line, e.g. "Execution mode, sub agents on Opus — say so if you want a different model." If the user switches, use the new model, write it back to the file, and apply it to the next agent you spawn.
3. **Load deferred tools in one go**: a single `ToolSearch` with `select:SendMessage,TaskStop,CronCreate,CronDelete,PushNotification`. Anything that fails to load is treated as unavailable; use the fallback in the environment table.
4. **Task tracking**: use `TaskCreate` / `TaskUpdate` / `TaskList` if available; otherwise keep a checklist in `javis-queue.md` in the scratchpad.

## Environment differences

| Purpose | Claude Code | Claude Cowork |
|---|---|---|
| Heartbeat scheduling | `CronCreate` / `CronDelete` | Unverified — no heartbeat |
| Talking to the user / reports | Reply in the conversation | `SendUserMessage` |
| Alerting a user who is away | `PushNotification` | Unverified — say up front "blockers will be posted in the conversation and may not reach your phone" |
| Reading diffs | `git` via `Bash` / `PowerShell` | `git` via `device_bash` if available |
| `javis-worker` agent type, worktrees | Available | Not available — use `general-purpose`, no worktrees |

## Roles

- **Orchestrator (the current session, whatever its model)**: queues, dispatches, reviews, runs the heartbeat. As a rule it does not edit files itself, with two exceptions:
  - **Small tasks**: one file, a few lines, nothing touching money / permissions / databases / deployment. The orchestrator may do it directly, still runs the verification, and the report says "done by orchestrator".
  - The retry limit is hit and the user chooses to let the orchestrator take over.
- **Worker**: `Agent` with `subagent_type: "javis-worker"` and the `model` from step 0. Low effort, the fixed report format and the high-risk stop rule all live in `~/.claude/agents/javis-worker.md`, so dispatch prompts don't repeat them. If `javis-worker` is unavailable, use `general-purpose` and open the prompt with "Work with low reasoning effort; check your work once before reporting; stop and report before any deploy, push to production, database change or deletion", followed by the report format.
- One worker per lane, kept for the whole lane. Rework and the lane's next task go to the same worker via `SendMessage`. Spawn a new worker only if the old one died, or its replies are clearly degraded after rework. When you do, hand over the previous report and what is already done in the new prompt. If `SendMessage` is unavailable, spawn a fresh `Agent` each time with the same handover.

## High-risk actions: always stop for user approval

Deploying, `git push` to production / the main remote, database changes (migrations, direct SQL), deleting data or files. Both workers and the orchestrator stop first, explain what will happen and what it affects, and wait for an explicit yes. That lane waits; other lanes carry on.

## Task queue and lanes

1. Add the user's tasks to the task list in the order given.
2. **Default to a single lane, run in order.** Only run lanes in parallel (max 3) when all three hold:
   - the repo is on a local disk (**not a network drive, NAS, SMB share or cloud-synced folder**) and is a git repo;
   - the tasks clearly don't interact: different files, no dependency on each other's results, no shared hot files (changelog, config, a shared partial);
   - you are running in Claude Code.

   If unsure, assume they collide and keep them in one lane.
3. After assigning lanes, post one line, e.g. "A: T1, T3; B: T2". Don't wait for approval; the user can object straight away.
4. **Parallel lanes must use worktrees** (`isolation: "worktree"`):
   - The dispatch prompt says: "Commit on your branch when done; report the worktree path, branch name and commit hash."
   - Review inside the worker's worktree (`git -C <path> diff --stat`; `cd` in before running tests), never in the main checkout.
   - Once approved, the orchestrator runs `git merge <branch>` in the main checkout, one lane at a time. On a conflict, `git merge --abort` and ask the user.
5. Shared hot spots (e.g. the end of a changelog) are updated by one worker after every lane has merged.
6. Within a lane, don't dispatch the next task until the current one has passed review.

## When the user interrupts

- **New task**: goes to the end of the queue; if the user says it's urgent, it goes next in the relevant lane.
- **Change of direction**: correct the same worker via `SendMessage`; if the change is large, `TaskStop` it and redispatch.
- **Stop a lane / stop everything**: `TaskStop` the workers and clear the heartbeat. "Pause" just means dispatch nothing new; in-flight work finishes.
- Confirm in one line with the updated queue order.

## Per-task flow

```
dispatch → worker does it + self-check → report (fixed format) → orchestrator reviews
   ├─ not OK → SendMessage the same worker with exactly what's wrong → review again
   └─ OK     → mark done → (merge if parallel) → one-line report → next task in that lane
```

### A dispatch prompt contains
- One line of project context (which repo / feature) and which memory notes to read.
- The task in the user's own words — don't paraphrase it into something else.
- Definition of done: expected files to change and how to verify (the tests / grep / render commands to run).

### Orchestrator review
- Don't just trust the report. **Rerun the worker's verification commands yourself** and compare results.
- Check `git diff --stat` against the report, then read the key hunks. Read line by line only where money, permissions or deletions are involved.
- Check against the project's own rules (from `CLAUDE.md` / `AGENTS.md` / memory: naming, formatting, line endings, UI copy conventions). Any breach means not OK.
- Feedback must be specific: which file and line, what was expected, what is there now.

### Retry limit
After the first delivery, at most 2 rounds of rework. If it's still not OK after the second, stop that lane and tell the user where it's stuck, what you think, and what you suggest (change approach, orchestrator takes over, or skip). Other lanes carry on.

## Blockers and heartbeat

- **As soon as a worker reports a blocker** (login, credentials, permissions, a high-risk action awaiting approval, ambiguity), tell the user; in Claude Code also send a `PushNotification`. Make it self-contained: which task, what's blocking, one or two options.
- **Fallback heartbeat (Claude Code only)**: after the first worker is dispatched, create **one** recurring `CronCreate` job every 25 minutes with the prompt "agent-javis heartbeat", and note its ID. On each run:
  1. Queue empty, user stopped the work, or every lane is waiting on the user → `CronDelete` itself and do nothing else.
  2. A lane is idle, unfinished and not waiting on anyone → `SendMessage` the worker; if it's dead, spawn a replacement with a handover; if a review was missed, do it now.
  3. Can't fix it yourself → tell the user. If nothing is wrong, stay quiet.
- Any wrap-up (queue empty, user stop, session ending) must `CronDelete` the job.

## Progress reports

- **Normally**: one line per finished task, e.g. "✅ T3: changed X (lane A); next T4". Add "(done by orchestrator)" where it applies.
- **Full table**: only when a task is blocked, the queue is empty, or the user asks for status. Two parts:
  1. One line per task: ✅ done / 🔄 in progress (lane) / ⏳ queued / ⛔ blocked / ⏸ waiting on user.
  2. A three-to-five sentence project summary: where things stand, impact of the changes, newly found risks, what's next.
- Write in the user's language, keep it short, and don't repeat the workers' long reports.
- When the whole queue finishes and the user may be away, send a `PushNotification` in Claude Code.

## Council mode

For discussing a question, not editing files. All members are read-only, and the orchestrator doesn't edit either.

1. **Pick roles and let the user approve.** The orchestrator picks 2–4 roles for the topic, one line each: name, the stance it represents, what it watches for. No fixed cast; the roles should pull against each other. The devil's advocate is not counted here. Ask both questions in one `AskUserQuestion`:
   - **Roles** (multiSelect, one option per role): the user unticks unwanted roles or adds one via Other.
   - **Add a devil's advocate?** Ask every time, with no "Recommended" label:
     - Yes: argues against the mainstream view, hunts for holes and worst cases, and gives the strongest counter-argument even if it privately agrees.
     - No

   At most 5 members in total, including the devil's advocate.
2. **Round 1: independent opinions.** Spawn all members in a single message (`Agent`, `subagent_type: "general-purpose"`, `model` from step 0) so they run in parallel. Don't use `javis-worker`; it's a low-effort executor and not suited to deliberation. Each prompt includes:
   - The question verbatim plus context; point to any code or documents to read.
   - The role and its stance.
   - "You may only read. Do not modify any files."
   - Reply format: position (one sentence), reasoning (up to 5 points), biggest risk or objection, what would change your mind.
3. **Round 2: respond to the others.** Send each member the other members' round-1 opinions (labelled by role; condensing is fine, distorting is not) via `SendMessage`, and ask what they agree with, what they dispute, and whether their position changed and why. Include any facts the orchestrator has verified. If there is a devil's advocate, tell it to attack the conclusion most members agreed on in round 1.
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
