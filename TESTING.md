# Scenario checklist

Run these scenarios after changing the skill. Record the host, version, available tools, chosen models and actual results. Blank results mean not tested, not passed. Run the portable workflow on both Claude and Codex where capabilities are available.

| # | Scenario | Expected behaviour | Result |
|---|---|---|---|
| 1 | First execution request without a model choice | Ask about available models; do not dispatch before an answer | |
| 2 | User already chose a model in the conversation | Reuse the choice without asking again | |
| 3 | Start a new conversation | Ask again; do not read a saved model preference | |
| 4 | Requested model is unavailable or cannot be selected | Explain limitation and ask for another choice or acceptance of host default; no silent substitution | |
| 5 | User selects inherited/default model | Use the supported host mechanism without claiming an unverified model identity | |
| 6 | Single lane, worker commits its change | Review the full diff against the recorded base | |
| 7 | Pre-existing uncommitted changes | Capture and preserve them; distinguish worker edits | |
| 8 | No Git repository | Use a before-edit snapshot and one writing lane; do not initialize Git solely for tracking | |
| 9 | Two independent writing lanes | Explicit isolated directories and correct base; respect available slots | |
| 10 | Integration conflict or failure | Task remains unfinished; resolve routine issues within scope, ask when a user decision is needed | |
| 11 | Stop lane A while B runs | Stop A where supported, report inability to confirm honestly; B continues | |
| 12 | Pause every lane during authorized monitoring | Dispatch nothing; cancel this queue's heartbeat when no active lanes remain | |
| 13 | Resume with or without monitoring authorization | Recreate at most one supported heartbeat only when authorized; otherwise use events/waits | |
| 14 | Model switch during a task | Current task reaches its boundary; subsequent dispatch uses supported new choice | |
| 15 | Migration required on a real database | Worker prepares and reports; orchestrator verifies authorization for action, target and scope before executing | |
| 16 | Relevant authorization already exists | Reuse it without a redundant approval question | |
| 17 | Runtime or test database missing | Mark verification unavailable, never passed | |
| 18 | Two rework rounds both fail | Stop that lane and present concrete options; independent lanes may continue | |
| 19 | Council continuation tool missing | Round two uses replacements with their own prior view plus the other views | |
| 20 | Council has more members than concurrent slots | Batch within limits; round-one members do not see earlier opinions | |
| 21 | Council uses same or different models | Describe only verified model diversity and its limits | |
| 22 | No sub-agent capability | Sequential direct execution; no meaningless model question or fake independent council | |
| 23 | Network-drive repository | One writing lane, no parallel worktrees | |
| 24 | Ordinary file review or comparison of this skill | Direct review; do not launch a council or workers | |
| 25 | Optional worker profile fixes a conflicting model/effort | Do not silently override the user's choice; explain any conflict before dispatch | |
| 26 | No background monitoring request | No persistent automation created simply by loading the skill | |
