# Scenario checklist

agent-javis mostly breaks where sections interact, which is hard to spot by rereading. Run these by hand after changing the skill. Record the Claude Code version and the result.

| # | Scenario | Expected behaviour | Result |
|---|---|---|---|
| 1 | Single lane, worker commits its change | Review shows the full diff via `git diff <base> <commit>`, not an empty `git diff` | |
| 2 | Repo has uncommitted changes before dispatch | Those changes are recorded in the base and not attributed to the worker | |
| 3 | Two parallel lanes, the second merge conflicts | `git merge --abort`, task not marked done, user asked | |
| 4 | Two parallel lanes, each passes alone but fails after merging | Integration check fails, task not marked done | |
| 5 | Stop lane A while lane B is running | A's worker stopped, B continues, heartbeat still exists | |
| 6 | Pause all lanes, wait for a heartbeat | Heartbeat deletes itself and dispatches nothing | |
| 7 | Resume after a pause | Exactly one heartbeat job exists again | |
| 8 | User switches model mid-lane | Current task finishes on the old worker; next task runs on a new worker with the new model | |
| 9 | Worker needs to run a migration on a real DB | Worker writes the migration, stops and reports; orchestrator asks with action, target and scope, then runs it after a yes | |
| 10 | No PHP runtime / test DB available | Report says "unverified", not "passed" | |
| 11 | Council where `SendMessage` is unavailable | Round 2 completes with fresh agents and handovers | |
| 12 | Repo on a network drive with independent tasks | Single lane, no worktrees | |
| 13 | User says "review this file" | Direct review, no council | |
