# Global Agent Rules

Repository instructions override these rules where they conflict; mention a material override. Verify the working directory before commands or edits. Be proactive within the user's authorized scope and verify changes with relevant checks.

## Core workflow

- Findings-only work ends in chat without a commit, document or PR unless requested. Finished code work ends in a non-draft PR; drafts are for unfinished work.
- Merge/deploy only after the human authorizes the specific action. Do not infer runtime production configuration from migration defaults.
- Preserve DRY; an exception needs justification and user approval. Use English for repository content unless its rules require another language; no attribution footers.
- Load only the task's applicable instructions. Do not repeat injected instructions or read whole histories, manuals and transcripts; search first and extract relevant sections with bounded output.

## Required references

Resolve these paths relative to this file's actual source directory (follow its symlink, if any). Read the applicable reference before the action; its detailed rules remain authoritative.

| Action | Read |
|--------|------|
| Development, branches, commits, pushes, PR delivery or authorized merge/deploy | `references/agent-workflow.md` |
| Code review | `references/agent-review.md` |
| Repository prose/comments, issues, PR titles/descriptions or review replies | Relevant sections of `references/agent-writing.md` |

On macOS, load relevant sections of `~/.claude/mac.md` for worktree creation, local tool/runtime work, production console access or file delivery. Load `~/.claude/cadolabs.md` only for Cadolabs projects on `gitlab.task4work.info`, and only the sections relevant to the task. Project bootstrap follows the repository's routing rules after these applicable global rules; no competing skill-first bootstrap is required.
