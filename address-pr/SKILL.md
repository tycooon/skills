---
name: address-pr
description: Use when asked to address a pull request or merge request — resolves merge conflicts with the base branch if present, then reads and addresses the open/unresolved review comments (making code changes, replying on each thread, resolving them), and fixes or retries failing CI checks, then pushes. Works on the current branch's PR or one given by number/URL, on GitHub or GitLab. The counterpart to review-pr, which posts comments rather than addressing them.
---

# Address a Pull Request

Get a pull request (GitHub) or merge request (GitLab) — the one for the current branch, or the one given by number/URL — ready by resolving conflicts, working through its review comments, and clearing failing CI, then pushing once.

## 1. Resolve conflicts with the base branch (if present)

- Fetch origin and determine the PR's base branch. Merge `origin/<base>` into the branch — merge, not rebase, so the follow-up push stays a plain `git push` with no force-push.
- If the merge is clean, there's nothing to resolve. If it conflicts, resolve each conflict with real code judgment: understand what both sides changed, preserve the intent of each, and favor a correct result over mechanically taking one side.
- Flag any resolution you're unsure about in the final report. If a conflict genuinely can't be resolved with confidence, stop and ask rather than pushing a broken merge.

## 2. Address the review comments

Find the open review comments and unresolved threads and work through them:

- Understand each comment. Apply technical rigor: don't blindly implement. Make the code change when the comment is right; when it's wrong, misguided, or based on a misunderstanding, don't make a bad change — explain your reasoning in the reply instead. When a comment is genuinely ambiguous or a judgment call, ask the user rather than guessing.
- For each actionable comment, make the code change. Group related comments so a single change can resolve several threads.
- Reply on each thread saying what you did, or why you didn't. Resolve threads you've addressed. Replies should include your identity (AI agent name).
- Skip already-resolved threads and comments that are just approval or acknowledgement.

## 3. Check and fix CI failures

Take one snapshot of the PR's checks — don't poll or wait for in-progress runs (GitHub: `gh pr checks`, then `gh run view --log-failed` on a failing run; GitLab: the MR's pipeline and its failed job traces). Failing CI is reason enough to act on its own — check it even when there were no conflicts and no review comments.

- For each failing check, read the failing job's logs and diagnose the cause.
- Fix the failures this branch is responsible for and can confidently fix, folding the fix into the same change batch as the conflict and comment work.
- Retry the failed jobs when a failure is infra, flaky, or otherwise external rather than a real problem in the code (GitHub: `gh run rerun --failed`; GitLab: retry the job or pipeline). Don't wait for the retry to finish.
- Leave the rest — failures that also fail on the base branch, or that you can't confidently fix or clear — and report them instead of chasing them.

## 4. Push and report

- Push once, carrying the conflict resolution, comment fixes, and CI fixes together. If there was genuinely nothing to change — no conflicts, no actionable comments, no fixable CI failures — don't push a no-op; a job retry stands on its own and needs no push.
- Give the user a short report: what changed, what you pushed back on, which conflicts you resolved (calling out anything uncertain), and how each failing check was handled — fixed, retried, or left (and why). If you pushed, a fresh CI run starts on the new commit; the skill doesn't wait for it.
