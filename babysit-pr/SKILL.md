---
name: babysit-pr
description: Use when asked to babysit, watch, or keep an eye on a pull request or merge request until it's approved — repeatedly runs the address-pr flow (conflicts, review comments, CI) round after round, waiting between rounds on a schedule that backs off as things stay quiet, and stops once the PR is done — an external AI reviewer has approved the current head (a person's approval doesn't count), every discussion is resolved, the pipeline is green and the base branch is synced — or it's merged/closed. Works on the current branch's PR or one given by number/URL, on GitHub or GitLab. The looping counterpart to address-pr, which runs a single pass.
---

# Babysit a Pull Request

Watch one pull request (GitHub) or merge request (GitLab) — the one for the current branch, or the one given by number/URL — and keep it moving by running the address-pr flow round after round, until it is done — approved by an external AI reviewer on the current head, every discussion resolved, pipeline green, base synced — or it's merged/closed, polling every few minutes at first and backing off as it stays quiet.

## The loop

Resolve the target PR once, then keep watching that same one. Each round:

1. **Check the stop conditions first** (see below). If the PR is done, or merged/closed, stop and give the final report — don't keep working a finished PR. An approval alone does not finish it: if a condition is still unmet, this round works that condition.
2. **Otherwise run one address-pr pass** by invoking the address-pr skill: it resolves conflicts with the base, addresses the open/unresolved review comments, handles failing CI, and pushes once — but only if something actually changed. Just invoke it every round and let it no-op when nothing has changed; don't try to pre-detect new activity yourself. A round with nothing new is a no-op, and that's expected; most rounds while you wait on a reviewer will be quiet.
3. **Wait** — how long is under *Waiting between rounds* below, and it grows as the PR stays quiet — then repeat from step 1. The wait is *between* rounds, so run the first round right away rather than waiting first.

## When to stop

Stop the loop and report when the PR is **done** — which takes all four of these together, not any one of them:

1. **Approved by the external AI reviewer** — see *What counts as an approval* below.
2. **Every discussion resolved.**
3. **The pipeline green** on the current head.
4. **Synced with the base branch** — no commits behind it, no conflicts.

An approval on its own is not done. A PR can carry a current approval and still be unmergeable: a red pipeline, one unresolved thread, or a base that moved underneath it each leave it stuck, and none of them resolve themselves. Keep working the remaining conditions — that is what the address-pr pass is for — and stop only once all four hold at the same time, on the same head. Anything that moves the head resets the ones that depend on it.

Stop immediately, without waiting for the four, when any of these is true instead:

- **Merged or closed.**
- **Gone quiet** — about 4 hours have passed with no update to the PR: a run of quiet rounds (roughly a dozen, as the wait backs off) where nothing changed — no new commits, comments, reviews, or CI results, and nothing for you to do. Stop, say so, and let the user re-run to keep watching. Any real update resets this clock, so an actively moving PR is never abandoned. The clock also never predates this watch, so it can't already be spent when you arrive: a PR nobody has touched in days still gets its first round and then its full four hours (see *Waiting between rounds*). This stop is for a PR that goes quiet **while you watch it**, never a reason to decline to start.
- **Hard error** — the PR or branch is gone, auth fails, or a push is rejected in a way a retry won't fix. Stop and report rather than spinning on it.

### What counts as an approval

Condition 1 is met when an external AI reviewer has signed off on the current head. It's an *approval* when that reviewer's latest review is a formal approval (GitHub: its newest `reviews` entry is `APPROVED`; GitLab: an active MR approval) **or** a review/comment genuinely signals go (LGTM, "approved", "ship it", 👍) — judge intent, not keywords: "not approving until you fix X" is not approval, and neither is a comment that merely describes or acknowledges the change, the loop's own address-pr replies included.

**The watch never supplies that approval itself.** It waits for whatever AI reviewer covers this repo's PRs; reviewing the PR yourself and then stopping on your own verdict is the watch approving its own work. If nothing reviews this PR, condition 1 simply never comes true and the watch ends at the quiet cap — report that plainly instead of signing off in the reviewer's place.

A person's approval is not it either. A teammate clicking Approve, or a human "LGTM", means a person is happy — it says nothing about whether the AI reviewer has signed off. Keep working, and name it in the final report so the user knows the PR is human-approved.

An approval speaks for the head it named. Once a push or a rebase moves the head, it no longer describes what is on the PR, so condition 1 is unmet again until the reviewer approves the new head — even when the rebase changed nothing but the sha.

Transient trouble — rate limits, a flaky network, a single failed API call — is *not* a stop: treat that round as a no-op and try again next round.

## Waiting between rounds

You run as one continuous, self-paced task — not a fresh invocation per round — so between rounds wait *without blocking*: hand control back and schedule yourself to resume, rather than a foreground `sleep` (which is typically blocked). In Claude Code that's `ScheduleWakeup`, the mechanism `/loop`'s dynamic mode uses; end the loop with its `stop` once a stop condition is met.

Let the wait grow while the PR sits idle. How long to wait depends on one thing only — how many quiet rounds you have behind you right now, a quiet round being one where nothing had changed and there was nothing to do:

| Quiet rounds in a row so far | Wait before the next round |
|---|---|
| 0 (the round you just ran did something) or 1 | 5 min — `delaySeconds: 300` |
| 2 or 3 | 10 min — `600` |
| 4 or 5 | 20 min — `1200` |
| 6 or more | 30 min — `1800`, the cap |

Read that table after every round and take the wait from it; don't track a wait that you double by hand. Any round that finds an update puts the count back to 0, so a PR that just came alive returns to a 5-minute pulse immediately, while one nobody has touched costs two wake-ups an hour instead of twelve. Never shorten a wait to keep your context warm — that buys nothing and burns a round.

Measured from the last update, that puts rounds at 5, 10, 20, 30, 50 and 70 minutes, then every 30 after — about a dozen rounds to reach the 4-hour stop.

Keep the wake-up's `reason` short — on a silent watch it's the one line the user sees each round. Each wake-up re-enters this skill, so it has to find your state somewhere: the target PR, when the PR itself last moved, how many quiet rounds have run in a row (back to 0 on any round that finds one — that count is what the table above reads), and the time the next round is due.

**Keep that state in a file the next round reads, not in the wake-up message.** Passing it through the message seems natural and quietly wrecks the cadence: state changes whenever something lands, so you re-schedule to carry the new state forward, and because scheduling a resume replaces the pending one, every such carry-forward resets the interval from that moment. The wait you configured is then never the wait you get, and it stretches exactly when the watch is busiest — with nothing in the output to show for it, since no round was skipped, only postponed. A file separates the two concerns: **the schedule changes only when you deliberately change the schedule.** Keep the wake-up message thin and let it point at the file. Stop once about **4 hours** have passed with no movement, measured from that mark — which comes off the PR rather than out of your own bookkeeping, and never sits earlier than this watch's own start.

Take that mark from the PR, not from your own bookkeeping. Read the actual time of the newest activity on it — commit, comment, review, CI result — rather than stamping "now" whenever a round happens to notice something. The two come apart in both directions: stamping now on a three-hour-old comment you only just read buys the watch four more hours it has not earned, and a round that misses an update leaves the mark stuck in the past, so the quiet cap can fire on a PR that is in fact moving. Reading the PR's own latest-activity time each round is self-correcting — a missed round costs nothing, because the next one sees the same activity and gets the same answer.

That mark has one floor: **it never predates this watch.** Take it as the later of the PR's newest activity and the moment you started watching, so a fresh watch always runs its first round and then measures the cap from when it actually began. Without the floor, a PR last touched three days ago hands you a cap that expired long before the user asked for anything, and the watch stops on round one having done nothing — asking for a babysitter is asking you to *start* watching, and how stale the PR was on arrival says nothing about whether it's worth watching now. Note this is not the "stamping now" mistake above: that one lets any round restamp the mark on activity it merely noticed, over and over, buying unearned time every time. The floor applies once, at the start, and never again — from the first round on, the mark moves only on genuine new activity.

If your runtime has no way to resume without blocking, don't busy-wait or fake the delay — tell the user this skill needs a scheduled-resume capability (e.g. running it on a recurring interval) and stop.

## Detecting the stop signal

Conditions 2-4 come straight off the host: unresolved threads from the PR's discussions, the pipeline status for the current head (not for an older one — a stale green says nothing), and the base comparison for commits-behind and mergeability. Read all four each round rather than assuming an earlier round's answer still holds.

### Reading it off the PR

- **GitHub:** `gh pr view <pr> --json state,mergedAt,reviews,comments` — stop on a `state` of `MERGED`/`CLOSED`, or count condition 1 met on a reviewer whose *latest* `reviews` entry is `APPROVED`, or on an approval-style go-signal (a "LGTM"/`APPROVED` left as a plain comment or review, not just the Approve button) landing either as a top-level `comments` entry or in a `reviews` body. GitHub keeps stale `APPROVED` reviews in the list after later changes, so go by each reviewer's newest review, not any historical one.
- **GitLab:** `glab mr view <mr>` shows the MR's state, its approvals (who has approved), and its notes — stop on a merge/close, or count condition 1 met on an approval-style note. An AI reviewer's verdict usually arrives as a note rather than a formal approval, so read the notes and not just the approvals list.

An AI reviewer names itself in its verdict, which is what separates it from a teammate's Approve. The posting account does not settle it: a reviewer posts under a dedicated bot account when one is configured for the repo and under the PR author's own account when none is.

## Report

Stay silent on quiet rounds. Most rounds while you wait on a reviewer find nothing new — address-pr makes no change and there's nothing to say, so say nothing, and don't surface address-pr's own per-round report on those rounds either. Speak up only on a round that actually did something — a brief note of what it changed or pushed. When the loop ends, give one summary: why it stopped (all four conditions met — say who approved and on which head — or merged, closed, cap reached, or the error), and where any condition still stands unmet if you stopped for one of the other reasons, everything addressed across all the rounds (conflicts resolved, comment fixes, CI handled), and anything still uncertain or unresolved — the same honesty address-pr reports with on a single pass. If the PR picked up a person's approval, say so there too, so the user knows it's sitting on the PR and can decide what it's worth.
