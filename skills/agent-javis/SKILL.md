---
name: agent-javis
description: "Orchestrate a user task list with sub-agents, or run an explicitly requested two-round council. Works with the capabilities available in Claude or Codex. Ask the user to choose sub-agent models before dispatch; review actual changes and verification before marking work done."
---

# agent-javis

The current session orchestrates; sub-agents execute and self-check. These rules are portable across Claude and Codex. Use the tools and permissions actually available in the current session, not assumed product defaults.

An ordinary request to review or compare this skill is not a request to launch workers or a council.

## Before dispatch

1. **Mode:** a delegated task list means execution mode. Council mode requires an explicit request for a council, debate or multiple perspectives. An ordinary review is done directly. Ask only when the intended mode is unclear.
2. **User chooses the model:** before the first sub-agent is launched, ask which available model the user wants. List only options verified by the current tool schema or runtime; include inheriting the session/default when supported. Do not hardcode a model, vendor or reasoning effort. If the user already specified the choice in this conversation, use it without asking again. Wait for the answer before dispatching; queue preparation and read-only inspection may continue.
   - Apply the choice to execution workers and council members unless the user specifies different models per role.
   - Do not read or write a saved model preference. A new conversation asks again unless its user has already chosen. A user-requested switch takes effect at the next task boundary.
   - If a chosen model is unavailable, or the host cannot select it, explain the limitation and ask the user to choose an available model or accept the host default. Never silently substitute or claim to run a model the tool cannot select.
   - If no sub-agent capability exists, explain that and proceed sequentially yourself for execution. Do not ask a meaningless model question or present a simulated council as independent agents.
3. **Capabilities:** identify available spawning, continuation, interruption, waiting, questions, task tracking and scheduling tools. Load deferred tools only through the host's supported discovery mechanism when needed.
4. **Queue:** use native task tracking if available; otherwise keep a checklist in the conversation or a writable scratch file. Record each lane's state, current task, worker ID, retry count, working directory and review baseline.

## Host tool mapping and fallbacks

These are examples, not required names. Read the current schemas before calling a tool. Never call an unavailable tool or copy arguments between incompatible tools.

| Capability | Claude examples | Codex examples | If unavailable |
|---|---|---|---|
| Spawn a sub-agent | `Agent` | `collaboration.spawn_agent` | Execute sequentially yourself; no independent council. |
| Continue a worker | `SendMessage` | `collaboration.followup_task` for new work; `collaboration.send_message` for an active worker | Spawn a replacement with its prior report and verified state. |
| Stop / inspect / wait | `TaskStop`, completion events | `collaboration.interrupt_agent`, `list_agents`, `wait_agent` | Stop dispatching; do not claim the worker stopped. Check its changes before another writer uses those files. |
| Ask the user | `AskUserQuestion` | An available user-input tool | Ask in chat. Required model choices and approvals need an actual answer. |
| Isolated checkout | `Agent` worktree isolation | Available worktree tools or supported Git worktrees | Use one writing lane. |
| Heartbeat | Available create/delete scheduling tools | Available thread automation tools | Rely on completion events and bounded waits. |
| Notifications | Available push tool | Host-supported notifications | Report in the conversation; do not promise a phone alert. |

Use actual sub-agents for delegation, not user-owned chats/threads. Creating or messaging a separate user chat requires the user's authorization under the host's rules.

Pass model choices only through supported parameters. For example, a host may restrict model overrides with full-history forks; use a supported context mode and provide the necessary task context. Explain any conflict between the user's choice and a fixed-model agent profile before dispatch.

The skill is self-contained: no external worker profile or product-specific home directory is required. Use a compatible general worker capability by default. A specialized profile such as `javis-worker` is optional and must not override the user's model choice or these instructions.

## Execution roles

The orchestrator queues, dispatches, reviews and reports. It may handle a small, low-impact edit directly, carry out already-authorized high-risk actions, or take over after the user chooses that remedy for repeated worker failure. Mark direct work in the report.

Use one worker per lane, continuing it for rework and subsequent tasks when possible. Replace it at a task boundary if it dies, its replies degrade despite correction, or a requested model switch requires replacement. Stop it where possible, inspect what it left behind and pass verified progress to the replacement. Never introduce a competing writer while the old worker may still be editing the same files.

### Worker dispatch contract

Give each worker:

- Project context and applicable project instructions.
- The user's task, preserving its intent and constraints.
- Explicit ownership of files or modules. Tell it other agents may be working in the repository, not to revert their work, and to accommodate their changes.
- Observable acceptance criteria and appropriate verification commands.
- A baseline commit plus existing changes, or a before-edit snapshot when Git is unavailable.
- These working rules: understand the relevant code before changing it; keep edits scoped; verify the final change; distinguish unverified results from passes. Prepare deployment scripts or migrations when requested, but hand execution of deployments, production pushes, real database changes and deletion of pre-existing data back to the orchestrator.

Worker report: result; changed files; checks and outcomes; unverified items or blockers; and, for isolated work, checkout path, branch, base commit and delivery commit. Do not require an external profile to define this format.

### Authorization

Preparing a high-risk action is allowed within the requested task. Before the orchestrator executes a deployment, production/main-remote push, real database change or deletion of pre-existing files/data, check that the user has authorized the action, target and scope. Reuse clear authorization already given in the conversation; do not ask again merely because this skill is active. If authorization is missing, finish the preparation, describe the concrete action and ask. Other independent lanes may continue.

## Queue and lanes

- Preserve the user's task order. New tasks go at the end unless marked urgent; urgent tasks go next in the relevant lane.
- Default to one lane. Parallel writing lanes require a local Git repository, clean integration checkout, independent tasks without shared hot files, and isolated checkouts with a known base. Do not use parallel writing lanes on NAS, SMB, network drives or cloud-synced folders.
- At most three writing lanes, further limited by available agent slots. If uncertain about overlap, use one lane. Tell the user the lane allocation without a separate approval round.
- Use the intended project branch/ref as the base; do not silently replace the user's current work with the default branch. Record each actual base and working directory. Explicitly direct workers to their own checkouts; spawning alone may not isolate files.
- After a lane is reviewed and integrated, bring that verified integration state into the lane before its next dependent task. One worker updates shared hot files after the other lanes integrate.
- A task is done only after review and, for isolated lanes, integration and relevant checks. Do not dispatch the next task in a lane before this point.

## Review and rework

1. Before dispatch, record `git rev-parse HEAD` and `git status --short` where available. Capture relevant pre-existing changes so they can be distinguished from new work. Without Git, retain a before-edit snapshot of the files in scope; do not initialize a repository merely for this workflow.
2. Review the actual delivery: committed diff against its base, uncommitted changes and untracked files. Inspect money, permissions and deletions closely. Check applicable `AGENTS.md`, `CLAUDE.md` and project conventions.
3. Decide whether the verification demonstrates the acceptance criteria. A passing lint check alone does not establish functional correctness. Rerun only appropriate, safe, repeatable checks; do not repeat database writes or external actions as a verification shortcut.
4. Report unavailable checks as unverified. Give specific rework feedback with file/line, expected behavior and actual behavior.
5. Integrate reviewed isolated work one lane at a time and check the combined result. A conflict or failed check leaves the task unfinished. Resolve routine conflicts within the authorized scope; ask when resolution needs a user decision. Abort only this workflow's merge when needed to preserve a recoverable state.
6. Allow at most two rework rounds after the initial delivery. If still unsuccessful, stop that lane and explain the blocker and options: change approach, orchestrator takeover or skip. Other independent lanes continue.

## Interruptions and progress

- Changes of direction go to the active worker; replace it only when necessary and after checking its state.
- On pause, mark the relevant lanes paused and dispatch no new work there. On stop, interrupt the relevant workers where supported and mark lanes cancelled. Be honest if interruption cannot be confirmed.
- Report blockers promptly with the affected task and a concrete way forward. Use supported notifications only within the host's authorization rules.
- Normally report one short line per finished task and what is next. Show the full queue when blocked, finished or asked for status. Include material impact, risks and unverified checks; do not repeat workers' full reports.
- Use the user's language.

## Optional heartbeat

Use completion events and bounded waits by default. If background monitoring is requested or already authorized, and the host supports creating and cancelling a suitable job, keep at most one heartbeat for this queue, typically every 30 minutes. Record its identifier and follow the host's scheduling schema and lifecycle. Do not create persistent automation just because this skill was loaded.

On a heartbeat: review missed completions; continue an idle, unfinished active lane when safe; replace a dead worker using the handover rules. Leave paused or user-blocked lanes alone. Stay quiet while nothing actionable changes; notify on completion, failure or required user action. Cancel the job when no active lanes remain; recreate it only when authorized monitoring resumes. A heartbeat does not guarantee recovery from a hung host, and idle-only execution is host-dependent.

## Council mode

Council members only read; the orchestrator does not edit files for the council. Requires real sub-agent capability.

1. Propose two to four roles with distinct areas of focus, not predetermined conclusions. Ask the user to approve or adjust them and whether to add a devil's advocate. Combine this with the required model question if still unanswered. At most five members including the devil's advocate.
2. Round 1: give every member the original question, relevant context, a role and read-only instructions. Use a fixed source snapshot if execution is changing the same material. Request: one-sentence position; up to five reasons labelled fact, assumption or opinion with sources where available; largest objection; what would change its mind. Use the user's model choice(s). Run within the host's concurrency limit, batching when necessary without showing earlier members' replies to later round-1 members.
3. Round 2: identify the important disagreements and send each member the other round-1 views, its own prior view if it is a replacement, and any verified facts. Ask what it accepts, disputes and changes its mind about. The devil's advocate challenges the strongest round-1 consensus. Continue existing members when supported; otherwise use read-only replacements with the full handover.
4. Synthesize consensus, remaining disagreements and their strongest arguments, the orchestrator's own recommendation, and unresolved questions. Describe actual model diversity honestly: same-model role-play has limited diversity; do not claim different vendors or models unless verified.
5. Ask whether to turn the conclusion into execution tasks. If yes, queue them and reuse the model choice already made in this conversation unless the user changes it. Existing execution lanes may continue; council outputs join their queue.
6. No heartbeat for councils.

## Wrap-up

Stop any remaining workers belonging to this run when the queue finishes or the user stops it, cancel this queue's heartbeat if any, and report results and limitations. Preserve reusable project decisions in existing project memory when appropriate, but do not persist a model preference.
