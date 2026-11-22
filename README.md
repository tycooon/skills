# Agent skills

This repo holds agent skills (one folder per skill, each with a `SKILL.md`) and the four helper tools those skills call.

- [`review-loop`](review-loop/SKILL.md) keeps reviewing one PR or MR in the current session as it changes.
- [`rfc-review`](rfc-review/SKILL.md) reviews an RFC from the problem up: it measures the problem, tests the chosen path against it and offers a smaller one when it fits.

Run `bin/setup` to copy the skills into the Claude, Codex, ZCODE, OpenCode and Gemini skills directories and install executable copies of the four tools in `~/.claude/bin`. `--cloud` is accepted for cloud environments and runs the same install.

The shared references, the global agent rules and the remaining helper tools (such as `claude-memory` and its hooks) live in a separate private repo. Install it alongside this one: the skills link `../references/…`, which that repo mirrors beside the skills in every runtime.

## Installed tools

`bin/setup` installs executable copies at mode `0755` in `~/.claude/bin`:

- `wait-pr-ready` — background PR/MR readiness waiter used by `babysit-pr`, `release` and the `review-scan` skill.
- `gh-review-thread` — allowlisted GitHub review-thread reply and resolve helper used by `review-pr`, `babysit-pr` and `address-pr`.
- `review-scan` — scheduled PR/MR review scanner and single-PR waiter used by `review-pr`, the `review-scan` skill, `review-loop`, `babysit-pr` and `address-pr`.
- `agent-watch` — shared watch engine loaded by `review-scan` from beside itself (along with `wait-pr-ready`).

### `gh-review-thread`

`gh-review-thread` exposes only two GitHub review-thread writes:

```text
gh-review-thread reply THREAD_ID BODY_FILE
gh-review-thread resolve THREAD_ID
```

Thread IDs must have the `PRRT_` node-id form. Replies read a regular UTF-8 file, refuse empty or whitespace-only text, append `<!-- posted-by-agent -->` unless already at the end, and print the new comment ID. Resolves print `resolved`. The tool uses `gh` authentication and pins API requests to `github.com`.

Before a write, it reads the thread's repository owner and requires that login in `~/.config/gh-review-thread/owners`, one login per line (case-insensitive). Create this file outside the repository with the owners you intend to permit. A missing or empty file refuses every write. Exit codes are 0 for success, 2 for usage or invalid input, 3 for an allowlist refusal, and 4 for an API error; errors go to stderr.

To allow these operations without permitting arbitrary GraphQL mutations, add the Claude permission rule `Bash(<home>/.claude/bin/gh-review-thread:*)`, replacing `<home>` with your absolute home directory. Invoke that installed path. The helper does not support GitLab; use the skills' existing `glab api` instructions there. Raw GitHub API calls remain a fallback only where the helper is not installed, never to bypass its allowlist.

## Tests

Run all tests with:

```bash
for test in tests/test-*; do "$test" || exit; done
```
