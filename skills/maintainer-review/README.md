# maintainer-review

A [Claude Code](https://claude.ai/code) skill that reviews a branch, diff, GitLab MR or GitHub PR as the repository's maintainer: the person who will own the code after the merge and isn't bound by the author's design decisions.

## Install

```sh
npx skills add fabianeichfeldt/claude-skills --skill maintainer-review
```

## What it does

1. Pins down the business goal in one sentence (from your prompt, the MR/PR description, the ticket or the commits)
2. Learns the repo's rules from `CLAUDE.md` / `AGENTS.md` and existing patterns
3. Splits the diff into numbered logical changes (C1, C2, ...)
4. Reviews five aspects, in parallel subagents for bigger diffs:
   - **Scope:** classifies every change as required, optional, not needed or harmful
   - **Quality:** does the codebase get better or worse?
   - **Dependencies:** packages, layering, cycles, leaks, coupling
   - **Tests:** runs them and finds behavior changes without a test
   - **Security:** always its own agent; checks every new endpoint and scans for open vectors (see below)
5. Merges and verifies the findings, then reports a verdict, an endpoint table, the fixes before merge, the proposed minimal change and follow-ups
6. On request, posts the findings you pick to the MR/PR as **draft** comments, which only you see until you submit the review

It never changes code, commits or posts on its own.

## Security review

The security agent reviews the diff the way an attacker would, and reports a vector only when it can trace the path from attacker input to harm (`file:line` → `file:line`). It checks against:

- **OWASP API Security Top 10 (2023)** for every endpoint the diff adds or changes: HTTP routes, GraphQL, gRPC, webhooks, message consumers, CLI commands, jobs, MCP tools
- **OWASP Top 10 (2025)** for the whole diff, plus your ecosystem's dependency audit (`npm audit`, `pip-audit`, `dotnet list package --vulnerable`, ...)
- **OWASP MCP Top 10 (2025)** when the diff adds or changes an MCP server, client or MCP configuration

Every finding carries its OWASP ID (`A05`, `API1`, `MCP03`, ...).

## GitHub and GitLab

Reviewing needs only `git`. To post comments, the skill detects the forge from the `origin` remote and uses:

- **GitLab:** `glab`, posting draft notes
- **GitHub:** `gh`, posting one pending review

For self-hosted instances it picks whichever CLI is authenticated for the host.

## Usage

> "Review this MR as the maintainer"
> "Is this PR the minimal change for the ticket?"
> "Is this branch secure? Check the new endpoints"
> "Post F1 and F3 to the PR"

## Why

Reviews tend to check whether code works, not whether it should be there. This skill makes "is this the smallest, cleanest, safest change for the goal?" the question, so scope creep, unrelated refactors, new contracts and open attack vectors get caught before they become the maintainer's problem.
