---
name: review-loop
description: "Keep reviewing one PR or MR in this session: review it with review-pr, wait in the background for a new head or reply, and review again until it merges, closes or goes quiet."
---

# Keep Reviewing One Pull Request

Resolve the URL, `repo#N`, `repo!N` or bare number (in the current repository) once and use the canonical PR or MR URL for the whole loop. Run each review in this session with its model and account, following review-pr's posting identity rules.

## Check for another reviewer

Before the first round, check `review-scan list` and `review-scan status <watch>` for a local watch covering the target. Read recent verdicts on the PR for another AI reviewer's identity line or bot account. If another reviewer already covers the PR, tell the user and hold off on posting. Offer to wait for and report that reviewer's verdict with `wait-pr-ready --until-review`. Proceed with your own review only if the user explicitly asks you to continue.

## Run a round

Run one [review-pr](../review-pr/SKILL.md) pass in full, including *Re-reviewing a PR you have already reviewed*. Follow its coverage check and resolve settled threads as directed. A covered head gets no re-review or post. Record the full head SHA the pass evaluated, even when an earlier verdict covers it.

## Wait for a change

Run `~/.claude/bin/review-scan wait <url> --head <head> --after <activity mark>` as one background command. In Claude Code, use `run_in_background` so the exit re-invokes this session. Pass the mark the round started from: the previous waiter's exit line, or for the first round your own PR read before reviewing. Never re-read the PR after the round's own posts or thread resolutions to take a fresher mark: a reply that landed in between would fold into it and never wake the waiter. Your own posts then wake one extra round, which review-pr's coverage and last-commenter checks settle without a post, and its exit line gives the next mark. Always take the mark from the PR, never a typed time.

Keep one waiter per PR. Stop the previous one before arming another, and say you are waiting only after arming it. Use no `ScheduleWakeup` or `CronCreate` for this loop.

- **0** — run the next round.
- **10 or 11** — stop. The PR merged or closed.
- **13** — stop. The PR went quiet.
- **2** — stop and report the error.

If the runtime cannot resume the session when a background command exits, as in the Codex app, use [babysit-pr's Polling fallback](../babysit-pr/SKILL.md#polling-fallback) for the polling cadence and quiet cap. Check for changes at each poll and run a review round only when the waiter would owe one.

## Report

Stay silent on rounds that post nothing. Give a short note when a round posts. At the end, report why the loop stopped, the last verdict and the head it covers, and which threads are still open.
