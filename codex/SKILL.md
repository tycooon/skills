---
name: codex
description: "Hand a task to a headless Codex session from Claude, review its PR and resume the session until it is ready."
---

# Run a Codex Session

The user asked for a Codex session or agent on a task. Codex writes the code and you review it: launch it headless with `codex exec`, review what it delivers, and resume the same session whenever something is left to change, until its PR is ready. Don't make those changes yourself. A running Codex session counts as a subagent toward any limit your instructions put on running them in parallel.

Every run sets its model and reasoning effort explicitly, so the user's own Codex config never decides them. Unless the user names others, use `medium` effort and the newest listed Sol model, read once per conversation from the model list Codex keeps locally (no network call):

```bash
python3 -c 'import json,os,re;m=json.load(open(os.path.expanduser("~/.codex/models_cache.json")))["models"];v=[(tuple(map(int,x.group(1).split("."))),s) for s in (e["slug"] for e in m if e.get("visibility")=="list") if (x:=re.fullmatch(r"gpt-([\d.]+)-sol",s))];print(max(v)[1])'
```

It prints a slug such as `gpt-6.1-sol`. If the file is missing or the command fails, say so and ask the user which model to use rather than guessing one.

Every run also has a speed, SPEED. Use the standard speed unless the user asks for another: leave `-c service_tier=SPEED` out of the commands below. A faster speed is the id of one of the model's service tiers, such as `priority`, the tier Codex calls Fast, which the model list rates at 1.5x to 2x the standard speed for more usage. Codex has no `--fast` flag; a tier goes in as `-c service_tier=SPEED`. The same model list says which tiers the model offers and what each claims:

```bash
python3 -c 'import json,os,sys;m={e["slug"]:e for e in json.load(open(os.path.expanduser("~/.codex/models_cache.json")))["models"]};[print(t["id"],"-",t["name"],"-",t.get("description","")) for t in m[sys.argv[1]].get("service_tiers",[])]' MODEL
```

On 2026-10-10 `gpt-6.1-sol` listed `priority` ("Fast", "2x speed, increased usage") and `ultrafast`; `gpt-6-sol` and `gpt-5.6-sol` listed `priority` at 1.5x. Measured end to end on `gpt-6.1-sol`, eight alternating runs of one 600-word answer took 50.4 s (median) at `priority` against 67.7 s standard, 1.34x: Codex's startup is not sped up, and neither is time a session spends waiting on CI. When the user asks for a tier the model does not list, say so and use the standard speed. A tier the model does not offer is not an error: Codex prints `Configured service tier … is not advertised as supported … and will be omitted from requests` in the log and runs at the standard speed, so read the log's opening lines once after a launch with a tier.

Reuse the same MODEL, EFFORT and SPEED for every run and resume of a session, and name them when you report its launch. A session that is already running keeps its speed: to change it, stop the run and resume the session with the new tier, telling it to pick up where it stopped.

For an opinion only, with no code change and no follow-up, skip the steps and run `codex exec -m MODEL -c model_reasoning_effort=EFFORT -c service_tier=SPEED --ephemeral -s read-only -C DIR -o ANSWER - < PROMPT > LOG 2>&1`.

## On a remote host

When the user asks for the session on another machine, or your machine rules send the repository's coding work to a dev host, Codex runs there over SSH and you stay where you are. HOST is that machine's SSH name. What touches the worktree or Codex runs on HOST; the review and everything else on GitHub stays with you, since it needs only `gh`.

- **Paths are HOST's.** Its own rules (its `~/.claude/CLAUDE.md` and what that loads) say where its main checkouts and worktrees live, and which command makes a worktree. Read them once per conversation: `ssh HOST cat '~/.claude/CLAUDE.md'`, then the file it names for the machine.
- **Every command in this skill that reads `~/.codex`, touches the worktree or starts Codex** runs on HOST, with MODEL, EFFORT, SPEED and the paths filled in before it is sent. A command with single quotes of its own, such as the `models_cache.json` lookups above, can't sit inside `ssh HOST '…'`: send its script on stdin instead, `ssh HOST python3 - MODEL <<'PY'` with the script's body and a closing `PY` line. Mockups go across with `scp FILE HOST:WORKTREE/.plans/mockups/`.
- **Launch and resume.** The brief and follow-ups stay local and go in on stdin, so `< BRIEF` stays outside the quotes. The run's output stays on HOST, in the worktree's ignored `.plans/`, so a dropped connection can't cut it off, and the shell records its pid first:

  ```bash
  ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=6 HOST 'cd WORKTREE && mkdir -p .plans && echo $$ > .plans/NAME.pid && exec codex exec -m MODEL -c model_reasoning_effort=EFFORT -c service_tier=SPEED -C WORKTREE --approve-for-me --color never -o .plans/NAME-last.md - > .plans/NAME.log 2>&1' < BRIEF
  ```

  A resume is the same command with step 4's `resume SESSION_ID`, its follow-up on stdin, and the next numbered `-o` file, log and pidfile (`NAME-2.pid`, …), so a watch can never read the previous run's pid. Read both files with `ssh HOST cat` (or `tail` for the log) wherever this skill reads the `-o` file or the log.
- **A dropped connection or the timeout doesn't stop the run.** It ends only the local `ssh`, and Codex carries on in the worktree. Before reading the run as cut off, and before every resume, check it with the latest run's pidfile: `ssh HOST 'kill -0 $(cat WORKTREE/.plans/NAME.pid) 2>/dev/null && echo running'`. While it runs, it is still working: wait for it rather than starting a second run on the same session and worktree. Stop it with `kill` only when the user asks to stop.
- **Watch the process, not the connection.** The local `ssh` ending is not the run ending, so on a remote host it is not what re-invokes you. Right after every launch and resume, arm one background command on that run's own pidfile, with `timeout: 7200000`. It exits on its own only when that process has exited; failed reads are retried, and a pidfile the launch has not written yet counts as not started, not as exited:

  ```bash
  n=0; while :; do r=$(ssh -o ConnectTimeout=15 -o BatchMode=yes HOST 'P=WORKTREE/.plans/NAME.pid; if [ ! -s $P ]; then echo missing; elif kill -0 $(cat $P) 2>/dev/null; then echo running; else echo exited; fi' 2>/dev/null); case "$r" in exited) echo RUN_ENDED; break;; missing) n=$((n+1)); [ $n -ge 10 ] && { echo PIDFILE_MISSING; break; };; esac; sleep 60; done
  ```

  `RUN_ENDED` is the run's end: read the `-o` file then. When the `ssh` command exits first, the watch is still armed, so nothing more is needed. `PIDFILE_MISSING` means the launch never started its shell in ten minutes: read the local `ssh` output and the log. A watch that ends without either line was stopped by the Bash timeout, not by the run: check the pid and arm it again while the run is alive. Never tell the user a run is still working from a check you made earlier: check the pid again first, or say when you last checked. Once it has exited, its `-o` file holds the final message as usual. Keep the connection up with the `ServerAlive` options above all the same, so the local side sees the end of a run.
- **Pass the rules on.** Tell Codex in the brief that it runs on HOST, so it follows HOST's rules on worktrees and builds rather than yours.

## 1. Write the brief

Save it as `<working-dir>/.plans/YYYY-MM-DD-<topic>-codex-brief.md`, kept out of git as [design-brainstorming](../design-brainstorming/SKILL.md) describes, so the user can open it. Codex follows the same global rules and has the same skills, so the brief carries only the task: the problem with its evidence, what done looks like, what is out of scope. It also tells Codex to:

- work in the worktree it starts in and on the branch it finds there, making no other worktree: step 2 cuts both from the latest `origin/<default>` for it, or from the PR it stacks on when the brief says so;
- stop once its non-draft PR is open, with no babysit-pr and no waiting for a review, because you are the reviewer;
- not merge or deploy;
- end its final message with a report block: PR URL, one-line summary, each command it ran with its result, anything the reviewer or the user must know or answer.

**Mockups.** When the task changes a user interface, Codex gets the mockups as files. It can't open an artifact, a canvas or anything else behind your sign-in, and a layout described only in words loses its structure. If no mockup exists yet and your instructions call for one, make it before the brief.

Once step 2 has made the worktree, copy each mockup's source into `<worktree>/.plans/mockups/`: its HTML, or an exported image when there is no source. Check that the path is ignored there (`git check-ignore`); where it isn't, add it to the worktree's `.git/info/exclude`.

The brief then:
- names each file and the part of the task it shows;
- tells Codex to match their structure, wording and hierarchy while using the project's own styles and components;
- says the brief's text wins where it and a mockup differ;
- says the mockups are never committed.

A follow-up that adds UI work does the same before the resume.

## 2. Launch

**Check for overlapping work first.** List the open PRs that change the files or module this task will touch (GitHub: `gh pr list -L 200 --json number,title,headRefName,changedFiles,files`; `files` stops at 100 per PR, so for a PR with more `changedFiles` read `gh pr diff <n> --name-only`). Parallel PRs on the same files cost a Codex round per merge, since each one that lands leaves the other with a conflict to resolve and its checks to rerun. When one is in flight, pick one of these and say which when you report the launch: wait for it to merge, stack on it, or go ahead in parallel because the overlap is small. To stack, cut the worktree below from `origin/<that PR's branch>` instead of `origin/<default>`, open the PR against that branch, and say both in the brief (address-pr unstacks it once the PR underneath merges); where the repository's own worktree command takes no base, don't stack. Ask the user only when the overlapping PR is not part of their current work.

Make the session's worktree yourself, where your instructions put worktrees, on a new branch cut from the latest `origin/<default>`. Name the worktree `codex-<topic>` after the brief, and the branch the way your instructions name branches, `codex/<topic>` when they don't say.

```bash
git -C MAIN_CHECKOUT fetch origin
git -C MAIN_CHECKOUT worktree add --no-track -b BRANCH WORKTREE origin/DEFAULT
```

A repository's own command for making a worktree comes first: when its instructions give one, run it with the branch name in place of these two commands, and use the path it prints as the worktree.

Then start Codex in it, in a background Bash with `timeout: 7200000`. Two hours is the maximum, and the default stops it after 30 minutes.

```bash
codex exec -m MODEL -c model_reasoning_effort=EFFORT -c service_tier=SPEED -C WORKTREE --approve-for-me --color never -o SCRATCHPAD/NAME-last.md - < BRIEF > SCRATCHPAD/NAME.log 2>&1
```

- Never pass `--worktree`. Codex would make a worktree of its own at `<its worktree root>/<4 hex digits>/<repo>`, detached at the main checkout's HEAD: outside the place your instructions and the project's tooling expect a worktree, and on a base that can be behind.
- `--approve-for-me` keeps the `workspace-write` sandbox and sends approval requests to Codex's automatic reviewer, since nobody is there to answer a prompt. Never pass `--dangerously-bypass-approvals-and-sandbox`.
- Never pass `--ephemeral`: it keeps no session to resume.

The log opens with `stale rollout path` errors, which are noise, and within seconds a header with the `session id:` line a resume needs. Read it and tell the user the session id, the worktree and the brief.

**When auto mode refuses.** The first launch in a conversation is usually refused as `[Create Unsafe Agents]` when the user's request said no more than "run codex". Whenever a launch or a resume is refused, don't try variants: show the user the exact command, say what `--approve-for-me` does, and re-issue it unchanged on their go. If it is refused again, the command is theirs to run.

## 3. Review

Don't wait for a resumed run to exit before reviewing. Codex spends most of a run waiting on CI after its push, and a review that starts at the push lands while it still waits, where step 4's follow-up has it answer new threads in the same run. So alongside each resume, arm a background poll that exits when the PR's head moves:

```bash
OLD=HEAD_SHA; while :; do H=$(gh pr view PR_URL --json headRefOid --jq .headRefOid 2>/dev/null); [ -n "$H" ] && [ "$H" != "$OLD" ] && { echo "$H"; break; }; sleep 60; done
```

A failed read prints nothing and the poll reads again. When it prints a new head, review that head as below (items 3 to 5) while the run goes on, note it as the last head you reviewed, and arm the poll again with it. Stop the poll when the run exits. Use this poll rather than `wait-pr-ready`, whose readiness mode also exits on a conflict, on review activity (Codex's own replies included) and on readiness, all with the old head. The first launch gets no poll: its brief has Codex stop once its PR is open, so the run exits about when the PR appears.

You are re-invoked when the run exits as well (on a remote host, by the process watch, not by the `ssh` command). Run item 3 then only when the head differs from the last one you reviewed mid-run; otherwise go from item 2 straight to item 4.

1. Read the `-o` file. A run that was cut off leaves none, so read the log's tail instead.
2. Bind the PR it opened to your session, as the global rules require: the user follows its checks and review state there. Bind it and nothing more: no watch goes on with it, as the next step says.
3. Review the PR with [review-pr](../review-pr/SKILL.md), against the brief as well as the diff. You are this PR's external AI reviewer, so keep [babysit-pr](../babysit-pr/SKILL.md) and any Auto-fix watch off for it: both would have you fix what you review. Where the repo has an AI reviewer of its own (step 5) that posts from the account you post as, use the distinct identity `AI reviewer: Claude (Codex handoff)` on everything you post as the handoff reviewer: findings, review bodies, thread replies and verdicts, on the first pass and every re-review. Give your clean verdict as a plain review comment (`gh pr review --comment`), never a formal approval, naming the full reviewed head sha, for example `APPROVED. AI reviewer: Claude (Codex handoff). Reviewed <full head sha>.` This role identifies your review history to step 5 while excluding it from the repo reviewer's own reviews and threads under review-pr's *Whose comment it is*. The repo's reviewer must still review independently: otherwise a review from the shared account would stand in for the independent review step 5 waits for, and every readiness check would pass on yours alone.
4. Read the PR's checks and mergeability yourself afterwards: review-pr leaves CI out.
5. Read every unresolved thread on the PR, whoever opened it. The repo may have an AI reviewer of its own (a review bot that covers every PR), and it reviews on its own schedule, often after you: its threads are Codex's to address exactly as yours are, and the PR can't merge while they are open.
6. Pass the user any question Codex left that the brief doesn't answer.

## 4. Resume

Resume the session when anything is left for it to do: an open review thread, yours or another reviewer's, a failed or still-running check, a conflict with the base, or a run that stopped short of its PR. Use the same background Bash and timeout.

```bash
codex exec -m MODEL -c model_reasoning_effort=EFFORT -c service_tier=SPEED -C WORKTREE --approve-for-me --color never -o SCRATCHPAD/NAME-last-2.md resume SESSION_ID - < FOLLOW_UP > SCRATCHPAD/NAME-2.log 2>&1
```

- Always pass `-C` with the session's worktree. A resumed run works in the directory it is given (the current one by default), not where the session began.
- Keep the flags before the word `resume`, which takes neither `-C` nor `--approve-for-me` after it.
- Pass the session's MODEL, EFFORT and SPEED again, so a resume never falls back to the user's config.

The follow-up says what is left: for one [address-pr](../address-pr/SKILL.md) pass (fix or argue, reply without resolving, push), the links to each open thread and failed check and the text of any finding that has no thread; or what a run that stopped short still owes. It always ends by asking Codex, once that is done, to wait for the checks on the head it leaves and to report them with what changed, and, while it waits, to check the PR for review threads opened since its pass and give each the same address-pr treatment before it reports. A run spends most of its time waiting on CI, and Codex can't be messaged mid-run, so a finding the repo's reviewer posts then is handled in that run rather than after it exits and you resume it. When the run exits, re-review the head it leaves unless you already reviewed it mid-run (step 3).

## 5. Finish

Done is the four conditions from babysit-pr's *When to stop*, with your review as the approval (a comment, when step 3 says so). Where the repo has an AI reviewer of its own, done also needs its approval of the current head (or of an earlier head the current one differs from only by a base-sync, per babysit-pr), on top of yours: its threads and its verdict gate the merge as much as yours do. You can tell the repo has one when its recent PRs carry reviews from an AI reviewer other than you (usually a bot account), or when its AGENTS.md or CLAUDE.md names one; when neither holds, your approval is the only one, and there is nothing to wait for. Tell the user then; merging and deploying stay their call.

Nothing re-invokes you when that reviewer posts: Codex's run has exited, and no babysit or Auto-fix watch is on. So once you have approved, and while its verdict on the current head is missing, watch the PR yourself: run `~/.claude/bin/wait-pr-ready <PR URL> --until-review --head <head sha> --after <time you read the PR>` (installed by this repo's `bin/setup`) as one background Bash command. `--after` makes a verdict that lands between your read and the waiter's first one count as new. It reads the PR with backoff and exits 14 when a review, comment or thread lands (that reviewer's approval included), 12 when the head moves, 15 when the head conflicts with its base, 10/11 when the PR merges or closes, and 13 after babysit-pr's quiet limit with nothing new; a failed read is retried, never taken for a state. On exit 14, read what landed: take threads to Codex through step 4, and on an approval check the four conditions again. On exit 15, resume Codex through step 4 to merge the base and resolve the conflicts: no Auto-fix watch is on for its PR, so nothing else will. If the run leaves that head still conflicting, note that head (in release's state file as `knownConflictHead` under `/release`), bring the conflict to the user instead of resuming for it again, and pass `--known-conflict` on every waiter you arm for that head, whatever woke you. On a new head, re-review it. On exit 13, tell the user the reviewer is still silent. Under `/release`, the watch lasts until delivery: once the PR is approved, arm the plain readiness waiter (`--head`, without `--until-review`), which also exits 15 on a conflict and 14 on findings that leave the PR not ready, so a base that moves or a reviewer that posts while the PR waits still brings you back: take findings to Codex through step 4. Exit 2 or any other code is babysit-pr's hard-error stop: report it. When a resume leaves an item as it was (a thread you still hold open, a check still red, a run cut off or blocked again), bring that item to the user rather than resuming for it again.
