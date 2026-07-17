---
name: review-open-prs
description: Use when asked to review all open pull requests / merge requests in a repository at once (rather than one specific PR) and post code-review findings back on each — enumerates every open non-draft PR and reviews each the way review-pr handles a single one (skipping any unchanged since its last review), and keeps sweeping on a loop that backs off as things go quiet, until the repo has had no new activity for hours.
---

# Review All Open Pull Requests

Find every open, non-draft pull request (GitHub) or merge request (GitLab) in the current repository, then review each one — dispatching a subagent per PR so the reviews run in parallel.

## Keep watching on a loop

Run this as a repeating watch, not a single sweep, so PRs get re-reviewed as they gain commits and replies and newly-opened PRs get picked up. Drive it with the self-paced watch the babysit-pr skill specifies, and take that whole thing as written — read its *Waiting between rounds* section and its *Gone quiet* stop rule and apply them here: the non-blocking scheduled resume (never a foreground `sleep`), the wait that backs off while nothing is happening and snaps back on any update, the state carried across wake-ups, the first pass run immediately, the silence on quiet passes, and the graceful fallback where no scheduled resume exists. The two watches are deliberately identical, so don't invent a different cadence or cap for this one — babysit-pr is the single source of both.

Only one thing differs here: what counts as an "update". For a whole-repo watch it's any pass with something to do — a PR reviewed (new or newly-changed), replies answered, or a newly-opened PR seen — so a pass where every PR is skipped as unchanged is a quiet round. Refresh the last-update timestamp whenever a pass does something; the backoff and the quiet cap both follow from that timestamp exactly as babysit-pr describes.

Before reviewing a PR, check whether it has already been reviewed: look for the most recent review you left under your identity (AI agent name), then compare its timestamp against both the PR's latest commit and the latest reply on the review threads you left. Skip the PR only when neither is newer — no new code, and nobody has come back to you. Always review PRs you have never reviewed.

Count only genuine new code, though. A commit that merely syncs the branch with its base branch — a merge of master/main into the branch, or a rebase onto it — is **not** new code: it leaves the PR's introduced diff (`base...head`, e.g. `gh pr diff` / `glab mr diff`) unchanged. Judge by that introduced diff, not the raw commit list, since a rebase rewrites every commit SHA without touching a line of code. When such a base-sync is the only thing that has landed since your last review, there is nothing to re-review — leave nothing at all: no findings, no summary, no new APPROVED comment. (If new replies also landed, still answer those, per the cases below.)

A PR can need you again with no new commit at all: instead of pushing code, the author often pushes back on a finding in its thread, explaining why they did not make the change — exactly what the address-pr skill does when it judges a comment wrong. Going by commits alone leaves that pushback unanswered forever.

So when you pick a PR back up:

- **New commits** — review the new code, following the conventions below.
- **New replies, no new code** — don't re-review unchanged code and don't re-post findings you already made. Answer each new reply on its own thread: concede and resolve the thread when the pushback is right, or hold the finding, with your reasoning, when it isn't.
- **Both** — do both.

After answering the replies, re-audit every finding you left. If none remain, approve the current head/diff using the review-pr flow below unless that state is already approved. Do this even when no code changed: the approval is the final clean verdict, not a re-review of unchanged code. If any finding remains, do not approve.

Never open a second thread for a finding that already has one — reply on the existing thread rather than duplicating it.

Each subagent handles one pull request. When there is new code to review, it does a thorough code review, following the same conventions as the review-pr skill: post each finding as an inline review comment anchored to its file and line — submitted as one review, with nothing summarizing the findings alongside them — never as a single top-level PR comment, which the "address comments" auto-fix and the address-pr skill cannot detect. Prefix each finding with its severity (P0/P1/P2). If that review turns up no findings, it approves the PR the way review-pr does — on GitHub a formal approval (falling back to an APPROVED comment when that's refused), on GitLab a top-level APPROVED comment only, never a formal `glab mr approve`. Every comment includes your identity (AI agent name). Post under the reviewer-bot account when one is configured, the same way review-pr's "Posting identity" section describes — exporting `PR_REVIEW_GH_TOKEN` (GitHub) or `PR_REVIEW_GITLAB_TOKEN` (GitLab) for just the posting commands when the variable is set.

Skip draft PRs. Stay quiet on a pass with nothing to do; when the watch ends, tell the user why it stopped and, across the whole watch, which PRs you reviewed, which you only answered replies on, and which never changed.

Whenever you name a PR/MR in anything the user reads — progress notes and the closing summary alike — give it as a clickable markdown link to the PR/MR's full URL (e.g. `[!170](https://gitlab.domain.com/group/repo/-/merge_requests/170)`), never a bare number or title.
