---
name: babysit-pr
description: "Watch a GitHub PR or GitLab MR, address new feedback and CI failures, and stop when its external AI review and merge checks pass."
---

# Babysit a Pull Request

Watch one pull request (GitHub) or merge request (GitLab) — the one for the current branch, or the one given by number/URL — and keep it moving by running the address-pr flow round after round, until it is done — an external AI reviewer's approval covering the current head, every discussion resolved, pipeline green, mergeable with the base — or it's merged/closed. Prefer native PR events when `WatchPullRequest` is available; in a desktop-app session where Auto-fix already fixes the PR, wait on the background waiter; otherwise poll every few minutes at first and back off as it stays quiet.

## The loop

**Delivery mode.** If the user invokes [release](../release/SKILL.md), reuse this watch and its waiting transport. Store `deliveryMode: merge-and-deploy` and the scoped authorization in the existing state file; read the mode at the start of every round and again before stopping on readiness. When enabled, the four ready conditions hand off to release's merge/deploy phases instead of ending the watch. Resume an in-progress delivery phase through release without running another address-pr pass. An ordinary babysit invocation remains a readiness-only watch; it does not authorize merge or deploy. A merged target in delivery mode also resumes through release so a restart or an external merge does not discard the pending deployment; a closed, unmerged target still stops.

Resolve the target PR and its canonical URL once, then keep watching that same one. Discover `WatchPullRequest` by name and description, including namespaced variants exposed in restored chats, before selecting the waiting transport below, and check whether Auto-fix is on for this PR (the desktop app's PR status reports it). Keep the target, transport and delivery mode in the watch's state file so later events and restored chats reuse them. Each round:

1. **Check the stop conditions first** (see below). For a readiness-only watch, if the PR is done, or merged/closed, stop and give the final report — don't keep working a finished PR. In delivery mode, hand a ready or merged target to release instead; a closed, unmerged target still stops. An approval alone does not finish it: if a condition is still unmet, this round works that condition.
2. **Otherwise run one address-pr pass** by invoking the address-pr skill: it resolves conflicts with the base, addresses the open/unresolved review comments, handles failing CI, and pushes once — but only if something actually changed. Just invoke it every round and let it no-op when nothing has changed; don't try to pre-detect new activity yourself. A round with nothing new is a no-op, and that's expected; most rounds while you wait on a reviewer will be quiet.

   **Merge the base only when you have to.** In a babysit round the pass merges `origin/<base>` only in three cases: the PR conflicts with the base, the host refuses to merge a branch that is behind (see *Detecting the stop signal*), or the round pushes other changes anyway. A base that merely moved ahead is not a reason to merge or push. A sync-only push reruns the whole pipeline on an unchanged branch and cancels the run in progress, and the base branch's own pipeline tests the merged result after the merge anyway. Unstacking a PR whose PR underneath has merged is not a sync; it still happens as address-pr describes.
3. **Wait** — with the native watch, yield until a PR event; with the background waiter, arm it and yield; with the polling fallback, schedule the next round using *Waiting between rounds* below. Each native PR event re-enters this skill at step 1: fetch current host state, check the stop conditions, otherwise run one address-pr pass. The event's “ready” label does not replace the four conditions or authorize merging. Run the first round immediately after arming the watch, without waiting for an event or timer.

## When to stop

For a readiness-only watch, stop the loop and report when the PR is **done**. In delivery mode, these conditions trigger release's final recheck and delivery instead. Readiness takes all four together, not any one of them:

1. **Approved by the external AI reviewer** — see *What counts as an approval* below.
2. **Every discussion resolved.**
3. **The pipeline green** on the current head, or no CI to wait for under the rule in release's step 1: the repository has no CI at all, or no workflow ran for the head 10 minutes after the push and its changed files match none of the workflows' path filters.
4. **Mergeable with the base** — no conflicts, and not held back for being behind where the host requires an up-to-date branch. Otherwise, being behind doesn't count against this; see step 2 of the loop.

An approval on its own is not done. A PR can carry a current approval and still be unmergeable: a red pipeline, one unresolved thread, or a base that blocks the merge each leave it stuck, and none of them resolve themselves. Keep working the remaining conditions — that is what the address-pr pass is for — and stop only once all four hold at the same time, on the same head. Anything that moves the head resets the ones that depend on it.

Stop immediately, without waiting for the four, when any of these is true instead:

- **Merged or closed.** In delivery mode, a merged target resumes through release; a closed, unmerged target stops.
- **Gone quiet in the polling fallback** — about 4 hours have passed with no update to the PR: a run of quiet rounds (roughly a dozen, as the wait backs off) where nothing changed — no new commits, comments, reviews, or CI results, and nothing for you to do. Stop, say so, and let the user re-run to keep watching. Any real update resets this clock, so an actively moving PR is never abandoned. The clock also never predates this watch, so it can't already be spent when you arrive: a PR nobody has touched in days still gets its first round and then its full four hours (see *Waiting between rounds*). This stop is for a PR that goes quiet **while you watch it**, never a reason to decline to start. Native watches wait without polling turns and use the host's delivery cap instead.
- **Hard error** — the PR or branch is gone, auth fails, or a push is rejected in a way a retry won't fix. Stop and report rather than spinning on it.

### What counts as an approval

Condition 1 is met when an external AI reviewer has signed off on the current head, or on an earlier head it differs from only by a base-sync (see below). It's an *approval* when that reviewer's latest review is a formal approval (GitHub: its newest `reviews` entry is `APPROVED`; GitLab: an active MR approval) **or** a review/comment genuinely signals go (LGTM, "approved", "ship it", 👍) — judge intent, not keywords: "not approving until you fix X" is not approval, and neither is a comment that merely describes or acknowledges the change, the loop's own address-pr replies included.

**The watch never supplies that approval itself.** It waits for whatever AI reviewer covers this repo's PRs; reviewing the PR yourself and then stopping on your own verdict is the watch approving its own work. If nothing reviews this PR, condition 1 simply never comes true and the watch ends at the quiet cap — report that plainly instead of signing off in the reviewer's place.

A person's approval is not it either. A teammate clicking Approve, or a human "LGTM", means a person is happy — it says nothing about whether the AI reviewer has signed off. Keep working, and name it in the final report so the user knows the PR is human-approved.

An approval speaks for the head it named, and for every later head that differs from it only by a base-sync: a merge of the base in, or a rebase onto it, that left the PR's authored lines as they were. The reviewer posts nothing on such a head, because review-pr counts its approval as still standing there, so waiting for a fresh one only runs the watch into its quiet cap. When the head has moved since the approval, run the base-sync test in review-pr's *Re-reviewing a PR you have already reviewed* between the head the approval named (the full sha in the verdict, or the commit a GitHub review was submitted on) and the current one. Read the approved head's lines against the base the PR had when the approval was posted: a stacked PR retargeted after its base PR merged has a new base, and against it the old head also carries the merged base PR, so it never matches (when that base branch was deleted on merge, its PR's last head stands in for it). Identical authored lines keep condition 1 met, even when the sync resolved conflicts: a resolution that reverts either side changes those lines. Any other difference means the approval no longer describes what is on the PR, so condition 1 is unmet until the reviewer approves the new head.

Transient trouble — rate limits, a flaky network, a single failed API call — is *not* a stop: treat that round as a no-op and try again next round.

## Waiting between rounds

### Native PR events

When `WatchPullRequest` exists, call it with `{"url":"<canonical PR/MR URL>"}` once when starting the watch, before the immediate first round. Record `transport: native` only after success. The tool attaches the PR if necessary and enables its watch; attachment alone does not enable watching. On subsequent PR events, reuse the persisted watch rather than arming it again. Schedule no `ScheduleWakeup`, `CronCreate` or backup heartbeat for this transport. A tool refusal or failure is not tool absence: report the reason and stop rather than bypassing ownership or watch limits with polling.

Before the final report, or before handing a ready/merged target to release's delivery phases, call `WatchPullRequest` with the same URL and `stop: true`. Stop only this PR watch; leave unrelated wakeups and PR watches intact. If release returns to readiness waiting, arm a new native wait. When converting a restored polling round to native events, cancel its recorded backup heartbeat and do not schedule another polling wakeup; cancel a still-pending polling wakeup only through a mechanism that targets that owned job. Do not use session-wide stop to remove unrelated jobs.

The host app must be running and the machine awake. Native watches persist through restart and pause after 20 event deliveries; do not automatically re-arm a paused watch to reset that limit. Its pause and refresh-failure notices ask the user to resume or repair access. Do not add a timer to enforce the polling fallback's quiet cap.

### Background waiter (Auto-fix sessions)

Use this transport when `WatchPullRequest` is unavailable and Auto-fix is on for the PR in the Claude desktop app. Auto-fix wakes the session with a `<ci-monitor-event>` for a CI failure, a merge conflict or review comments, and each such event runs one address-pr pass as a round. Green CI and an approval produce no event, so nothing would resume the session once the last fix lands. The waiter covers that gap. Keep `transport: waiter` in the state file.

The waiter is `~/.claude/bin/wait-pr-ready <canonical PR URL> --head <head sha>`, installed by this repo's `bin/setup`. Run it as one background shell command (Claude Code's `run_in_background`), which re-invokes the session when it exits. It reads the PR with backoff, from 1 minute up to a 5-minute cap, and exits with a one-line reason:

| Exit | Reason | Next |
|---|---|---|
| 0 | the four conditions of *When to stop* hold on the head, the approval matched as *What counts as an approval* describes, base-sync included; the line ends with the PR's activity mark | run a round: it rechecks fresh host state and stops, or hands to release. If the recheck disagrees (a keyword the waiter read as a go-signal, a host merge block), re-arm with `--after <that mark>`, so the waiter reports ready again only after something on the PR moves |
| 10 / 11 | merged / closed | the merged-or-closed stop |
| 12 | the head moved | arm a waiter for the new head unless one already watches it |
| 13 | about 4 hours with no activity on the PR | the quiet stop |
| 14 | a review, comment or thread landed and the PR is still not ready (findings); an AI reviewer's approval covering the head, with no thread open, doesn't count while checks are still running or mergeability is still being computed, and the waiter keeps waiting | run a round: address-pr answers them. This wakes the session even when Auto-fix relays nothing, as it doesn't for an AI reviewer posting from a bot account it skips; whichever wakes first runs the round, and the other finds nothing left to do |
| 15 | the head conflicts with the base | run a round now: its address-pr pass merges the base and resolves the conflicts. Auto-fix may report the same conflict; whichever wakes the session first runs the round, and the other finds nothing left to do. When the round leaves that head still conflicting (address-pr stopped to ask rather than push a merge it couldn't resolve with confidence), record that head as `knownConflictHead` in the state file, re-arm with `--known-conflict`, which won't exit 15 for that head again, and put the conflict to the user |
| 2, or any other | bad usage, the PR could not be read six times in a row, or the script failed | the hard-error stop |

The waiter finds both no-CI cases release's step 1 describes on its own, each only once the head has had no checks for 10 minutes after the waiter first saw it, so a late external status lands first: a repository with no Actions workflow, and a head no workflow ran for. For the second its only evidence is the timeout, which a delayed Actions run also produces, so the round that takes its ready confirms the path filters before counting condition 3 as met. `--no-ci` skips the gate from the first read; pass it only for a project with no CI at all; pass `--reviewer <login>` when the AI reviewer posts from an account of its own (a bot account) or its verdicts carry no identity line. Given `--reviewer`, only the listed accounts' verdicts count toward the approval, so an identity-line approval from another account, such as a session reviewer posting from the PR author's account, neither stands in for the repo's reviewer nor wakes the wait as a finding; an open thread, a failed check or a blocked base still wakes it. Logins match ignoring case and a trailing `[bot]`.

It doesn't compete with Auto-fix: it never changes the PR, never fixes CI and never answers a comment. It only detects readiness, the one transition Auto-fix doesn't report, plus a conflict with the base and new findings, which nothing else reports where Auto-fix is off (a Codex PR, a PR whose `set_monitor` failed) or doesn't relay them. The two never act on the same thing, so its read-only polling is not the CI polling the app advises against.

End every round by making sure a waiter is armed for the current head: arm one, with `--head` set to that head (and `--known-conflict` when that head is the state file's `knownConflictHead`), unless the waiter task recorded in the state file is still running for it. Checking the task, not the state file's word, matters because the waiter is a child of the session: an app quit, a restart or a sleep kills it without an exit, unlike the native watch, and a round that only replies pushes nothing to prompt a re-arm. Before arming a new one, stop the previous waiter (`TaskStop` on its task id); an old one left running only exits 12 on the new head. Schedule no `ScheduleWakeup`, `CronCreate` or backup heartbeat for this transport. The waiter is GitHub-only: it exits 2 on a GitLab MR, and since the desktop app's Auto-fix never monitors a GitLab MR, a GitLab watch uses the polling fallback.

### Never claim a wait you haven't armed

Tell the user you are waiting for CI, a review or an approval only while a native watch, a background waiter or a polling wake-up is armed for this PR. If none is, say the watch has ended and what is still unmet, so they know to re-run it.

### Polling fallback

Use the following loop only when the native PR-watch tool is unavailable and no Auto-fix watch covers the PR. Keep `transport: polling` in the state file. Existing polling cadence, quiet cap and backup recovery remain unchanged.

You run as one continuous, self-paced task — not a fresh invocation per round — so between rounds wait *without blocking*: hand control back and schedule yourself to resume, rather than a foreground `sleep` (which is typically blocked). In Claude Code that's `ScheduleWakeup`, the mechanism `/loop`'s dynamic mode uses; end the loop with its `stop` once a stop condition is met.

For Codex app review watches, create heartbeat automations with `notificationPolicy=failed_runs_only` by default, unless the user asks for routine notifications. Keep that policy when updating or resuming an existing watch; review-open-prs inherits this default. It mutes routine app notifications while allowing failed-run alerts, and it does not pause the watch or change its schedule. Task messages are separate: change the heartbeat prompt's reporting rule only when the user asks for fewer messages in the task.

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

**A scheduled resume can be lost, and nothing will tell you.** The wake-up is the only thing that brings the watch back, so one that never fires ends the watch as surely as a stop — with no error, no final report, and a last turn that looks finished. So when the watch starts, arm a backup heartbeat next to the wake-up: a recurring, session-only job that re-enters this skill on a fixed clock — in Claude Code, `CronCreate` on off-minutes every 15 minutes (e.g. `7,22,37,52 * * * *`). Its prompt reads the state file first and does nothing unless the next round is overdue — the due time you recorded has passed by a few minutes; only then does it run the round exactly as the wake-up would, arming the next wake-up included. It never paces the watch — the table above still does — it only caps what a lost resume can cost. Record its id in the state file and delete it (`CronDelete`) in the same turn as any stop: `ScheduleWakeup`'s `stop` does not touch it, and an orphaned heartbeat keeps re-entering a finished watch until it expires.

If your runtime has no way to resume without blocking, don't busy-wait or fake the delay — tell the user this skill needs a scheduled-resume capability (e.g. running it on a recurring interval) and stop.

## Detecting the stop signal

Conditions 2-4 come straight off the host: unresolved threads from the PR's discussions, the pipeline status for the current head (not for an older one — a stale green says nothing), and whether the base blocks the merge. Read all four each round rather than assuming an earlier round's answer still holds.

For the base, GitHub reports `mergeable` and `mergeStateStatus` (`gh pr view <pr> --json mergeable,mergeStateStatus`). `CONFLICTING` is a conflict. `BEHIND` means branch protection requires the branch to be up to date. Right after the head or the base moves, `mergeable` reads `UNKNOWN` for a while. Then fetch and run `git merge-tree --write-tree origin/<base> HEAD`: it exits 1 on a conflict and touches neither the worktree nor the index. On GitLab, the MR's `detailed_merge_status` (`glab api projects/:id/merge_requests/<iid>`) reads `conflict` for a conflict, and `need_rebase` when the project's merge method requires an up-to-date branch.

### Reading it off the PR

- **GitHub:** `gh pr view <pr> --json state,mergedAt,reviews,comments` — stop on a `state` of `MERGED`/`CLOSED`, or count condition 1 met on a reviewer whose *latest* `reviews` entry is `APPROVED`, or on an approval-style go-signal (a "LGTM"/`APPROVED` left as a plain comment or review, not just the Approve button) landing either as a top-level `comments` entry or in a `reviews` body. GitHub keeps stale `APPROVED` reviews in the list after later changes, so go by each reviewer's newest review, not any historical one.
- **GitLab:** `glab mr view <mr>` shows the MR's state, its approvals (who has approved), and its notes — stop on a merge/close, or count condition 1 met on an approval-style note. An AI reviewer's verdict usually arrives as a note, with or without a formal approval, so read the notes and not just the approvals list.

An AI reviewer names itself in its verdict, which is what separates it from a teammate's Approve. The posting account does not settle it: a reviewer posts under a dedicated bot account when one is configured for the repo and under the PR author's own account when none is.

## Report

Stay silent on quiet rounds. Most rounds while you wait on a reviewer find nothing new — address-pr makes no change and there's nothing to say, so say nothing, and don't surface address-pr's own per-round report on those rounds either. Speak up only on a round that actually did something — a brief note of what it changed or pushed. When the loop ends, give one summary: why it stopped (all four conditions met — say who approved and on which head, and name the current head too when the approval carried across a base-sync — or merged, closed, cap reached, or the error), and where any condition still stands unmet if you stopped for one of the other reasons, everything addressed across all the rounds (conflicts resolved, comment fixes, CI handled), and anything still uncertain or unresolved — the same honesty address-pr reports with on a single pass. If the PR picked up a person's approval, say so there too, so the user knows it's sitting on the PR and can decide what it's worth.
