# Agent skills

Run `bin/setup` to copy the skills and shared references into the Claude, Codex, ZCODE and OpenCode skills directories. Use `bin/setup --cloud` to register the memory pull hook for a cloud runtime.

## Installed tools

Setup installs executable copies in `~/.claude/bin`: `claude-memory`, `wait-pr-ready`, `gh-review-thread`, `agent-watch` and `review-scan`.

`gh-review-thread` exposes only two GitHub review-thread writes:

```text
gh-review-thread reply THREAD_ID BODY_FILE
gh-review-thread resolve THREAD_ID
```

Thread IDs must have the `PRRT_` node-id form. Replies read a regular UTF-8 file, refuse empty or whitespace-only text, append `<!-- posted-by-agent -->` unless already at the end, and print the new comment ID. Resolves print `resolved`. The tool uses `gh` authentication and pins API requests to `github.com`.

Before a write, it reads the thread's repository owner and requires that login in `~/.config/gh-review-thread/owners`, one login per line (case-insensitive). Create this file outside the repository with the owners you intend to permit. A missing or empty file refuses every write. Exit codes are 0 for success, 2 for usage or invalid input, 3 for an allowlist refusal, and 4 for an API error; errors go to stderr.

To allow these operations without permitting arbitrary GraphQL mutations, add the Claude permission rule `Bash(<home>/.claude/bin/gh-review-thread:*)`, replacing `<home>` with your absolute home directory. Invoke that installed path. The helper does not support GitLab; use the skills' existing `glab api` instructions there. Raw GitHub API calls remain a fallback only where the helper is not installed, never to bypass its allowlist.

Run all tests with:

```bash
for test in tests/test-*; do "$test" || exit; done
```
