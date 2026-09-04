---
name: design-implementation
description: "Use when you have a spec or requirements for a multi-step task, before touching code. Personal fork replacing superpowers:writing-plans and superpowers:executing-plans - when those also match, use this one."
---

# Implementation: Plan, Then Execute

## Overview

Write a comprehensive implementation plan, then implement it in this session. The plan and the execution are one continuous flow — there is no handoff, no menu, and no review gate between them.

Write the plan assuming the engineer has zero context for this codebase and questionable taste. Document everything they need: which files to touch for each task, code, testing, docs they might need to check, how to test it. Give them the whole plan as bite-sized tasks. DRY. YAGNI. TDD. Frequent commits.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain. Assume they don't know good test design very well.

**Announce at start:** "I'm using the design-implementation skill to plan and implement this."

**Context:** If working in an isolated worktree, it should have been created via the `superpowers:using-git-worktrees` skill at execution time.

## Two Hard Rules

These two are the reason this skill exists. Everything else is inherited from upstream.

**1. Execute inline. Never dispatch subagents per task.**

Do NOT invoke `superpowers:subagent-driven-development`. Do NOT invoke `superpowers:executing-plans` — its own text redirects back into the per-task subagent flow whenever subagents are available. Do NOT dispatch a fresh agent per task, per phase, or "just for the review step."

**No exceptions:**
- Not when the plan has many independent tasks
- Not when a task looks self-contained enough to farm out
- Not for "a quick parallel pass" or "just the boilerplate ones"
- Not because context is getting long
- Parallelism is not the goal here. Doing the work in one continuous session is.

**2. Never present the plan for approval.**

The plan is a working artifact for you, not a deliverable for the user. After self-review, go straight into Execution. Do not summarize it and ask "look right?", do not offer execution options, do not ask which approach to take, do not pause for a checkpoint before Task 1.

**No exceptions:**
- Not "just a quick sanity check on the task list"
- Not "confirming before I start on something this large"
- Not paraphrasing the plan in chat and waiting for a reply
- The user approved the spec. That approval covers the plan.

Stopping when you hit a genuine blocker is different, and still required — see [When to Stop and Ask for Help](#when-to-stop-and-ask-for-help).

## Plan Document Location

**Save plans to:** `<working-dir>/.plans/YYYY-MM-DD-<feature-name>-plan.md`, the same git-ignored directory the spec lives in. Inside the working directory so the desktop app can open it; `~/.claude/plans/` is what the app refuses to open. Before the first write, make sure the directory is ignored: `git check-ignore -q .plans || echo '.plans/' >> "$(git rev-parse --git-path info/exclude)"` (per repo, never committed, shared by every worktree of that repo). If the spec was written into another worktree's `.plans/`, copy it into this working directory's `.plans/` first, so both documents open from this session.

**Never commit the plan.** `.plans/` is excluded precisely so it can't dirty a working tree or leak into a PR diff. Do not add it to `.gitignore`, do not `git add` it, do not mention it in a commit message.

## Scope Check

If the spec covers multiple independent subsystems, it should have been broken into sub-project specs during brainstorming. If it wasn't, suggest breaking this into separate plans — one per subsystem. Each plan should produce working, testable software on its own.

## File Structure

Before defining tasks, map out which files will be created or modified and what each one is responsible for. This is where decomposition decisions get locked in.

- Design units with clear boundaries and well-defined interfaces. Each file should have one clear responsibility.
- You reason best about code you can hold in context at once, and your edits are more reliable when files are focused. Prefer smaller, focused files over large ones that do too much.
- Files that change together should live together. Split by responsibility, not by technical layer.
- In existing codebases, follow established patterns. If the codebase uses large files, don't unilaterally restructure - but if a file you're modifying has grown unwieldy, including a split in the plan is reasonable.

This structure informs the task decomposition. Each task should produce self-contained changes that make sense independently.

## Task Right-Sizing

A task is the smallest unit that carries its own test cycle. When drawing task boundaries: fold setup, configuration, scaffolding, and documentation steps into the task whose deliverable needs them; split only where the tasks are genuinely independent deliverables. Each task ends with an independently testable deliverable.

## Bite-Sized Task Granularity

**Each step is one action (2-5 minutes):**
- "Write the failing test" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
- "Run the tests and make sure they pass" - step
- "Commit" - step

## Plan Document Header

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

## Global Constraints

[The spec's project-wide requirements — version floors, dependency limits, naming and copy rules, platform requirements — one line each, with exact values copied verbatim from the spec. Every task's requirements implicitly include this section.]

---
```

## Task Structure

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py:123-145`
- Test: `tests/exact/path/to/test.py`

**Interfaces:**
- Consumes: [what this task uses from earlier tasks — exact signatures]
- Produces: [what later tasks rely on — exact function names, parameter and return types]

- [ ] **Step 1: Write the failing test**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/path/test.py::test_name -v`
Expected: FAIL with "function not defined"

- [ ] **Step 3: Write minimal implementation**

```python
def function(input):
    return expected
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/path/test.py::test_name -v`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
````

## No Placeholders

Every step must contain the actual content an engineer needs. These are **plan failures** — never write them:
- "TBD", "TODO", "implement later", "fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above" (without actual test code)
- "Similar to Task N" (repeat the code — the engineer may be reading tasks out of order)
- Steps that describe what to do without showing how (code blocks required for code steps)
- References to types, functions, or methods not defined in any task

## Remember

- Exact file paths always
- Complete code in every step — if a step changes code, show the code
- Exact commands with expected output
- DRY, YAGNI, TDD, frequent commits

## Self-Review

After writing the complete plan, look at the spec with fresh eyes and check the plan against it. This is a checklist you run yourself — not a subagent dispatch, and not something you show the user.

**1. Spec coverage:** Skim each section/requirement in the spec. Can you point to a task that implements it? List any gaps.

**2. Placeholder scan:** Search your plan for red flags — any of the patterns from the "No Placeholders" section above. Fix them.

**3. Type consistency:** Do the types, method signatures, and property names you used in later tasks match what you defined in earlier tasks? A function called `clearLayers()` in Task 3 but `clearFullLayers()` in Task 7 is a bug.

Fix issues inline. No need to re-review — fix and move straight into Execution.

## Execution

Create todos for the plan's tasks and begin. For each task:

1. Mark as in_progress
2. Follow each step exactly (the plan has bite-sized steps)
3. Run verifications as specified
4. Mark as completed

Never start implementation on main/master without explicit user consent.

## Completing the Work

After all tasks are complete and verified:
- Announce: "I'm using the finishing-a-development-branch skill to complete this work."
- **REQUIRED SUB-SKILL:** Use `superpowers:finishing-a-development-branch`

## When to Stop and Ask for Help

**STOP executing immediately when:**
- Hit a blocker (missing dependency, test fails, instruction unclear)
- The plan has critical gaps preventing you from starting
- You don't understand an instruction
- Verification fails repeatedly

**Ask for clarification rather than guessing.** Don't force through blockers.

If the user updates the spec or the approach needs rethinking, revise the plan and resume — still without presenting it.

## Red Flags — STOP

- About to dispatch a subagent for a task
- About to invoke `superpowers:subagent-driven-development` or `superpowers:executing-plans`
- About to write "Plan complete — here's the breakdown, look right?"
- About to offer execution options
- About to `git add` the plan file
- "This plan is big enough that I should check first"
- "I'll just parallelize the independent tasks"

**All of these mean: write the plan, then implement it yourself, in this session.**

## Integration

- **superpowers:using-git-worktrees** — ensures isolated workspace
- **design-brainstorming** — produces the spec this skill consumes
- **superpowers:finishing-a-development-branch** — completes development after all tasks
