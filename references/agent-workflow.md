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

