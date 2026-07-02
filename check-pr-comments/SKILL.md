---
name: check-pr-comments
description: Use when asked to check and address review comments on a pull request or merge request — reads the open/unresolved comments on a GitHub PR or GitLab MR (the current branch's, or one given by number/URL), makes the code changes to resolve the actionable ones, replies on each thread, and pushes. The counterpart to review-pr, which posts comments rather than addressing them.
---

# Check and Address PR Comments

Find the open review comments and unresolved threads on the pull request (GitHub) or merge request (GitLab) — the one for the current branch, or the one given by number/URL — and work through them:

- Understand each comment. Apply technical rigor: don't blindly implement. Make the code change when the comment is right; when it's wrong, misguided, or based on a misunderstanding, don't make a bad change — explain your reasoning in the reply instead. When a comment is genuinely ambiguous or a judgment call, ask the user rather than guessing.
- For each actionable comment, make the code change. Group related comments so a single change can resolve several threads.
- Reply on each thread saying what you did, or why you didn't. Resolve threads you've addressed. Replies should include your identity (AI agent name).
- Skip already-resolved threads and comments that are just approval or acknowledgement.
- Once the comments are addressed, push and give the user a short report of what changed and what you pushed back on.
