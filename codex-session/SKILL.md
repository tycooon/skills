---
name: codex-session
description: "Hand a task to a headless Codex session from Claude, review its PR and resume the session until it is ready."
---

# Run a Codex Session

The user asked for a Codex session or agent on a task. Codex writes the code and you review it: launch it headless with `codex exec`, review what it delivers, and resume the same session whenever something is left to change, until its PR is ready. Don't make those changes yourself. A running Codex session counts as a subagent toward any limit your instructions put on running them in parallel.

For an opinion only, with no code change and no follow-up, skip the steps and run `codex exec --ephemeral -s read-only -C DIR -o ANSWER - < PROMPT > LOG 2>&1`.

## 1. Write the brief

Save it as `<working-dir>/.plans/YYYY-MM-DD-<topic>-codex-brief.md`, kept out of git as [design-brainstorming](../design-brainstorming/SKILL.md) describes, so the user can open it. Codex follows the same global rules and has the same skills, so the brief carries only the task: the problem with its evidence, what done looks like, what is out of scope. It also tells Codex to:

- work in the worktree it starts in and on the branch it finds there, making no other worktree: step 2 cuts both from the latest `origin/<default>` for it;
- stop once its non-draft PR is open, with no babysit-pr and no waiting for a review, because you are the reviewer and start only when its run exits;
- not merge or deploy;
- end its final message with a report block: PR URL, one-line summary, each command it ran with its result, anything the reviewer or the user must know or answer.

## 2. Launch

Make the session's worktree yourself, where your instructions put worktrees, on a new branch cut from the latest `origin/<default>`. Name the worktree `codex-<topic>` after the brief, and the branch the way your instructions name branches, `codex/<topic>` when they don't say.

```bash
git -C MAIN_CHECKOUT fetch origin
git -C MAIN_CHECKOUT worktree add --no-track -b BRANCH WORKTREE origin/DEFAULT
```

A repository's own command for making a worktree comes first: when its instructions give one, run it with the branch name in place of these two commands, and use the path it prints as the worktree.

Then start Codex in it, in a background Bash with `timeout: 7200000`. Two hours is the maximum, and the default stops it after 30 minutes.

```bash
codex exec -C WORKTREE --approve-for-me --color never -o SCRATCHPAD/NAME-last.md - < BRIEF > SCRATCHPAD/NAME.log 2>&1
```

- Never pass `--worktree`. Codex would make a worktree of its own at `<its worktree root>/<4 hex digits>/<repo>`, detached at the main checkout's HEAD: outside the place your instructions and the project's tooling expect a worktree, and on a base that can be behind.
- `--approve-for-me` keeps the `workspace-write` sandbox and sends approval requests to Codex's automatic reviewer, since nobody is there to answer a prompt. Never pass `--dangerously-bypass-approvals-and-sandbox`.
- Never pass `--ephemeral`: it keeps no session to resume.

The log opens with `stale rollout path` errors, which are noise, and within seconds a header with the `session id:` line a resume needs. Read it and tell the user the session id, the worktree and the brief.

**When auto mode refuses.** The first launch in a conversation is usually refused as `[Create Unsafe Agents]` when the user's request said no more than "run codex". Whenever a launch or a resume is refused, don't try variants: show the user the exact command, say what `--approve-for-me` does, and re-issue it unchanged on their go. If it is refused again, the command is theirs to run.

## 3. Review

You are re-invoked when the run exits.

1. Read the `-o` file. A run that was cut off leaves none, so read the log's tail instead.
2. Attach the PR it opened to your session, once, when your runtime can show a PR beside the conversation, as the Claude desktop app does: the user follows its checks and review state there. Attach it and nothing more: no watch goes on with it, as the next step says.
3. Review the PR with [review-pr](../review-pr/SKILL.md), against the brief as well as the diff. You are this PR's external AI reviewer, so keep [babysit-pr](../babysit-pr/SKILL.md) and any Auto-fix watch off for it: both would have you fix what you review.
4. Read the PR's checks and mergeability yourself afterwards: review-pr leaves CI out.
5. Pass the user any question Codex left that the brief doesn't answer.

## 4. Resume

Resume the session when anything is left for it to do: an open review thread, a failed or still-running check, a conflict with the base, or a run that stopped short of its PR. Use the same background Bash and timeout.

```bash
codex exec -C WORKTREE --approve-for-me --color never -o SCRATCHPAD/NAME-last-2.md resume SESSION_ID - < FOLLOW_UP > SCRATCHPAD/NAME-2.log 2>&1
```

- Always pass `-C` with the session's worktree. A resumed run works in the directory it is given (the current one by default), not where the session began.
- Keep the flags before the word `resume`, which takes neither `-C` nor `--approve-for-me` after it.

The follow-up says what is left: for one [address-pr](../address-pr/SKILL.md) pass (fix or argue, reply without resolving, push), the links to each open thread and failed check and the text of any finding that has no thread; or what a run that stopped short still owes. It always ends by asking Codex, once that is done, to wait for the checks on the head it leaves and to report them with what changed. Re-review that head when the run exits.

## 5. Finish

Done is the four conditions from babysit-pr's *When to stop*, with your review as the approval. Tell the user then; merging and deploying stay their call. When a resume leaves an item as it was (a thread you still hold open, a check still red, a run cut off or blocked again), bring that item to the user rather than resuming for it again.
