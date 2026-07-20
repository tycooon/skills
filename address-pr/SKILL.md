---
name: address-pr
description: Use when asked to address a pull request or merge request — syncs with the base branch (rebasing a stacked PR onto the default branch once the PR underneath it has merged) and resolves merge conflicts, then reads and addresses the open/unresolved review comments (making code changes and replying on each thread, but leaving the threads open for the reviewer to resolve), and fixes or retries failing CI checks, then pushes. Works on the current branch's PR or one given by number/URL, on GitHub or GitLab. The counterpart to review-pr, which posts comments rather than addressing them.
---

# Address a Pull Request

Get a pull request (GitHub) or merge request (GitLab) — the one for the current branch, or the one given by number/URL — ready by getting it onto the right base, resolving conflicts, working through its review comments, and clearing failing CI, then pushing once.

## 1. Sync with the base branch

Fetch origin, then determine the PR's base branch (GitHub: `gh pr view <pr> --json baseRefName`; GitLab: the MR's target branch) and the repo's default branch.

**First work out whether this PR is — or was — stacked on another PR**, built on that PR's branch rather than straight on the default branch. It shows up in one of two ways, and the second is easy to miss:

- Its base is still a feature branch rather than the default branch. The PR underneath hasn't merged yet.
- Its base already reads as the default branch, but the branch still carries another PR's commits. Don't go by the base field alone: GitHub and GitLab both retarget a stacked PR to the default branch automatically the moment the PR underneath merges, so the base looks ordinary while the history is still stacked. The tells are the PR's commit list and diff still showing the other PR's work, and the old base branch being gone from the remote.

Then act on the state of the PR underneath:

- **Still open** — leave the stack alone. Stay stacked on it and merge `origin/<base>` in as below.
- **Merged** — its work has landed, so this PR belongs on the default branch now. Unstack it, as below.
- **Closed without merging** — do *not* rebase onto the default branch. That work never landed, so rebasing would silently drop it. Stop and ask.

To unstack, replay only this PR's own commits onto the default branch, cutting off everything that came from the PR underneath:

```
git rebase --onto origin/<default> <head commit of the PR underneath>
```

Get that commit from the merged PR itself (GitHub: `gh pr view <that-pr> --json headRefOid`) — it is still reachable from this branch even though its branch has been deleted. Then retarget this PR's base to the default branch if the platform hasn't already.

Do **not** reach for a plain `git rebase origin/<default>` instead. The PR underneath was almost certainly squash-merged, so the default branch holds its *squashed end state* while this branch still holds its *intermediate* commits — replaying those on top collides with the squash and conflicts on nearly every file it touched. `--onto` cuts them out wholesale, which is what you want.

If the PR was never stacked — an ordinary one straight onto the default branch — merge `origin/<base>` into the branch: merge, not rebase, so the follow-up push stays a plain `git push` with no force-push. Rebasing to unstack is the one exception, and it makes the final push a `git push --force-with-lease`.

If the merge or rebase is clean, there's nothing to resolve. If it conflicts, resolve each conflict with real code judgment: understand what both sides changed, preserve the intent of each, and favor a correct result over mechanically taking one side.

Flag any resolution you're unsure about in the final report. If a conflict genuinely can't be resolved with confidence, stop and ask rather than pushing a broken merge — and never force-push a branch you couldn't rebase cleanly.

## 2. Address the review comments

Find the open review comments and unresolved threads and work through them:

- Understand each comment. Apply technical rigor: don't blindly implement. Make the code change when the comment is right; when it's wrong, misguided, or based on a misunderstanding, don't make a bad change — explain your reasoning in the reply instead. When a comment is genuinely ambiguous or a judgment call, ask the user rather than guessing.
- For each actionable comment, make the code change. Group related comments so a single change can address several threads.
- Reply on each thread saying what you did, or why you didn't, then leave the thread open. Resolving a thread is the reviewer's call, not the author's — the reviewer verifies the fix (or your reasoning) and resolves it on re-review. Replies should include your identity (AI agent name).
- Skip already-resolved threads and comments that are just approval or acknowledgement. Also skip any thread whose latest reply is already your own: you've addressed it and it's now waiting on the reviewer, so replying again would only duplicate — re-engage such a thread only once someone has replied back after you.

## 3. Check and fix CI failures

Take one snapshot of the PR's checks — don't poll or wait for in-progress runs (GitHub: `gh pr checks`, then `gh run view --log-failed` on a failing run; GitLab: the MR's pipeline and its failed job traces). Failing CI is reason enough to act on its own — check it even when there were no conflicts and no review comments.

- For each failing check, read the failing job's logs and diagnose the cause.
- Fix the failures this branch is responsible for and can confidently fix, folding the fix into the same change batch as the conflict and comment work.
- Retry the failed jobs when a failure is infra, flaky, or otherwise external rather than a real problem in the code (GitHub: `gh run rerun --failed`; GitLab: retry the job or pipeline). Don't wait for the retry to finish.
- Leave the rest — failures that also fail on the base branch, or that you can't confidently fix or clear — and report them instead of chasing them.

## 4. Push and report

- Push once, carrying the conflict resolution, comment fixes, and CI fixes together — a plain `git push`, or `git push --force-with-lease` if you rebased to unstack. If there was genuinely nothing to change — no conflicts, no actionable comments, no fixable CI failures — don't push a no-op; a job retry stands on its own and needs no push.
- Give the user a short report: what changed, what you pushed back on, which conflicts you resolved (calling out anything uncertain), and how each failing check was handled — fixed, retried, or left (and why). If you pushed, a fresh CI run starts on the new commit; the skill doesn't wait for it.
