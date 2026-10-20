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
- **Launch.** Instead of codex's `codex exec`, in a background Bash with `timeout: 7200000`:

  ```bash
  ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=6 HOST 'cd WORKTREE && claude -p --session-id SESSION_ID --model MODEL --effort EFFORT --permission-mode auto' < BRIEF > SCRATCHPAD/NAME.log 2>&1
  ```

  The final message is the end of the log: there is no `-o` file to read.
- **Resume.** Same command, with `--resume SESSION_ID` in place of `--session-id SESSION_ID` and the follow-up on stdin. Run it in the same WORKTREE: Claude keeps a session under the directory it started in, and a resume from anywhere else does not find it.
- **Permissions.** `auto` lets HOST's classifier judge each action, since nobody is there to answer a prompt. A refusal ends up in the run's report like any other blocker. Never pass `--dangerously-skip-permissions` or `--permission-mode bypassPermissions`.
- **Rules.** The session loads HOST's global rules and skills, as Codex does, so the brief still carries only the task, plus the line codex's remote section asks for: that it runs on HOST.
- **When auto mode refuses** the launch or a resume here, codex's *When auto mode refuses* applies as written.
