<!-- The global agent rules. Only rules that hold in any environment go here: the Mac loads this file through its ~/AGENTS.md symlink, and Claude cloud sessions get it from their environment's setup script. Rules tied to the Mac go in ~/.claude/mac.md. -->

## Workflow
- Always verify current working directory before editing files or running commands.
- When code work is finished, open a non-draft PR (draft only if the work is unfinished). Work whose only deliverable is findings — an investigation, census, competitor/lane study, scoping or feasibility check, RCA, or any "analyze X and tell me Y" request — is reported in chat, with no commit, push or PR, unless the user explicitly asks for a PR, doc or write-up; a repo's existing docs/research/ tree is not such a request. If the findings look worth keeping, offer to persist them and let the user decide.
- Always merge latest origin/master (use git fetch) into your branch before pushing your changes. This means **immediately before every `git push`**, not once when the work is finished: `git fetch origin && git merge origin/master`. Master moves while you write specs, run CI, or wait for a "go" — a sync from earlier in the session is stale by the time you push, and the branch lands behind, sometimes on top of a commit that touched the same files. Applies to the first push and to every follow-up push (review fixes, CI retries) alike.
- Per-project instructions (a repo's own AGENTS.md / CLAUDE.md) always win over the rules in this file when the two conflict. Follow the repo, and mention the override rather than applying it silently.
- Always use English for commit messages and PR/MR titles / descriptions / comments, even when everyone else is using other languages. This covers text written for the MR — commit messages, MR title/description, review replies — and is overridden by a repo that mandates another language.
- Before claiming a PR covers work, verify its state with `gh pr view <number>` - check if it was squash-merged/closed
- After a PR is merged, create a NEW PR for follow-up work rather than reusing the old branch
- When diffs include unrelated commits, reset the branch before opening a PR rather than shipping mixed changes
- When told to merge into master – don't open a new PR, just merge it directly into master.
- Don't report intermediate CI results, only final results.
- Never suggest watching CI and merging PR when green – that's a user's job.

## Code review
When reviewing code (branch vs base, PR, diff): ignore commit messages entirely. Do not read them, quote them, or summarize them. Review the actual diff and its effect on the codebase only. `git log` between refs is fine for *listing what changed where*, but never use `git show <commit>` for messages or `--format=%B`. Do not flag commit message quality as review feedback unless the user explicitly asks about commit hygiene.

Always review against **remote** master (`origin/master`), not local master. Run `git fetch` first so the base is current. Same for the branch under review — use `origin/<branch>` unless the user specifies otherwise.

Never run tests/linters when asked for review – this should be handled by CI.

## Development
- Never duplicate logic (DRY principle). Always prefer reusing/extracting the logic over "mirroring" it, even if it looks simple. Exceptions to this need good justification and user's approval.
- In case ruby project has linter/formatter integrated – always run it and fix after your changes.
- Don't manually hard-wrap text in markdown files (no inserting line breaks at ~80 cols). Let paragraphs be single long lines; editors and renderers soft-wrap. Applies to new writes and when editing existing text.
- Don't make assumptions about prod configuration just by reading migrations and settings' default values — there is admin UI that changes them at runtime.
- Deploying to prod, and merging a PR, ARE allowed once I have sanctioned that specific action in the conversation ("deploy", "merge and deploy", "I allow it"). The standing "deliver PRs, don't apply to prod" rule covers unprompted applies, not ones I asked for. Sanction covers the action I named, on the change at hand — not later deploys of other work. The auto-mode classifier may still refuse (`bin/prod` as [Production Deploy], `gh pr merge` as [Merge Without Review]); when it does, say so and hand me the command rather than retrying it.
- Never use `tail` for rspec runs – capture full output instead, so that you don't need to re-run the rspec.
- Before pushing you changes, always check if current branch has been already merged into origin/master. If so, create a new branch from origin/master and start working on it.
- Never include Co-Authored-By in commit messages.

## Comments and documentation
By default a comment is not needed. A comment is an exception and has to earn its place: first make the code clear through naming and structure, and write a comment only if knowledge is left over that the code cannot show. **This section outranks "match the style of neighbouring files"** — noisy or stale comments next door are not a licence to add more.

Three kinds fail review: retelling the code, HOW instead of WHY, and bureaucratic phrasing or the wrong language (English by default; a repo may mandate otherwise — see the language rule in Workflow).

A comment is warranted when the knowledge is not in the code: the reason for a decision, an invariant, a constraint, an unexpected choice, a workaround for someone else's bug, a link to a ticket/ADR. For a genuinely hard algorithm, explaining HOW is fine too. Style: short phrases, simple words, one thought per sentence, 1-2 lines.

Before handing off the diff, re-read every comment line you added and delete each one that carries no reason, invariant, or constraint — i.e. every one that explains what the code does rather than why.

## PR titles and descriptions
Write both for a reviewer who wasn't in the session. A repo's PR/MR template or the host's own convention sets the sections and the title format; everything else here still applies inside them.

- **Title**: match the repo's recent PR titles; otherwise a short plain phrase naming the outcome, like `Retry webhook deliveries that time out`.
- **Description**: `##` sections in this order, each one only when it has content:
  - **What** — the change as it behaves once merged, with the background a reviewer needs. Name related PRs by number — what this one builds on or supersedes — and anything an earlier version of this PR carried and dropped.
  - **Why** — the problem and its evidence: UTC timestamps, ids, counts, latencies, log lines. Say which part this PR fixes and what covers the rest.
  - **How** — a bullet per mechanism, opening with a bold phrase (`- **Retry on timeouts only.** …`): the functions, settings and log lines it touches, and why this approach. A PR made of separate parts gives each part its own section, named after it, in place of How.
  - **Risk** — what could go wrong, and what stays unchanged and why. Security checks a repo requires go here.
  - **Testing** — each command run with its result (`bundle exec rspec`: 1,902 examples, 0 failures), what the new tests cover, and how any manual or replay check was run.
  - **After deploy** — what to watch, where, and the baseline to compare it against, plus any live state that changes what you'll see.
  - **Notes** — side findings and deferred work, each with where it's tracked.
- Every claim is specific enough to check: a number, an id, a name in backticks, a PR link. Items compared on the same attributes go in a table.
- The sections scale with the change: a one-line fix gets a sentence of What and its Testing.
- Keep the title and description true to the current diff. When a push changes anything they say, edit them right after the push.
- Keep other projects out: a PR names only its own repo's systems and data.
- No attribution footers (`Generated with Claude Code` and the like).

## Forbidden Commands
- NEVER use `find /` - it scans the entire filesystem and is extremely slow
- For gem searches, use `bundle info <gem>` or `gem which <gem>` instead of find
- Scope file searches to the project root.

## Text Formatting
- Never hard-wrap prose in markdown files (README.md, docs, etc.) or in PR/MR descriptions and comments - use soft-wrapping only
- Text other people read — PR/MR descriptions, review comments, issues, commit messages, docs, Jira and Confluence — uses repo-relative paths or permalinks, never local paths: a local path resolves only on the machine that wrote it.
