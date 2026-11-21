# Agent skills

This repo holds only skills: one folder per skill, each with a `SKILL.md`.

The shared references, the global agent rules and the helper tools (`~/.claude/bin`) live in a separate private repo. Install it alongside this one: the skills link `../references/…`, which that repo mirrors beside the skills in every runtime, and some skills call its tools.

Run `bin/setup` to copy the skills into the Claude, Codex, ZCODE, OpenCode and Gemini skills directories. `--cloud` is accepted as a no-op for old cloud environments.

Run the tests with:

```bash
for test in tests/test-*; do "$test" || exit; done
```
