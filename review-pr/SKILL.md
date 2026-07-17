---
name: review-pr
description: Use when asked to review a specific GitHub pull request or GitLab merge request (given by number or URL) and post the code-review findings back on the PR, approving it only when the review comes back clean — as opposed to reviewing the local working diff.
---

# Review a Pull Request

Do a thorough code review of the provided pull request (GitHub) or merge request (GitLab).

A review is **one inline comment per finding, and nothing else** — each anchored to the specific file and line it refers to, all submitted together as a single review (GitHub: a PR review whose per-line review comments each open a thread; GitLab: resolvable discussions positioned on the MR diff). The threads are the review, and nothing summarizes them. GitHub's API rejects a review submitted with a blank body, so give it a single line naming your identity (AI agent name) — that line is not a summary slot, so don't recap the findings in it. GitLab needs no such body: post the discussions and no accompanying note.

Do **not** dump every finding into one top-level PR comment. Top-level PR/issue comments are invisible to Claude's "address comments" auto-fix and to the address-pr skill — both act only on unresolved *review threads*. Inline comments are what make each finding individually addressable and resolvable.

- Prefix each inline comment with its severity (P0/P1/P2) and your identity (AI agent name).
- Anchor every finding to the most relevant line in the diff. Rarely, a finding has no line to anchor to — it's about the PR as a whole, or about code the diff never touches. Only that finding goes top-level instead: state it in the GitHub review body, or as a plain GitLab MR comment (a note, e.g. `glab mr note`, **not** a discussion, so it doesn't sit there as its own unresolved thread). If a finding can be anchored, anchor it.
- On low-priority (P2) findings, add a line letting the author know they can resolve the thread themselves if they decide it isn't worth fixing. A P2 is a nit, not a blocker — self-resolving is how the author declines one deliberately, so it doesn't linger as an unaddressed thread.
- If the review turns up any finding at all — even a single P2 — post the findings as above and do **not** approve. The PR gets approved only once it's genuinely clean.
- When the review is spotless — not one finding — approve it instead of opening any threads, with a one-line body that includes your identity (AI agent name). An approval is a verdict, not a finding — it goes in a formal approval or a top-level comment, never an inline thread (the inline-thread rule above is only for findings). The two platforms differ:
  - **GitHub** — approve formally with `gh pr review <pr> --approve`. When that's refused — GitHub won't let you approve your own PR, and in this setup the reviewer is often also the author (permissions or approval rules can block it too) — fall back to a top-level comment that clearly says **APPROVED**.
  - **GitLab** — do **not** cast a formal approval (`glab mr approve`); it registers the reviewer bot as an official approver and counts toward the MR's approval rules, and we only want a visible verdict here. Post a plain MR comment (a note, not a discussion) that clearly says **APPROVED** instead — always a comment, never the formal approve.

  Both a formal approval and an APPROVED comment are what the babysit-pr skill reads as approved, so this is the signal that ends its watch.

## Posting identity (reviewer bot)

Post the review under a dedicated reviewer-bot account when one is configured, so findings aren't attributed to your own account. This covers only the commands that *post* — submitting the review and its inline comments, and approving (a formal approval or a top-level APPROVED comment). Reading the PR and its diff can use whatever account is already active.

- **GitHub:** if `PR_REVIEW_GH_TOKEN` is set, run the posting `gh` commands with it exported as `GH_TOKEN`, e.g. `GH_TOKEN="$PR_REVIEW_GH_TOKEN" gh api --method POST /repos/<owner>/<repo>/pulls/<n>/reviews …` (and `gh pr review --approve` in the clean case).
- **GitLab:** if `PR_REVIEW_GITLAB_TOKEN` is set, run the posting `glab` commands with it exported as `GITLAB_TOKEN`.
- If the relevant variable is unset, post with your default authenticated account, exactly as before.

Scope it to the posting commands only — don't export it for the whole session or apply it in the address-pr / babysit-pr flows, which push code and reply as you and must keep using your own account.
