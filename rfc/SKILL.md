---
name: rfc
description: Use when asked to write an RFC, or to put a design — or a spec already written — up for review as a pull request or merge request before building it. Surveys the code, writes the spec docs with their open questions and your recommendations, and opens a docs-only PR tagged [RFC] whose review settles every question before approval, so the user only hears about the questions you and the reviewer can't settle between you. Works on GitHub or GitLab.
---

# Write an RFC

Put a design up for review as a docs-only pull request (GitHub) or merge request (GitLab) before any of it is built. The PR is the RFC. Its spec docs say what today's code does, what should change and how it ships, and they list the questions the evidence could not settle, each with your recommendation. The review settles those questions — the reviewer takes a position on each, and the two of you agree or argue it out — and the PR is approved only once none is left. The user hears only about the questions you and the reviewer could not settle, or that only they can decide.

You are the author. The AI reviewer that covers the repo reviews the RFC with the review-pr skill, and your babysit-pr watch settles its questions with that reviewer. The user is the owner: they make the calls neither of you can, and they merge.

## 1. Read the request

`/rfc <topic>` starts from scratch. `/rfc <path>` publishes a spec that already exists — typically a `.plans/` doc from the design-brainstorming skill.

- **A topic.** Ask the user before writing anything only when the request reads two ways that lead to different designs. Short of that, write your reading into the spec's Goal and let the review challenge it: a question asked now is one the review could have settled without them.
- **An existing spec.** Treat it as your draft and take it through the steps below, in the repo it designs for, which may not be the one its `.plans/` sits in. From then on the RFC doc is the spec; don't keep editing the old copy. The survey re-checks its claims against today's code, since the spec may predate recent changes. The questions the user answered while it was written are theirs: record each as the owner's decision, with an ID from the same sequence as the open questions. The rest of it is your draft, for the review to challenge. Where the survey contradicts one of the owner's decisions, don't change it quietly: reopen it as an open question, with the evidence. The points the spec left open go through step 3 like any other.

## 2. Survey

The design stands on what the code and the data say today, so read them first:

- the code at `origin/master`, fetched now, noting the sha you read;
- the repo's earlier RFCs, specs, ADRs and TODO, and the PRs that built them, which can hold an answer the owner already gave;
- any open PR that changes the same area — name it in Related, and say in the Goal how this one relates to it;
- production or stage data, read-only and under the repo's rules and the global ones, wherever the design depends on how things behave there.

Cite as you go. Every claim about today's behaviour carries its repo-relative `path:line` at the sha you read, or the query and its time window, so the reviewer can check it rather than trust it.

**Found on the way.** A survey this close to the code turns up bugs. List them in the spec's *Found on the way* section. One that is hurting production now doesn't wait for the RFC: fix it in its own PR, or record it where the repo tracks work (an issue, TODO) outside this PR, so it doesn't wait for the merge. Link that from the list, and tell the user in the session. The RFC PR itself stays docs-only.

## 3. Decide what the evidence settles

A choice the evidence settles is not a question. Make it in the design and give its reason, where the reviewer can challenge it like any other part. The open questions are what is left: the choices the evidence splits, and the ones only the owner can make — a priority, an appetite for risk or cost, an irreversible step, a fact outside the repo and its data. Each one lists its options, your recommendation and why, and what would settle it.

Number them `Q1`, `Q2`… across all of the RFC's docs, and never renumber. Threads, decisions and the PR description refer to a question by its ID, so renumbering points every one of those references at the wrong question.

## 4. Write the docs

Put them where the repo's design docs already live — where its newest ones are, if they're spread out — and follow that directory's file naming. With no such directory, use `docs/rfcs/YYYY-MM-DD-<topic>.md`, dated the day you open the PR. Write one doc per implementation PR, so a design too big for one PR splits along its delivery. Write in the language the repo's rules set for docs.

Use this skeleton, scaled to the change, and drop any section that has nothing to say:

```markdown
# <Title>

**Date:** YYYY-MM-DD
**Status:** RFC: open questions under review
**Area:** <the modules and components it touches>
**Related:** <the other parts, earlier RFCs, PRs>

## Goal
## Today
<current behaviour, with its evidence>
## Design
## Delivery
<the implementation PRs, in order>
## Testing
## Rollout
<how it ships, how you'll know it worked, how to back out>
## Risks
## Found on the way
## Open questions
## Decisions
```

An open question reads like this:

```markdown
**Q3.** <the question>
- Options: (a) … (b) …
- Recommendation: (b), because …
- Settles it: <the evidence or experiment that would decide it>
```

One that is waiting on the owner stays in the open questions and gains a line with each side's position and what each option would change:

```markdown
- Waiting on the owner. Author: (b), because …; <the reviewer>: (a), because …; (a) would …, and (b) would …
```

A settled question moves to the decisions and states its answer in words, since its options leave with it:

```markdown
- **Q3. <the question, in a few words>:** <the answer>, because <the reason>. Settled with <the reviewer> on <date>.
```

`<the reviewer>` is the name the reviewer signs its comments with. A decision the owner made ends `Decided by the owner on <date>.` instead.

The Status moves from `RFC: open questions under review` to `RFC: questions settled on <date>`, then to `Implemented in <PRs>`. An RFC that opens with no questions starts at `RFC: questions settled on <date>`, so a merged doc never claims to be under review; a question added after that puts it back to `RFC: open questions under review`.

**Superseding.** When the design replaces an accepted RFC — one already merged — the same PR sets the old doc's Status to `Superseded by <new doc>`, or to `Partly superseded by <new doc> (§n)` when it replaces only some sections, and the new doc names the old one in Related. Two docs that each read as current leave a later reader building the wrong one.

## 5. Open the PR

Commit the docs, plus whatever pointers the repo keeps to its designs (TODO, an index), and nothing else: an RFC changes no code. Open it as a non-draft PR. Review sweeps skip drafts, so a draft RFC never gets its questions settled.

Title it `[RFC] <imperative summary>`. Where the host or the repo requires titles to start with something else, a Jira key for example, the tag goes right after it: `ABC-123 [RFC] Move rate limits into the gateway`.

Write the description in this order:

1. Two to four sentences: what it proposes and why.
2. One bullet list per doc, headed by its document title (the top-level heading) as the clickable Markdown link text to that file on the PR/MR source branch. Use the source repository for fork PRs. GitHub links use `https://github.com/<owner>/<repo>/blob/<source-branch>/<path>`; GitLab links use `https://<host>/<namespace>/<repo>/-/blob/<source-branch>/<path>`. Anchor these links to the branch name so they follow RFC revisions; keep them current if the source branch or document path changes. Verify each linked file exists on that branch.
3. **Open questions** — a table of ID, question, recommendation and state (`open` or `waiting on the owner`). The full text lives in the spec; the table is the view at a glance. Leave it out when there are none.
4. **Decisions** — each settled question's ID and answer, and who settled it.
5. **Found on the way**, when the survey found anything.
6. The review block below, verbatim.
7. "Docs-only."

For example, a document heading on GitHub reads:

```markdown
### [Registry routing design](https://github.com/owner/repo/blob/codex/routing-rfc/docs/rfcs/routing-design.md)
```

Where the host has a description template of its own, fill that and carry these parts inside it; the review block goes in verbatim either way. It tells every reviewer, the ones that don't run review-pr included, what approving this PR means:

```markdown
**How to review this RFC.** This PR proposes a design and changes no code. Review the design itself, and settle every open question before approving:
- Give each open question its own thread on its line: agree with the recommendation, or argue for another option, with evidence.
- Don't approve while any question is open or waiting on the owner.
- A question goes to the owner only when the author and the reviewers can't settle it between them, or when only the owner can make the call.
```

## 6. Babysit it

Watch the PR with the babysit-pr skill. Its address-pr passes settle the questions with the reviewer; address-pr's step 2 says how, and when a question goes to the user. The watch ends once the reviewer approves the settled text. If it ends without any review, because no AI reviewer covers the repo, nothing will settle the questions: put them to the user in the final report, each with your recommendation.

Merging is the user's call. Once the RFC has merged, the design-implementation skill builds it from the repo's copy of the doc.
