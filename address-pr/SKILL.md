---
name: address-pr
description: Use when asked to address a pull request or merge request — resolves merge conflicts with the base branch if present, then reads and addresses the open/unresolved review comments (making code changes, replying on each thread, resolving them), and pushes. Works on the current branch's PR or one given by number/URL, on GitHub or GitLab. The counterpart to review-pr, which posts comments rather than addressing them.
---

# Address a Pull Request

Get a pull request (GitHub) or merge request (GitLab) — the one for the current branch, or the one given by number/URL — ready by resolving conflicts and working through its review comments, then pushing once.

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

## 3. Push and report

- Push once, carrying both the conflict resolution and the comment fixes. If there was nothing to do — no conflicts and no actionable comments — don't push a no-op.
- Give the user a short report: what changed, what you pushed back on, and which conflicts you resolved (calling out anything uncertain).
