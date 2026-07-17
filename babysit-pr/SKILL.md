---
name: babysit-pr
description: Use when asked to babysit, watch, or keep an eye on a pull request or merge request until it's approved — repeatedly runs the address-pr flow (conflicts, review comments, CI) round after round, waiting between rounds on a schedule that backs off as things stay quiet, and stops once the PR gets a formal approval or an approval comment (or is merged/closed). Works on the current branch's PR or one given by number/URL, on GitHub or GitLab. The looping counterpart to address-pr, which runs a single pass.
---

# Babysit a Pull Request

Watch one pull request (GitHub) or merge request (GitLab) — the one for the current branch, or the one given by number/URL — and keep it moving by running the address-pr flow round after round, until it's approved (or merged/closed), polling every few minutes at first and backing off as it stays quiet.

## The loop

Resolve the target PR once, then keep watching that same one. Each round:

1. **Check for a stop signal first** (see below). If one is met, stop and give the final report — don't keep working an already-approved or closed PR.
2. **Otherwise run one address-pr pass** by invoking the address-pr skill: it resolves conflicts with the base, addresses the open/unresolved review comments, handles failing CI, and pushes once — but only if something actually changed. Just invoke it every round and let it no-op when nothing has changed; don't try to pre-detect new activity yourself. A round with nothing new is a no-op, and that's expected; most rounds while you wait on a reviewer will be quiet.
3. **Wait** — how long is under *Waiting between rounds* below, and it grows as the PR stays quiet — then repeat from step 1. The wait is *between* rounds, so run the first round right away rather than waiting first.

## When to stop

Stop the loop and report as soon as any of these is true:

- **Approved** — a reviewer's *latest* review is a formal approval (GitHub: that user's newest `reviews` entry is `APPROVED`; GitLab: an active MR approval) **or** a reviewer left a comment that genuinely signals go (LGTM, "approved", "ship it", 👍). Judge intent, not keywords — "not approving until you fix X" is not approval. Ignore the PR author's own reviews and comments.
- **Merged or closed.**
- **Gone quiet** — about 4 hours have passed with no update to the PR: a run of quiet rounds (roughly ten, as the wait backs off) where nothing changed — no new commits, comments, reviews, or CI results, and nothing for you to do. Stop, say so, and let the user re-run to keep watching. Any real update resets this clock, so an actively moving PR is never abandoned.
- **Hard error** — the PR or branch is gone, auth fails, or a push is rejected in a way a retry won't fix. Stop and report rather than spinning on it.

Transient trouble — rate limits, a flaky network, a single failed API call — is *not* a stop: treat that round as a no-op and try again next round.

## Waiting between rounds

You run as one continuous, self-paced task — not a fresh invocation per round — so between rounds wait *without blocking*: hand control back and schedule yourself to resume, rather than a foreground `sleep` (which is typically blocked). In Claude Code that's `ScheduleWakeup`, the mechanism `/loop`'s dynamic mode uses; end the loop with its `stop` once a stop condition is met.

Let the wait grow while the PR sits idle. How long to wait depends on one thing only — how many quiet rounds you have behind you right now, a quiet round being one where nothing had changed and there was nothing to do:

| Quiet rounds in a row so far | Wait before the next round |
|---|---|
| 0 — the round you just ran did something | 5 min — `delaySeconds: 300` |
| 1 | 10 min — `600` |
| 2 | 20 min — `1200` |
| 3 or more | 30 min — `1800`, the cap |

Read that table after every round and take the wait from it; don't track a wait that you double by hand. Any round that finds an update puts the count back to 0, so a PR that just came alive returns to a 5-minute pulse immediately, while one nobody has touched costs two wake-ups an hour instead of twelve. Never shorten a wait to keep your context warm — that buys nothing and burns a round.

Measured from the last update, that puts rounds at 5, 15, 35 and 65 minutes, then every 30 after — about ten rounds to reach the 4-hour stop.

Keep the wake-up's `reason` short — on a silent watch it's the one line the user sees each round. Each wake-up re-enters this skill, so carry your state along in it: the target PR, the timestamp of the most recent update you've seen (refreshed to now on any round that finds one), and how many quiet rounds have run in a row (back to 0 on any round that finds one — that count is what the table above reads). That timestamp is how the next round knows which PR to look at and whether the watch has gone quiet — stop once about **4 hours** have passed with no update, measured from that timestamp rather than from when the watch started. If your runtime has no way to resume without blocking, don't busy-wait or fake the delay — tell the user this skill needs a scheduled-resume capability (e.g. running it on a recurring interval) and stop.

## Detecting the stop signal

- **GitHub:** `gh pr view <pr> --json state,mergedAt,author,reviews,comments` — stop on a `state` of `MERGED`/`CLOSED`, a reviewer whose *latest* `reviews` entry is `APPROVED`, or an approval-style go-signal from a non-`author`, whether that lands as a top-level `comments` entry or in the body of a `reviews` entry (a "LGTM" left as a plain comment or review, not just the Approve button). GitHub keeps stale `APPROVED` reviews in the list after later changes, so go by each reviewer's newest review, not any historical one.
- **GitLab:** `glab mr view <mr>` shows the MR's state, its approvals (who has approved), and its notes — stop on a merge/close, an approval, or an approval-style note from someone other than the author.

## Report

Stay silent on quiet rounds. Most rounds while you wait on a reviewer find nothing new — address-pr makes no change and there's nothing to say, so say nothing, and don't surface address-pr's own per-round report on those rounds either. Speak up only on a round that actually did something — a brief note of what it changed or pushed. When the loop ends, give one summary: why it stopped (approved by whom, merged, closed, cap reached, or the error), everything addressed across all the rounds (conflicts resolved, comment fixes, CI handled), and anything still uncertain or unresolved — the same honesty address-pr reports with on a single pass.
