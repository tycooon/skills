## Code review
- Whatever you review (a branch against its base, a PR, a diff), ignore commit messages entirely: don't read, quote or summarize them, and don't flag their quality unless the user explicitly asks about commit hygiene. Review the diff and its effect on the codebase only. `git log` between refs is fine for listing what changed where, but never `git show <commit>` or `--format=%B` to read messages.
- Review against the remote: `git fetch` first, then compare the branch under review (`origin/<branch>`, unless the user names another) with `origin/master`, never with local master.
- Never run tests or linters for a review: CI does that.

