---
name: review-open-prs
description: Use when asked to review all open pull requests / merge requests in a repository at once (rather than one specific PR) and post code-review findings back on each — enumerates every open non-draft PR and reviews each the way review-pr handles a single one (skipping any unchanged since its last review), and keeps sweeping on a ~5-minute loop until an hour passes with no new activity on any PR.
---

# Review All Open Pull Requests

Find every open, non-draft pull request (GitHub) or merge request (GitLab) in the current repository, then review each one — dispatching a subagent per PR so the reviews run in parallel.

## Keep watching on a loop

Run this as a repeating watch, not a single sweep: make a fresh pass about every 5 minutes, so PRs get re-reviewed as they gain commits and replies and newly-opened PRs get picked up. Drive the loop with the same self-paced watch mechanism the babysit-pr skill uses — schedule a non-blocking resume ~5 minutes out (no foreground `sleep`), carry your state across wake-ups, run the first pass immediately, stay silent on quiet passes, and fall back gracefully where no scheduled resume exists; see babysit-pr for those details. Stop on the same rule as babysit-pr: quit once about an hour has passed with no update anywhere. For a whole-repo watch, an "update" is any pass with something to do — a PR reviewed (new or newly-changed), replies answered, or a newly-opened PR seen — so a pass where every PR is skipped as unchanged is a quiet round, and about an hour of consecutive quiet rounds ends the watch. Carry the timestamp of the last pass that had an update across wake-ups, and refresh it whenever a pass does something.

Before reviewing a PR, check whether it has already been reviewed: look for the most recent review you left under your identity (AI agent name), then compare its timestamp against both the PR's latest commit and the latest reply on the review threads you left. Skip the PR only when neither is newer — no new code, and nobody has come back to you. Always review PRs you have never reviewed.

A PR can need you again with no new commit at all: instead of pushing code, the author often pushes back on a finding in its thread, explaining why they did not make the change — exactly what the address-pr skill does when it judges a comment wrong. Going by commits alone leaves that pushback unanswered forever.

So when you pick a PR back up:

- **New commits** — review the new code, following the conventions below.
- **New replies, no new code** — don't re-review unchanged code and don't re-post findings you already made. Answer each new reply on its own thread: concede and resolve the thread when the pushback is right, or hold the finding, with your reasoning, when it isn't.
- **Both** — do both.

Never open a second thread for a finding that already has one — reply on the existing thread rather than duplicating it.

Each subagent handles one pull request. When there is new code to review, it does a thorough code review, following the same conventions as the review-pr skill: post each finding as an inline review comment anchored to its file and line — submitted as one review with a short severity-ordered summary in the body — never as a single top-level PR comment, which the "address comments" auto-fix and the address-pr skill cannot detect. Prefix each finding with its severity (P0/P1/P2). If that review turns up no findings, it approves the PR the way review-pr does — a formal approval when the platform allows, otherwise a clear top-level APPROVED comment. Every comment includes your identity (AI agent name).

Skip draft PRs. Stay quiet on a pass with nothing to do; when the watch ends, tell the user why it stopped and, across the whole watch, which PRs you reviewed, which you only answered replies on, and which never changed.
