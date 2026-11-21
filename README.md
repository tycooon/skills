# Agent skills

Run `bin/setup` to copy the skills and shared references into the Claude, Codex, ZCODE and OpenCode skills directories. Use `bin/setup --cloud` to register the memory pull hook for a cloud runtime.

## Installed tools

Setup installs executable copies in `~/.claude/bin`: `claude-memory`, `wait-pr-ready`, `gh-review-thread`, `agent-watch` and `review-scan`.

## claude-memory

`claude-memory` shares Claude Code's auto-memories between the Mac, trusted dev-box sessions and cloud sessions. Memories are keyed by `host/owner/repo` from each project's git remote.

- **Git mode** is on when `~/.claude/memory-sync.json` has `"repo"` (a private git repo; `"checkout"` optionally sets the local path, default `~/.claude/memory-repo`). Each project's `projects/<slug>/memory` folder becomes a symlink to `<checkout>/<host>/<owner>/<repo>`. `start` (SessionStart) pulls with a rebase, links the folder and prints one status line (it never imports notes: a folder with notes or a wrong link is moved to `<config>/memory-backup/<slug>-<ts>/` and replaced, with a line telling you to run `migrate`); `end` (SessionEnd, `--background` detaches) stages only regular files inside the checkout, never a symlink (logged), then commits, rebases, pushes and refreshes the Secrets snapshot. `MEMORY.md` merges as a union; any other conflict keeps this machine's version, saves upstream's as `<name>.conflict-<host>-<ts>.md` and adds a pointer line to `MEMORY.md`. Without a usable checkout, `start` behaves like `pull` on Linux; on a Mac it prints nothing, and `end` keeps running `push` of the project folders (with or without `"repo"`) until `migrate` has run. On Linux `end` then does nothing.
- **Cloud sessions** cannot reach git and read a read-only snapshot in Kubernetes Secrets (`claude-memory`, `-1` to `-7`) with `pull`. Every git writer refreshes it from the tracked files only; `push` does that on its own.
- **`migrate`** is run by hand on the Mac (`migrate --from <backup>` imports a `memory-backup` folder that `start` set aside; only `migrate` imports notes, because the memory folders are writable by untrusted processes and the checkout is not): it clones the repo, commits every note, links each project folder (moving the old one to `<config>/memory-backup/`), registers the hooks and refreshes the snapshot. It is safe to run again.
- **Exclusions**: a glob in `~/.claude/memory-sync.exclude` matching `host/owner/repo` leaves that repo's folder alone; one matching `host/owner/repo/note.md` keeps that note on this machine (listed in `<checkout>/.git/info/exclude`, never committed or archived).
- **Hooks**: `claude-memory hooks --git` (alias `--mac`) registers `start` and `end --background` and removes older `claude-memory` entries; `--cloud` registers the SessionStart `pull`. `bin/setup` runs `hooks --git` on a Mac and `hooks --cloud` with `--cloud`.
- **Log**: `CLAUDE_MEMORY_LOG`, default `~/Library/Logs/claude-memory.log` on macOS and `~/.local/state/claude-memory.log` elsewhere. It records every sync result and conflict, never note contents.

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
