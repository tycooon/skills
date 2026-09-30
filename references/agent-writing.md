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
