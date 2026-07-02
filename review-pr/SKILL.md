---
name: review-pr
description: Use when asked to review a specific GitHub pull request or GitLab merge request (given by number or URL) and post the code-review findings back on the PR — as opposed to reviewing the local working diff.
---

# Review a Pull Request

Do a thorough code review of the provided pull request (GitHub) or merge request (GitLab).

Post each finding as an **inline review comment** anchored to the specific file and line it refers to, submitted together as a single review (GitHub: a PR review whose per-line review comments each open a thread; GitLab: resolvable discussions positioned on the MR diff). Put a short severity-ordered summary in the review body, but keep the actual findings in the inline comments — one thread per finding.

Do **not** dump every finding into one top-level PR comment. Top-level PR/issue comments are invisible to Claude's "address comments" auto-fix and to the check-pr-comments skill — both act only on unresolved *review threads*. Inline comments are what make each finding individually addressable and resolvable.

- Prefix each inline comment with its severity (P0/P1/P2) and your identity (AI agent name).
- Anchor every finding to the most relevant line in the diff. Put any finding that isn't tied to a specific line in the review-body summary instead.
- If there are no findings and everything looks good, don't open any threads — just leave a short top-level comment (or approve) saying you did not find anything, including your identity.
