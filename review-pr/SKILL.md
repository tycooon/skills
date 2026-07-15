---
name: review-pr
description: Use when asked to review a specific GitHub pull request or GitLab merge request (given by number or URL) and post the code-review findings back on the PR — as opposed to reviewing the local working diff.
---

# Review a Pull Request

Do a thorough code review of the provided pull request (GitHub) or merge request (GitLab).

Post each finding as an **inline review comment** anchored to the specific file and line it refers to, submitted together as a single review (GitHub: a PR review whose per-line review comments each open a thread; GitLab: resolvable discussions positioned on the MR diff). Put a short severity-ordered summary in the review body, but keep the actual findings in the inline comments — one thread per finding.

Do **not** dump every finding into one top-level PR comment. Top-level PR/issue comments are invisible to Claude's "address comments" auto-fix and to the address-pr skill — both act only on unresolved *review threads*. Inline comments are what make each finding individually addressable and resolvable.

- Prefix each inline comment with its severity (P0/P1/P2) and your identity (AI agent name).
- Anchor every finding to the most relevant line in the diff. Put any finding that isn't tied to a specific line in the review-body summary instead.
- If there are no findings and everything looks good, don't open any threads — just leave a short top-level comment (or approve) saying you did not find anything, including your identity.

## Posting identity (reviewer bot)

Post the review under a dedicated reviewer-bot account when one is configured, so findings aren't attributed to your own account. This covers only the commands that *post* — creating the review and its inline comments, approving, or the top-level "looks good" comment. Reading the PR and its diff can use whatever account is already active.

- **GitHub:** if `PR_REVIEW_GH_TOKEN` is set, run the posting `gh` commands with it exported as `GH_TOKEN`, e.g. `GH_TOKEN="$PR_REVIEW_GH_TOKEN" gh api --method POST /repos/<owner>/<repo>/pulls/<n>/reviews …` (and `gh pr review --approve` in the clean case).
- **GitLab:** if `PR_REVIEW_GITLAB_TOKEN` is set, run the posting `glab` commands with it exported as `GITLAB_TOKEN`.
- If the relevant variable is unset, post with your default authenticated account, exactly as before.

Scope it to the posting commands only — don't export it for the whole session or apply it in the address-pr / babysit-pr flows, which push code and reply as you and must keep using your own account.
