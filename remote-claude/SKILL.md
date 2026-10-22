---
name: remote-claude
description: "Hand a coding task to a headless Claude Code session on a remote dev host, review its PR and resume the session until it is ready."
---

# Run a Remote Claude Session

The user asked for a Claude session on a remote dev host, or your machine rules send the repository's coding work there. It works like a Codex session: a headless Claude Code (`claude -p`) on HOST writes the code, you review it and resume the same session until its PR is ready, and you make none of the changes yourself. A running session counts as a subagent toward any limit your instructions put on running them in parallel.

Follow [codex](../codex/SKILL.md) from start to finish, with its *On a remote host* section, and swap in Claude's commands as below. Everything codex says about the brief, overlapping PRs, the worktree, mockups, review, resumes and finishing holds unchanged.

## What changes

- **Model and effort.** Every run passes `--model MODEL --effort EFFORT`, so HOST's settings never decide them. Unless the user names others, use the model you run on (its alias, such as `opus`) and `medium` effort. There is no speed tier: leave SPEED out.
- **Session id.** Make it yourself before the launch, so a resume never depends on the log: `uuidgen | tr 'A-Z' 'a-z'`. Tell the user the id with the worktree and the brief.
- **Launch.** As codex's *On a remote host* launch, in a background Bash with `timeout: 7200000`, with Claude in place of `codex exec`:

  ```bash
  ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=6 HOST 'cd WORKTREE && mkdir -p .plans && echo $$ > .plans/NAME.pid && exec claude -p --session-id SESSION_ID --model MODEL --effort EFFORT --permission-mode auto --output-format stream-json --verbose > .plans/NAME.jsonl 2>&1' < BRIEF
  ```

  `stream-json` writes each event as it happens, so the file shows how far a run got while it runs and after it stops; the plain text output would print nothing until the end. There is no `-o` file: the final message is the `result` field of the last `"type":"result"` line, `ssh HOST "grep '\"type\":\"result\"' WORKTREE/.plans/NAME.jsonl | tail -1" | jq -r .result`. A run that stopped without one left no final message: read the last few `assistant` events instead.
- **Resume.** Same command, with `--resume SESSION_ID` in place of `--session-id SESSION_ID`, the follow-up on stdin and the next numbered `.jsonl` file and pidfile (`NAME-2.pid`, …). Check first that the last run has exited, as codex's remote section describes, with that run's pidfile. Run it in the same WORKTREE: Claude keeps a session under the directory it started in, and a resume from anywhere else does not find it.
- **Process watch.** After every launch and resume, arm codex's *Watch the process, not the connection* watch on that run's own pidfile, with the same timeout and the same reading of its `RUN_ENDED` and `PIDFILE_MISSING` lines. Its `RUN_ENDED`, not the `ssh` command's exit, is the run's end, and a run's state is reported only from a fresh pid check.
- **Permissions.** `auto` lets HOST's classifier judge each action, since nobody is there to answer a prompt. A refusal ends up in the run's report like any other blocker. Never pass `--dangerously-skip-permissions` or `--permission-mode bypassPermissions`.
- **Rules.** The session loads HOST's global rules and skills, as Codex does, so the brief carries the task and the implementation plan, as codex's step 1 describes, without repeating those rules, plus the line codex's remote section asks for: that it runs on HOST.
- **When auto mode refuses** the launch or a resume here, codex's *When auto mode refuses* applies as written.
