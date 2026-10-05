# agent-javis

A skill for Claude and Codex that turns the current session into an orchestrator: it delegates tasks to sub-agents, reviews their actual changes, and reports results. An explicitly requested council runs two rounds of independent views and discussion.

## Model selection

Before launching sub-agents, the orchestrator asks which available model you want. You can choose the host's inherited/default model when supported, or specify different available models for council roles.

There is no hardcoded model or reasoning effort, and no saved model preference. A choice already made in the current conversation is reused. If the host cannot run your choice, the orchestrator explains the limitation and asks rather than silently substituting.

## What it does

- Queues tasks in your order; one execution lane by default.
- Allows up to three independent writing lanes only with suitable local Git repositories and isolated checkouts, within the host's agent limit.
- Reviews changes against a recorded baseline and checks observable acceptance criteria.
- Marks isolated work done only after integration and relevant checks.
- Allows at most two rework rounds before asking how to proceed.
- Preserves existing user changes and assigns explicit worker ownership.
- Prepares high-risk actions, with execution handled by the orchestrator only when the action, target and scope are authorized. Existing clear authorization is reused.
- Supports task additions, direction changes, pausing and stopping lanes.
- Reports unverified checks honestly and keeps progress updates short.
- Uses completion events by default. Background heartbeat monitoring is optional and requires authorization and host support.

## Install

The required component is `skills/agent-javis/SKILL.md`. The skill includes its worker instructions and report contract; no separate agent profile is required.

### Codex

Place the `skills/agent-javis` folder in the skill directory discovered by your Codex installation, for example:

```text
~/.agents/skills/agent-javis/SKILL.md
```

Use your configured skill location if it differs. Start a new session if needed to load the updated skill.

### Claude Code

Place the same folder at:

```text
~/.claude/skills/agent-javis/SKILL.md
```

Optionally install `agents/javis-worker.md` at `~/.claude/agents/javis-worker.md`. This is a Claude-style helper profile, not a portable prerequisite. It contains no fixed model or effort setting; the orchestrator must still apply the user's supported choice.

On Windows, `~` refers to your user profile directory.

### Claude.ai / Cowork

If your environment supports uploading skills, zip the folder so the archive contains `agent-javis/SKILL.md`, then use the host's skill upload flow.

Actual capabilities depend on the tools exposed in that session. Without sub-agents, execution falls back to sequential work by the current session, and an independent council is unavailable.

## Usage

Invoke the skill using your host's skill picker or supported command syntax, then provide tasks:

```text
Use agent-javis:
1. Fix the date format on the invoice page
2. Add a unit test for the tax calculation
3. Update the README
```

Or explicitly request a council:

```text
Use agent-javis to hold a council:
Should we move session storage from files to Redis?
```

The council proposes two to four roles and asks whether to add a devil's advocate. You approve or adjust the roles and select models before dispatch. Members give independent round-one opinions, then address disagreements in round two. The orchestrator synthesizes consensus, disagreements, its recommendation and open questions.

## Portability and limits

- Tool names in the skill are examples. The orchestrator follows current host schemas and permissions.
- Spawning an agent does not necessarily isolate its files. Parallel writing requires explicitly assigned isolated checkouts.
- NAS, SMB, network drives and cloud-synced folders use one writing lane.
- A model choice does not grant access to models absent from the host.
- Same-model role-play offers limited viewpoint diversity; actual model diversity must be reported honestly.
- Heartbeat scheduling and notifications are host-dependent and cannot guarantee recovery from a hung host.

## Validation status

The portable revision has passed basic local file-format and content checks. The bundled skill validator could not run because PyYAML was unavailable. The revised workflow has not yet completed end-to-end testing on both platforms.

Earlier council testing was reported on Claude Code desktop for Windows; that does not establish validation of this portable revision. Use [TESTING.md](TESTING.md) and record actual results.

## Files

- `skills/agent-javis/SKILL.md`: portable orchestration rules and worker contract.
- `agents/javis-worker.md`: optional Claude-style executor profile.
- `TESTING.md`: manual behavioral scenarios.

## License

MIT. See [LICENSE](LICENSE).
