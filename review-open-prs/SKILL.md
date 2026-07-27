---
name: review-open-prs
description: Use when asked to review all open pull requests / merge requests in a repository at once (rather than one specific PR) and post code-review findings back on each — enumerates every open non-draft PR and reviews each the way review-pr handles a single one (skipping any unchanged since its last review), and keeps sweeping on a loop that backs off as things go quiet, until the repo has had no new activity for hours.
---

# Review All Open Pull Requests

Find every open, non-draft pull request (GitHub) or merge request (GitLab) in the current repository, then review each one — dispatching a subagent per PR so the reviews run in parallel.

## Keep watching on a loop

Run this as a repeating watch, not a single sweep, so PRs get re-reviewed as they gain commits and replies and newly-opened PRs get picked up. Drive it with the self-paced watch the babysit-pr skill specifies, and take that whole thing as written — read its *Waiting between rounds* section and its *When to stop* rules and apply them here: the non-blocking scheduled resume (never a foreground `sleep`), the wait that backs off while nothing is happening and snaps back on any update, the state carried across wake-ups, the first pass run immediately, the silence on quiet passes, and the graceful fallback where no scheduled resume exists. Its *Gone quiet* and *Hard error* stops both apply, along with the carve-out that comes with them: transient trouble — a rate limit, a flaky network, one failed API call — is a no-op pass, not a stop. Only *Approved* and *Merged or closed* don't carry over; those judge a single PR, not a whole repo. The two watches are otherwise deliberately identical, so don't invent a different cadence or cap for this one — babysit-pr is the single source of both.

Two things differ here:

- **What counts as an "update"** — any pass with something to do: a PR reviewed (new or newly-changed), replies answered, or a newly-opened PR seen. A pass where every PR is skipped as unchanged is a quiet round. Refresh the last-update timestamp whenever a pass does something; the backoff and the quiet cap both follow from that timestamp exactly as babysit-pr describes.
- **How far an error reaches** — a hard error ends the watch only when it breaks the *whole* watch: auth fails, the reviewer-bot token is revoked, the repo is gone. A single PR failing is not that. When one PR errors or vanishes mid-pass, drop that PR, say so, and keep sweeping the rest — a repo-wide watch shouldn't die because one of its subagents hit a wall. Don't retry a dead PR every pass either; leave it dropped and report it at the end.

Before reviewing a PR, work out what it still owes you, from two independent checks. **New code:** compare the current head against the commit you last reviewed. **Unanswered replies:** go thread by thread and find every thread whose *last comment is not yours* — those are waiting on you, however long they have been sitting there. Skip the PR only when both come back empty. Always review PRs you have never reviewed.

Decide the replies question by last commenter, never by a timestamp. Comparing "the most recent review I left" against "the most recent reply on the PR" collapses per-thread state into two numbers, and then any thread whose reply is older than your latest action *anywhere on the PR* hides behind it: answer one thread, and every older unanswered thread reads as caught up — permanently, because your own timestamp only moves forward. The same goes for asking which replies arrived since the last round, which silently drops anything that landed while you were busy elsewhere in that round. Last-commenter has no such blind spot: a thread you have not answered keeps failing the test every round until you answer it. On GitHub that is one `reviewThreads` query and a look at `comments.nodes[-1].author`; on GitLab, the last note on each unresolved discussion.

Count only genuine new code, though. A commit that merely syncs the branch with its base branch — a merge of master/main into the branch, or a rebase onto it — is **not** new code: it leaves the PR's introduced diff (`base...head`, e.g. `gh pr diff` / `glab mr diff`) unchanged. Judge by that introduced diff, not the raw commit list, since a rebase rewrites every commit SHA without touching a line of code. When such a base-sync is the only thing that has landed since your last review, there is nothing to re-review — leave nothing at all: no findings, no summary, no new APPROVED comment. (If any thread is still waiting on you, answer it, per the cases below.)

A PR can need you again with no new commit at all: instead of pushing code, the author often pushes back on a finding in its thread, explaining why they did not make the change — exactly what the address-pr skill does when it judges a comment wrong. Going by commits alone leaves that pushback unanswered forever.

Resolving threads is your job as reviewer, not the author's: the address-pr flow replies on each thread but leaves it open, so on re-review you resolve every thread that's been settled — the finding's fix has landed, or you've accepted the author's pushback — and leave open only the ones that still stand.

So when you pick a PR back up:

- **New commits** — review the new code, following the conventions below, and resolve each earlier thread whose finding the new code has genuinely fixed; leave a thread open when the change doesn't actually settle it, saying what's still wrong.
- **Unanswered replies, no new code** — don't re-review unchanged code and don't re-post findings you already made. Answer every thread whose last comment is not yours, oldest first, on its own thread: concede and resolve the thread when the pushback is right, or hold the finding, with your reasoning, when it isn't. "Unanswered" is the whole backlog, not just what arrived since the last round.
- **Both** — do both.

After answering the replies and resolving the threads that are now settled, re-audit every finding you left. If none remain, approve the current head/diff using the review-pr flow below unless that state is already approved. Do this even when no code changed: the approval is the final clean verdict, not a re-review of unchanged code. If any finding remains, do not approve.

Never open a second thread for a finding that already has one — reply on the existing thread rather than duplicating it.

Each subagent handles one pull request. When there is new code to review, it does a thorough code review, following the same conventions as the review-pr skill: post each finding as an inline review comment anchored to its file and line — submitted as one review, with nothing summarizing the findings alongside them — never as a single top-level PR comment, which the "address comments" auto-fix and the address-pr skill cannot detect. Prefix each finding with its severity (P0/P1/P2). If that review turns up no findings, it approves the PR the way review-pr does — on GitHub a formal approval (falling back to an APPROVED comment when that's refused), on GitLab a top-level APPROVED comment only, never a formal `glab mr approve`. Every comment includes your identity (AI agent name). Post under the reviewer-bot account when one is configured, the same way review-pr's "Posting identity" section describes — exporting `PR_REVIEW_GH_TOKEN` (GitHub) or `PR_REVIEW_GITLAB_TOKEN` (GitLab) for just the posting commands when the variable is set.

Skip draft PRs. Stay quiet on a pass with nothing to do; when the watch ends, tell the user why it stopped and, across the whole watch, which PRs you reviewed, which you only answered replies on, which never changed, and any you dropped on an error (with what failed).

Whenever you name a PR/MR in anything the user reads — progress notes and the closing summary alike — give it as a clickable markdown link to the PR/MR's full URL (e.g. `[!170](https://gitlab.domain.com/group/repo/-/merge_requests/170)`), never a bare number or title.
