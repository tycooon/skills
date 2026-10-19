---
name: pr-test-plan
description: "Write a manual test plan for QA and post it as its own comment on a pull request or merge request: one checkbox per case the tester ticks in place, each saying what to do and what to see. Also updates a plan already posted, and walks the plan on a test environment when asked. Not for reviewing a PR or checking production after a rollout."
---

# Test Plan for a Pull Request

A change is ready for manual testing, and the user wants a plan the tester can run without reading the diff. Write it, post it as a separate comment on the PR (GitHub) or MR (GitLab), and keep it true to the change afterwards.

The plan has one reader: a tester who knows the product but not the code. Three rules follow from that, and they win over everything below.

- **Each item is an action and its outcome.** "Pay 120.00 with a saved card → the order is paid, the receipt shows a fee of 2.40", not "check that fees work". Name the screen, the data and the values; an item the tester has to interpret is an item tested differently by everyone.
- **Each item is a checkbox, `- [ ]`.** Both hosts let anyone with access tick a task-list item right in the comment, so the plan doubles as the tester's progress. Tables and numbered lists can't be ticked; testers end up copying them into their own comment to get boxes, and then there are two plans. No nested sub-items either: a box covers one check.
- **It fits on one screen.** About a dozen items, each at most two lines. Six precise cases beat twenty vague ones; when the list grows, merge the cases that exercise the same branch rather than trimming the detail out of each.

## Find out what to test

Start from the ticket. Its acceptance criteria, if written as cases, already are the plan: turn them into checkboxes and add what the diff shows they miss, rather than inventing a parallel list.

Then read the diff and the new tests, because the plan tests what the code does, not what the ticket hoped for. For every branch the change adds or alters, find the condition that selects it and the value it produces, and write the item with that condition stated. Most wrong expectations in a plan come from a branch condition read too loosely: a rule that applies per payment method rather than per account, on the first event rather than on every one, above a threshold rather than at it. A tester who meets a different result than the plan promised files a bug against correct code, or worse, accepts a wrong result the plan happened to describe. Where the code computes a value the tester will see, compute it for the item (the amount, the count, the label) instead of writing "the correct amount".

A task often ships as several PRs across services. Unless the user asks otherwise, write one plan for the whole task, post it on the main PR, and list the others with the order they must be deployed in.

Removal and refactoring tasks have little new behaviour to test and a lot that must survive. Their plan is mostly regression: name the concrete flows that still go through the touched code, one item each, rather than "everything else works as before".

## Shape

```markdown
## Test plan

**What changed:** one to three plain sentences: what was wrong or missing, and what happens now.

**Setup:** environment, accounts, data, feature switches — only what the tester needs to reach the cases.

- [ ] **Main case.** Action → expected result. Before the fix: what happened instead.
- [ ] Next case — concrete data, concrete result.
- [ ] **Regression:** what nearby must behave exactly as before.

⚠️ Expected, not a bug: behaviour the tester would otherwise report.
```

- **Lead with the main case** and anchor it with what it looked like before the change, so the tester can tell the fix from the old behaviour at a glance.
- **Say where to look** when the outcome isn't on the screen: which log, request body or admin page shows it, or the one console command that reveals it. An outcome the tester can't observe is a check they can't tick.
- **Setup names who does what.** When a case needs something the tester can't do themselves — a setting only a developer can change, a second deploy during the test, data seeded from a console — say so in the setup line, so it gets arranged instead of skipped.
- **Group with `###` headings** only past about ten items or when the plan spans several surfaces (an admin panel and a customer-facing app). Add an **After production rollout** group when a check can only happen there, with the number it should move from.
- **The "expected, not a bug" line** is for behaviour that looks wrong and isn't: a third party's rendering, a deliberate product decision, old records that keep their old shape. Leave it out when there is nothing to say.

Leave out: a retelling of the diff, class and method names the tester never sees, the motivation of the ticket, the list of what automated tests cover, "make sure everything works correctly", the same scenario in three wordings, decorative emoji, and sections kept for symmetry. A case that can't be reproduced by hand — a race, a timing window, a dependency's rare failure — gets at most one line saying an automated test covers it.

The language is the one the tester reads; the project's or the user's instructions say which. Write it plainly, in short sentences, without hard-wrapping lines.

## Post it

Post the plan as a plain comment on the PR, not as a review thread: there is nothing to resolve, and a thread hides it among review findings. Before posting, look through the PR's comments for a plan that is already there, the tester's own included; extend or update that one instead of adding a second.

Send the body from a file, so newlines and Markdown arrive intact:

- **GitHub** — `gh pr comment <pr> --body-file plan.md`. To update: `gh api --method PATCH /repos/<owner>/<repo>/issues/comments/<id> -F body=@plan.md`.
- **GitLab** — `glab api --method POST projects/<id>/merge_requests/<iid>/notes --header "Content-Type: application/json" --input body.json` with `{"body": "<plan>"}` in the file. To update: the same payload with `PUT .../notes/<note_id>`. `glab api --input` sends no content type of its own, and the API answers 415 without the header.

When the user asks for it as a thread, post the same body as a discussion (GitLab) or say that GitHub has no standalone threads and use a comment.

After posting, give the user the link to the comment and show the plan's text in the chat.

## Keep it current

The plan is updated, never re-posted. When the change moves after the plan went up — a new commit changes behaviour, a case turns out wrong, the tester asks about an item — edit the existing comment.

- **Keep the ticks.** The tester may already have ticked items, and their `[x]` live in the same body. Fetch the current body right before editing, change only the items that moved, and leave every other line, ticked or not, exactly as it is. Overwriting from your local copy wipes their progress.
- **Answer a tester's question from the code.** When they report a result that differs from the plan, walk the branch condition again before deciding who is right. If the plan was wrong, fix the item and add the condition that selects the result they saw, so the next reader doesn't trip on it. If the code is wrong, that is a bug in the change, not in the plan.
- **Read it back.** After the edit, fetch the comment and check that the new wording is there and the ticks survived.

## Run it yourself

When the user asks you to walk the plan on a test environment, use the plan as written: the same cases, in order, with the data the items name. Confirm first that the environment runs the head you are testing, since a stale deploy passes or fails cases for reasons that have nothing to do with the change.

Report in the chat in the plan's own shape, `[x]` for a case that passed and `[ ]` for one that didn't or couldn't be run, each with what you observed and an id or value the user can check. Say which items you couldn't exercise and why. Don't tick the boxes in the posted comment: they record the tester's run, not yours. When the run shows the plan itself was wrong or incomplete, a limit nobody wrote down or a case that can't be reached as described, update the comment as above.
