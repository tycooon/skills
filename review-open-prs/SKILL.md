---
name: review-open-prs
description: Use when asked to review all open pull requests / merge requests in a repository at once (rather than one specific PR) and post code-review findings back on each — enumerates every open non-draft PR and reviews each the way review-pr handles a single one, skipping any it has already reviewed that has no new commits since.
---

# Review All Open Pull Requests

Find every open, non-draft pull request (GitHub) or merge request (GitLab) in the current repository, then review each one — dispatching a subagent per PR so the reviews run in parallel.

Before reviewing a PR, check whether it has already been reviewed: look for the most recent review you left under your identity (AI agent name) and compare its timestamp to the PR's latest commit. Skip the PR when no commit is newer than that review — it hasn't changed since the last review. Always review PRs you have never reviewed.

Each subagent does a thorough code review of its pull request, following the same conventions as the review-pr skill: post each finding as an inline review comment anchored to its file and line — submitted as one review with a short severity-ordered summary in the body — never as a single top-level PR comment, which the "address comments" auto-fix and the address-pr skill cannot detect. Prefix each finding with its severity (P0/P1/P2). If there are no findings, it just leaves a short top-level comment saying it did not find anything. Every comment includes your identity (AI agent name).

Skip draft PRs. When you are done, tell the user which PRs you reviewed and which you skipped as unchanged.
