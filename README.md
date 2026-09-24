# agent-javis

A Claude skill that turns your current session into an orchestrator: it hands your tasks to sub agents, reviews their work, and reports back. It also has a **council mode**, where several role-playing sub agents debate a question before the orchestrator sums up.

## What it does

**Execution mode**: give it a task list.

- Queues the tasks in your order and dispatches them to low-effort worker agents.
- Runs a single lane by default. It only goes parallel (up to 3 lanes, each in its own git worktree) when the repo is local, the tasks clearly don't touch each other, and you're in Claude Code.
- The orchestrator actually reviews the work: it diffs against a base commit recorded before dispatch, checks that the verification proves written acceptance criteria, and reruns only safe, repeatable checks. Anything it couldn't check is reported as unverified.
- A task only counts as done after review and, for parallel lanes, a clean merge plus an integration check.
- Bounded rework: at most 2 retries per task, then it stops and asks you.
- **High-risk actions need your approval.** Workers may prepare migrations or deploy scripts, but running a deploy, pushing to production, executing database changes or deleting existing data waits for an explicit yes naming the action, target and scope.
- Lanes can be paused or stopped individually; the others keep going.
- Short reports: one line per finished task, with a full status table only when something is blocked, the queue is empty, or you ask.
- Pings you (`PushNotification`) when a worker gets stuck. An optional 30-minute heartbeat, which only fires while the session is idle, picks up missed completion notifications.

**Council mode**: give it a question.

- The orchestrator proposes 2–4 roles that pull against each other; you approve or edit them.
- Every time, you choose whether to add a devil's advocate.
- Round 1: each member gives an independent view. Round 2: each member responds to the others.
- The orchestrator writes up consensus, disagreements, its own recommendation, and open questions. It can then turn the result into tasks for execution mode.

## Install

### Claude Code

Copy the two pieces into your Claude config folder:

| From this repo | To |
|---|---|
| `skills/agent-javis/` | `~/.claude/skills/agent-javis/` |
| `agents/javis-worker.md` | `~/.claude/agents/javis-worker.md` |

On Windows, `~` is `%USERPROFILE%` (e.g. `C:\Users\you`).

macOS / Linux:

```bash
git clone https://github.com/whatsphone-creator/agent-javis.git
cp -r agent-javis/skills/agent-javis ~/.claude/skills/
cp agent-javis/agents/javis-worker.md ~/.claude/agents/
```

Restart Claude Code (or start a new session) so it picks up the skill and the agent.

### Claude.ai / Cowork

1. Zip the `skills/agent-javis` folder so the zip contains `agent-javis/SKILL.md`.
2. Upload it under **Settings → Capabilities → Skills**.

The skill decides its behaviour from the tools that actually load in the session. Where the `javis-worker` agent type, worktrees, `SendMessage`, scheduling or push notifications are missing, it falls back to `general-purpose` agents, a single lane, fresh agents with handovers, and in-conversation alerts. Without the `Agent` tool at all, the orchestrator does the tasks itself.

**Status:** council mode has been run on Claude Code desktop (Windows). Execution mode has not yet been run end to end against [TESTING.md](TESTING.md), and Cowork / claude.ai are unverified. Reports welcome.

## Usage

```
/agent-javis
1. Fix the date format on the invoice page
2. Add a unit test for the tax calculation
3. Update the README install section
```

```
/agent-javis Hold a council: should we move our session storage from files to Redis?
```

The first line of its reply tells you which mode and model it's using. To switch model, just say so (e.g. "use sonnet"). Your choice is saved in `~/.claude/skills/agent-javis/last-model.txt` for next time. The default is `opus`.

You can interrupt at any time: add a task, change a task's direction, or stop a lane or everything.

## Notes and limits

- **`effort: low`** in `javis-worker.md` needs a Claude Code version that supports `effort` in agent frontmatter. Otherwise the worker runs at the default effort.
- **Council members are all the same model** playing different roles, so the range of views is limited. For a genuinely different model's view, pair it with a cross-vendor council tool.
- **The heartbeat is not a watchdog.** Scheduled jobs only fire while the session is idle, so it can't rescue a hung session.
- **Network drives** (NAS, SMB, cloud-sync folders) never get parallel lanes or worktrees, because git on them tends to hang or fight over `index.lock`.

## Files

```
skills/agent-javis/SKILL.md   the orchestration rules
agents/javis-worker.md        the low-effort executor agent and its report format
TESTING.md                    manual scenario checklist to run after changing the skill
```


## License

MIT. See [LICENSE](LICENSE).
