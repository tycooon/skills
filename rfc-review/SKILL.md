---
name: rfc-review
description: "Review an RFC PR or MR from the problem up: measure the problem, test the chosen path against it, propose a smaller one when it fits, then keep reviewing until it is settled."
---

# Review an RFC

An ordinary review checks a change against its own description. An RFC review first asks whether the description is solving the right problem at the right size. Use this for a PR or MR whose title carries `[RFC]`, when the owner names it for review. It adds to [review-pr](../review-pr/SKILL.md)'s *RFC pull requests* rules, which still govern what is posted: one thread per finding and per question, no approval while a question is open.

Assume nothing is decided, even where the RFC records an owner decision: a decision made before the evidence can be reopened by it. Say so when you argue against one, and leave the choice to the owner.

## 1. Read the problem before the solution

Read the specs, then the issues and incidents they cite, then the docs for how the area works today. Write down, for yourself, the problem in one paragraph and the harm it is supposed to cause. If you can't, that is the first finding.

## 2. Measure the problem

An RFC argues from incidents; a review checks them against production. Where the repository's rules allow read-only access, measure:

- **The loss the RFC exists to prevent:** how often it happened, on what, and what it cost, over a stated window. Find the system's own record of that loss before trusting a proxy for it.
- **What each cited symptom costs:** money, time and pages, not counts alone.
- **Whether the stated cause is where the loss is.** Trace a few real cases end to end.
- **The numbers each open question says would settle it.** If a question names its test, run the test.

Give every number with its window and source. Say what you could not measure.

## 3. Check the RFC's picture of today

Verify its claims about current code against the code, as review-pr asks. Look hardest at the claims the design leans on: who decides what at run time, what is stored and what is derived, which process may act on which resource. A design built on a wrong picture of today fails in the part nobody reviewed.

These three steps are independent, so run them in parallel where the runtime allows: one pass per part of the RFC for the code, one for production. Verify yourself the findings your review will rest on.

## 4. Test the chosen path

- **Proportion.** Set each part's cost (new state, new roles, new duties in critical processes) beside the measured harm it removes. Name the parts that carry most of the machinery and what they buy.
- **The normal state.** Find the condition the design treats as an exception and check how often it holds. A design that pages, stalls or starves in the common case is wrong at the root, however it handles the rare one.
- **What it removes.** For each rule, table or timer that goes, find what it protected and what protects that now.
- **Failure and rollout.** Follow a failure through the new paths under the conditions that trigger them. Check that backing out works against the migrated state.
- **Neighbours.** Read the other designs in flight in the same area and check the two agree where they meet.

## 5. Offer another path when one fits

If a smaller or different design reaches the same goals, say so as a `[new question]` thread on the line that records the approach, with:

- the evidence that the chosen path is larger than the harm;
- the alternative in a few numbered steps, each pointing at the thread that argues it;
- what the alternative still needs from the original.

If the path is right, say that plainly and keep the findings to its gaps.

## 6. Post and report

Post per review-pr. In the session, tell the owner, in this order:

1. the verdict on the path in one sentence;
2. the problem as measured;
3. what could go wrong with the proposal;
4. the alternative, if you proposed one;
5. your position on each open question, with the data behind it.

Bring a question only the owner can settle to the owner in the session. When they answer, record the decision on its thread so the author can move it into the spec.

## 7. Keep reviewing

Use [review-loop](../review-loop/SKILL.md) until the RFC merges, closes or goes quiet. Two rules for the later rounds:

- **Review your own proposal as hard as the author's.** When the author adopts your alternative, the next round is a first review of a new design. Look for the gaps your suggestion left, and name them as yours.
- **Correct your own numbers in the open.** When new evidence shows a figure you gave was wrong or too narrow, post the correction as a finding on the line that now relies on it, before anything else. A decision may already rest on it.
