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
- When the review is spotless — not one finding — approve it instead of opening any threads, with a one-line body that includes your identity (AI agent name). An approval is a verdict, not a finding — it goes in a formal approval or a top-level comment, never an inline thread (the inline-thread rule above is only for findings). The two platforms differ:
  - **GitHub** — approve formally with `gh pr review <pr> --approve`. When that's refused — GitHub won't let you approve your own PR, and in this setup the reviewer is often also the author (permissions or approval rules can block it too) — fall back to a top-level comment that clearly says **APPROVED**.
  - **GitLab** — do **not** cast a formal approval (`glab mr approve`); it registers the reviewer bot as an official approver and counts toward the MR's approval rules, and we only want a visible verdict here. Post a plain MR comment (a note, not a discussion) that clearly says **APPROVED** instead — always a comment, never the formal approve. Keep it non-resolvable: `glab mr note create` defaults to a *resolvable* thread, so pass `--resolvable=false` (or use the notes API). A verdict is nothing for the author to resolve — findings are threads, verdicts are notes.

  Both a formal approval and an APPROVED comment are what the babysit-pr skill reads as approved, so this is the signal that ends its watch.

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

## Posting identity (reviewer bot)

Post the review under a dedicated reviewer-bot account when one is configured, so findings aren't attributed to your own account. This covers only the commands that *post* — submitting the review and its inline comments, and approving (a formal approval or a top-level APPROVED comment). Reading the PR and its diff can use whatever account is already active.

- **GitHub:** if `PR_REVIEW_GH_TOKEN` is set, run the posting `gh` commands with it exported as `GH_TOKEN`, e.g. `GH_TOKEN="$PR_REVIEW_GH_TOKEN" gh api --method POST /repos/<owner>/<repo>/pulls/<n>/reviews --input body.json` (and `gh pr review --approve` in the clean case).
- **GitLab:** if `PR_REVIEW_GITLAB_TOKEN` is set, run the posting `glab` commands with it exported as `GITLAB_TOKEN`.
- If the relevant variable is unset, post with your default authenticated account, exactly as before.
- Test whether the variable is set **without printing it** — `[ -n "${PR_REVIEW_GH_TOKEN:-}" ] && echo set || echo unset`, or `${PR_REVIEW_GH_TOKEN:+set}`. Never `echo "$PR_REVIEW_GH_TOKEN"`, and never `${PR_REVIEW_GH_TOKEN:-unset}`: `:-` substitutes the token's **real value** whenever it is set, dumping the credential into the transcript — `:+` is the one that yields a placeholder. Pass the token by name, as above, so its value never reaches a command line, a log, or your output.

Scope it to the posting commands only — don't export it for the whole session or apply it in the address-pr / babysit-pr flows, which push code and reply as you and must keep using your own account.
