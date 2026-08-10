---
name: review-pr
description: Use when asked to review a specific GitHub pull request or GitLab merge request (given by number or URL) and post the code-review findings back on the PR, approving it only when the review comes back clean — as opposed to reviewing the local working diff.
---

# Review a Pull Request

Do a thorough code review of the provided pull request (GitHub) or merge request (GitLab).

Review the code, not the pipeline. CI belongs to the address-pr skill — don't pull up the PR's check or pipeline status, and never turn a failing, pending, or missing check into a finding, a line in the review body, or a remark to the user: a red pipeline is not a defect in the diff. It doesn't gate the verdict either — a diff you found nothing wrong with is clean and gets approved, whatever the checks are doing.

A review is **one inline comment per finding, and nothing else** — each anchored to the specific file and line it refers to, all submitted together as a single review (GitHub: a PR review whose per-line review comments each open a thread; GitLab: resolvable discussions positioned on the MR diff). The threads are the review, and nothing summarizes them. GitHub's API rejects a review submitted with a blank body, so give it a single line naming your identity (AI agent name) — that line is not a summary slot, so don't recap the findings in it. (The only other thing that ever belongs in the body is a finding with no line to anchor to, per the anchoring rule below.) GitLab needs no such body: post the discussions and no accompanying note.

Do **not** dump every finding into one top-level PR comment. Top-level PR/issue comments are invisible to Claude's "address comments" auto-fix and to the address-pr skill — both act only on unresolved *review threads*. Inline comments are what make each finding individually addressable and resolvable.

- Prefix each inline comment with its severity (P0/P1/P2) and your identity (AI agent name).
- Anchor every finding you can, including the ones that look like they're about untouched code: a change that duplicates, breaks, or fails to update code elsewhere is really about the diff line that introduces that relationship — anchor it there and name the other code in the comment text. An anchored finding opens a resolvable thread, and that is what makes it addressable at all.
- Send a finding top-level only when it genuinely has no line in the diff — it's about the PR as a whole (no tests at all, two unrelated changes bundled together). Keep even that one resolvable where the platform allows:
  - **GitLab** — post it as its own resolvable discussion: an MR discussion is resolvable even with no diff position, so it stays addressable like any inline finding. Create it via the discussions API (`POST projects/<id>/merge_requests/<iid>/discussions`, per Posting mechanics below) or `glab mr note create`, whose default is a resolvable thread — **not** a plain note (`--resolvable=false`), which can't be resolved and which the address-pr flow can't see.
  - **GitHub** — it goes in the review body. A GitHub review thread must sit on a diff line, so a finding without one cannot be a resolvable thread there; the body is its only home. That gap is exactly why you anchor everything you can.
- On low-priority (P2) findings, add a line letting the author know they can resolve the thread themselves if they decide it isn't worth fixing. A P2 is a nit, not a blocker — self-resolving is how the author declines one deliberately, so it doesn't linger as an unaddressed thread.
- If the review turns up any finding at all — even a single P2 — post the findings as above and do **not** approve. The PR gets approved only once it's genuinely clean.
- On GitHub, pick the review event by the worst finding in it: submit as **`REQUEST_CHANGES`** when any finding is a P0 or P1, and as **`COMMENT`** when every finding is a P2 — a nit shouldn't formally block the PR. GitHub refuses `REQUEST_CHANGES` on your own PR just as it refuses self-approval, so when the reviewer is also the author and it's rejected, retry the same review as `COMMENT`; the inline threads still carry the findings. GitLab has no per-review event — there the unresolved discussions are themselves the block.
- When the review is spotless — not one finding — approve it instead of opening any threads, with a one-line body that includes your identity (AI agent name). That identity isn't decoration: it is the only thing that tells a babysit-pr watch your verdict from a person's or another agent's, so an approval never goes out without it. An approval is a verdict, not a finding — it goes in a formal approval or a top-level comment, never an inline thread (the inline-thread rule above is only for findings). The two platforms differ:
  - **GitHub** — approve formally with `gh pr review <pr> --approve --body '<one line naming your identity>'`. When that's refused — GitHub won't let you approve your own PR, and in this setup the reviewer is often also the author (permissions or approval rules can block it too) — fall back to a top-level comment that clearly says **APPROVED**.
  - **GitLab** — do **not** cast a formal approval (`glab mr approve`); it registers the reviewer bot as an official approver and counts toward the MR's approval rules, and we only want a visible verdict here. Post a plain MR comment (a note, not a discussion) that clearly says **APPROVED** instead — always a comment, never the formal approve. Keep it non-resolvable: `glab mr note create` defaults to a *resolvable* thread, so pass `--resolvable=false` (or use the notes API). A verdict is nothing for the author to resolve — findings are threads, verdicts are notes.

  Both a formal approval and an APPROVED comment are what the babysit-pr skill reads as approved, so this is the signal that ends its watch — but only once it can tell the verdict is its own agent's, which it does by reading the identity out of the body. A verdict that doesn't name its agent is unattributable, and the watch runs straight past it.

## Re-reviewing a PR you have already reviewed

Usually the PR already carries threads from an earlier review of yours — this skill runs again on every push. A re-review is mostly about those threads, not about restating them.

**An open thread is the finding.** It says "this still stands" for as long as it stays open, and nothing else needs to say it. A new push is not a reason to comment on an open thread: a "still not fixed" note on every push tells the author nothing the open thread didn't, and it buries the comments that do need reading. Silence on an open thread means the finding stands.

Take each thread you left open and judge it against the current code:

- **Fixed** — resolve the thread. Resolving is the reviewer's job, not the author's: the address-pr flow replies and deliberately leaves threads open for you to verify and close. Resolve every thread that is genuinely settled — the fix landed, or you have accepted the author's pushback — and only those.
- **Nothing about this finding moved** — the push went elsewhere (another thread's fix, a CI fix, a base sync, unrelated work) and this finding simply wasn't addressed. Leave the thread open and post nothing at all.
- **It was addressed and fell short** — a partial fix, one that misses a case, one that relocates the problem or introduces a new one in the same place, or a reply claiming a fix that isn't in the code. The author acted and doesn't yet know it didn't work, so that is worth a comment: reply on the thread with what is still wrong. Say it once, for that attempt — if a later push leaves the finding untouched again, that's the silent case above, not a repeat.
- **The author asked you something** — a direct question, or an argument that the finding is wrong (what address-pr does when it judges a comment mistaken). Answer that on its own thread: concede and resolve when the pushback is right, or hold the finding, with your reasoning, when it isn't. A reply that only reports what they did ("fixed in `abc123`") asks nothing — judge that one by the code, per the cases above.

Never open a second thread for a finding that already has one — reply on the existing thread instead of duplicating it. And review only what is new since your last review; don't re-audit unchanged code or re-post findings you have already made.

A commit that merely syncs the branch with its base — a merge of master/main in, or a rebase onto it — is **not** new code: it leaves the PR's introduced diff (`base...head`, e.g. `gh pr diff` / `glab mr diff`) unchanged. Judge by that introduced diff rather than the commit list, since a rebase rewrites every SHA without touching a line. When such a base-sync is all that has landed since your last review, there is nothing to re-review — leave nothing at all: no findings, no summary, no fresh APPROVED comment.

Then re-audit the findings still standing. If none remain, approve the current state per the clean-review rule above — unless **you** have already approved this same state, in which case post nothing. Approve even when no code changed: the approval is the final clean verdict, not a re-review of unchanged code. If any finding still stands, don't approve. Only your own verdict counts as already-approved — a person's Approve, or another agent's, doesn't stand in for yours (read the identity off it the way babysit-pr's *Whose approval counts* describes), and treating someone else's as yours would leave the PR with no verdict from you, which a babysit-pr watch never stops on.

## Posting mechanics

Review text is made of backticked code spans and `$`, and a shell will happily run and expand them. Keep review text out of the shell entirely:

1. Build the request body as a JSON file with the Write tool (the scratchpad directory is fine). The Write tool never goes through a shell, so nothing in the text can execute.
2. Post that file with `--input`, which both CLIs take:
   - **GitHub** — `gh api --method POST /repos/<owner>/<repo>/pulls/<n>/reviews --input body.json`
   - **GitLab** — `glab api --method POST projects/<id>/merge_requests/<iid>/discussions --input body.json`

- Don't assemble the payload from `-f`/`-F`/`--raw-field`. They can't express a nested array or object, which is exactly what GitHub's `comments` array and GitLab's `position` object are — glab's own help spells this out: those flags send JSON arrays and objects as strings, and `--input` is what passes a body literally.
- If you do reach for a heredoc, quote the delimiter — `<<'EOF'`, never bare `<<EOF`. Unquoted, the shell executes backticks and expands `$` *inside* the heredoc, so a comment reading ``use `secure_compare` instead of `==` `` runs `secure_compare` as a command and posts its output. Quoting the delimiter turns all of it off.
- The Write tool solves *shell* quoting, not *JSON* escaping. Inside a JSON string a newline is `\n`, a double quote is `\"`, a backslash is `\\` — a raw newline is invalid JSON and the API will reject it.
- Anchor to lines that are actually in the diff: GitHub 422s a `line` outside it, and a positioned GitLab discussion needs the MR's `diff_refs` (`base_sha`/`start_sha`/`head_sha`, read from `glab api projects/<id>/merge_requests/<iid>`) in its `position`.
- On re-review, a reply to an existing thread is not a new review, and resolving is a separate call again — **GitHub:** reply with `POST /repos/<owner>/<repo>/pulls/<n>/comments/<comment_id>/replies`, resolve with the `resolveReviewThread` GraphQL mutation on the thread id from a `reviewThreads` query; **GitLab:** reply with `POST projects/<id>/merge_requests/<iid>/discussions/<discussion_id>/notes`, resolve with `PUT projects/<id>/merge_requests/<iid>/discussions/<discussion_id>` and `resolved=true`.

## Posting identity (reviewer bot)

Post the review under a dedicated reviewer-bot account when one is configured, so findings aren't attributed to your own account. This covers only the commands that *post* — submitting the review and its inline comments, and approving (a formal approval or a top-level APPROVED comment). Reading the PR and its diff can use whatever account is already active.

- **GitHub:** if `PR_REVIEW_GH_TOKEN` is set, run the posting `gh` commands with it exported as `GH_TOKEN`, e.g. `GH_TOKEN="$PR_REVIEW_GH_TOKEN" gh api --method POST /repos/<owner>/<repo>/pulls/<n>/reviews --input body.json` (and `gh pr review --approve` in the clean case).
- **GitLab:** if `PR_REVIEW_GITLAB_TOKEN` is set, run the posting `glab` commands with it exported as `GITLAB_TOKEN`.
- If the relevant variable is unset, post with your default authenticated account, exactly as before.
- Test whether the variable is set **without printing it** — `[ -n "${PR_REVIEW_GH_TOKEN:-}" ] && echo set || echo unset`, or `${PR_REVIEW_GH_TOKEN:+set}`. Never `echo "$PR_REVIEW_GH_TOKEN"`, and never `${PR_REVIEW_GH_TOKEN:-unset}`: `:-` substitutes the token's **real value** whenever it is set, dumping the credential into the transcript — `:+` is the one that yields a placeholder. Pass the token by name, as above, so its value never reaches a command line, a log, or your output.

Scope it to the posting commands only — don't export it for the whole session or apply it in the address-pr / babysit-pr flows, which push code and reply as you and must keep using your own account.
