---
name: review-open-prs
description: Use when asked to review all open pull requests / merge requests in a repository at once (rather than one specific PR) and post code-review findings back on each — enumerates every open non-draft PR and reviews each the way review-pr handles a single one.
---

# Review All Open Pull Requests

Find every open, non-draft pull request (GitHub) or merge request (GitLab) in the current repository, then review each one — dispatching a subagent per PR so the reviews run in parallel.

Each subagent does a thorough code review of its pull request and posts the findings as a single PR comment, prioritized by severity (P0/P1/P2). If there are no findings and everything looks good, it just leaves a short comment saying it did not find anything. Comments should always include your identity (AI agent name).

Skip draft PRs. When you are done, tell the user which PRs you reviewed.
