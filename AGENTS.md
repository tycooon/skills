<!-- The global agent rules. Only rules that hold in any environment go here: the Mac loads this file through its ~/AGENTS.md symlink, and Claude cloud sessions get it from their environment's setup script. Rules tied to the Mac go in ~/.claude/mac.md. -->

A repo's own AGENTS.md or CLAUDE.md wins over this file where the two conflict. Follow the repo, and mention the override rather than applying it silently.

## Workflow
- Verify the current working directory before editing files or running commands.
- Finished code work ends in a non-draft PR. Open a draft only for unfinished work.
- Work whose only deliverable is findings is reported in chat, with no commit, push or PR. That covers an investigation, a census, a competitor or lane study, a scoping or feasibility check, an RCA, and any "analyze X and tell me Y" request. Write a PR, doc or write-up only when the user explicitly asks for one; a repo's existing `docs/research/` tree is not such a request. If the findings look worth keeping, offer to persist them and let the user decide.
- Don't infer prod configuration from migrations or settings' default values: an admin UI changes them at runtime.

## Branches and pushes
- Follow-up work to a merged PR goes in a new PR, on a new branch from `origin/master`, never on the old branch. So before every push, check whether the current branch is already merged into `origin/master`, and if it is, move to a new branch first.
- Immediately before every `git push`, the first one and every follow-up (review fixes, CI retries) alike, run `git fetch origin && git merge origin/master`. Master moves while you write specs, run CI or wait for a "go", so a sync from earlier in the session is stale by the time you push, and the branch lands behind, sometimes on top of a commit that touched the same files.
- When a branch's diff carries unrelated commits, reset it before opening the PR rather than shipping mixed changes.
- Before claiming a PR covers some work, check its state with `gh pr view <number>`: it may have been squash-merged or closed.

## Merging, deploying and CI
- Merging a PR and deploying to prod **are** allowed once the user has sanctioned that specific action in the conversation ("deploy", "merge and deploy", "I allow it"). The standing "deliver PRs, don't apply to prod" rule covers unprompted applies, not ones the user asked for. A sanction covers the action named, on the change at hand, not later deploys of other work.
- The auto-mode classifier may still refuse (`bin/prod` as [Production Deploy], `gh pr merge` as [Merge Without Review]). When it does, say so and hand the user the command rather than retrying it.
- When told to merge into master, merge directly rather than opening a new PR.
- Report CI results once they are final, never intermediate ones.
- Never suggest watching CI and merging once it's green: that's the user's job.

## Development
- Never duplicate logic (DRY): reuse or extract it rather than mirroring it, even when it looks simple. An exception needs a good justification and the user's approval.
- When a Ruby project has a linter or formatter, run it after your changes and fix what it reports.
- Never pipe rspec through `tail`: keep the full output, so you never re-run it just to see more.
- Never run `find /`: it scans the whole filesystem and is extremely slow. Scope file searches to the project root, and locate gems with `bundle info <gem>` or `gem which <gem>`.

## Code review
- Whatever you review (a branch against its base, a PR, a diff), ignore commit messages entirely: don't read, quote or summarize them, and don't flag their quality unless the user explicitly asks about commit hygiene. Review the diff and its effect on the codebase only. `git log` between refs is fine for listing what changed where, but never `git show <commit>` or `--format=%B` to read messages.
- Review against the remote: `git fetch` first, then compare the branch under review (`origin/<branch>`, unless the user names another) with `origin/master`, never with local master.
- Never run tests or linters for a review: CI does that.

## Comments and documentation
By default a comment is not needed. A comment is an exception and has to earn its place: first make the code clear through naming and structure, and write a comment only if knowledge is left over that the code cannot show. **This section outranks "match the style of neighbouring files"**: noisy or stale comments next door are not a licence to add more.

Three kinds fail review: retelling the code, HOW instead of WHY, and bureaucratic phrasing or the wrong language (see Writing).

A comment is warranted when the knowledge is not in the code: the reason for a decision, an invariant, a constraint, an unexpected choice, a workaround for someone else's bug, a link to a ticket or ADR. For a genuinely hard algorithm, explaining HOW is fine too. Style: short phrases, simple words, one thought per sentence, 1-2 lines.

Before handing off the diff, re-read every comment line you added and delete each one that carries no reason, invariant or constraint, i.e. every one that explains what the code does rather than why.

## Writing
- Use English for everything you write for a repo and its reviews: commit messages, PR/MR titles and descriptions, review comments and replies, code comments and docs. That holds even when everyone else writes in another language; only a repo that mandates another language overrides it.
- Never hard-wrap prose (no line breaks at ~80 columns). Markdown files, PR/MR descriptions and comments keep each paragraph on one line, and editors and renderers soft-wrap it. That goes for new text and for text you edit.
- Text other people read (PR/MR descriptions, review comments, issues, commit messages, docs, Jira and Confluence) uses repo-relative paths or permalinks, never local paths: a local path resolves only on the machine that wrote it.
- No attribution lines: no `Co-Authored-By` in a commit message, no `Generated with Claude Code` footer or the like in a PR/MR description.

## PR titles and descriptions
Write both for a reviewer who wasn't in the session. A repo's PR/MR template or the host's own convention sets the sections and the title format; everything else here still applies inside them.

- **Title**: match the repo's recent PR titles; otherwise a short plain phrase naming the outcome, like `Retry webhook deliveries that time out`.
- **Description**: `##` sections in this order, each one only when it has content:
  - **What**: the change as it behaves once merged, with the background a reviewer needs. Name related PRs by number (what this one builds on or supersedes) and anything an earlier version of this PR carried and dropped.
  - **Why**: the problem and the evidence for it (UTC timestamps, ids, counts, latencies, log lines). Say which part this PR fixes and what covers the rest.
  - **How**: a bullet per mechanism, opening with a bold phrase (`- **Retry on timeouts only.** …`): the functions, settings and log lines it touches, and why this approach. A PR made of separate parts gives each part its own section, named after it, in place of How.
  - **Risk**: what could go wrong, and what stays unchanged and why. Security checks a repo requires go here.
  - **Testing**: each command run with its result (`bundle exec rspec`: 1,902 examples, 0 failures), what the new tests cover, and how any manual or replay check was run.
  - **After deploy**: what to watch, where, and the baseline to compare it against, plus any live state that changes what you'll see.
  - **Notes**: side findings and deferred work, each with where it's tracked.
- Every claim is specific enough to check: a number, an id, a name in backticks, a PR link. Items compared on the same attributes go in a table.
- The sections scale with the change: a one-line fix gets a sentence of What and its Testing.
- Keep the title and description true to the current diff. When a push changes anything they say, edit them right after the push.
- Keep other projects out: a PR names only its own repo's systems and data.
