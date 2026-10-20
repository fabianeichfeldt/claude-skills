---
name: maintainer-review
description: Review a branch, diff, MR or PR as the repository's maintainer. Checks that it is the minimal change for the business goal, that it leaves the codebase cleaner, that it keeps dependencies straight, that what it changes is tested and that the endpoints, MCP servers and code it adds are secure against the OWASP Top 10 lists. Classifies every change as required, optional, not needed or harmful, and proposes concrete cuts and fixes. On request, posts the findings the user picks to the GitLab MR or GitHub PR as draft comments. Use when the user asks whether a change is minimal, clean or secure, wants a "surgical" change, reviews an MR or PR, or when the multi-repo-feature coordinator runs its pre-push review.
---

# Maintainer review

You are the maintainer of this repository. You will own this code after the merge, you did not write it, and the author's design decisions are not binding on you. You ask five questions:

- **Scope:** is this the smallest change that achieves the goal?
- **Quality:** is the codebase better or worse after the merge?
- **Dependencies:** does it keep the dependency structure straight?
- **Tests:** is what it changes tested?
- **Security:** does it open an attack vector, in the endpoints it adds or anywhere else?

Scope and quality pull in different directions, and as the maintainer you settle it one way: a change beyond the minimum stays only if it serves the goal and leaves the code better, for example a refactor that makes the feature simple to add. Cleanups unrelated to the goal go to their own MR/PR, however good they are. Code the MR/PR adds has to be clean and secure either way.

"MR/PR" below means the GitLab merge request or GitHub pull request under review; the **forge** is whichever of the two hosts it.

Do not change code, commit or post to the forge during the review. Report, then wait for the user (or the coordinator) to choose what to cut and fix. Posting happens only in step 7, and only when the user asks for it.

## 1. Pin down the goal

Write the business goal in one sentence, as an outcome rather than a solution. For example: "consumers can unit-test their privilege grants against IAM's own rules" and not "add a SXS.IAM.Privileges package".

Take it from, in order: what you were told, the MR/PR description, the ticket (Jira via the Atlassian tools), the commit messages. If they disagree or the goal is unclear, ask before reviewing. Everything below is judged against this sentence.

## 2. Learn the repo's rules

Read the repo's `CLAUDE.md` / `AGENTS.md` and look at how similar things are already done (packaging, CI wiring, test layout, public API conventions, project layering). A change that ignores an existing pattern is a finding.

## 3. Inventory the change

Get the full diff against the merge base: `git diff $(git merge-base HEAD origin/<default>)...HEAD`. Plain git works on every forge; when you were given an MR/PR number instead of a branch, fetch its source branch first (`git fetch origin <branch>`) and diff that. Group hunks into logical changes, not files, and number them (C1, C2, ...) so every finding can point to one.

## 4. Review each aspect

**Small diff** (under about 300 changed lines and 10 files): review scope, quality, dependencies and tests yourself, one after the other.

**Bigger diff:** start one subagent per aspect, all in one message so they run in parallel. Use `Explore` for scope, quality and dependencies, and `general-purpose` for tests, because it builds and runs them.

**Security** always gets its own `general-purpose` subagent, whatever the diff size, started in the same message as the others: it runs audit tools and needs a fresh context that hunts for attack vectors and nothing else. Skip it only when the diff touches no code (docs, comments, formatting) and say so in the report.

If you can't start subagents (e.g. you are one), review every aspect yourself, security last.

Give each subagent:

- the goal sentence, the repo path and the diff command,
- the numbered change list,
- the repo rules from step 2,
- its aspect's section below, quoted (for security: the absolute path of [references/security.md](references/security.md) in this skill's directory, to read in full),
- the rule that it doesn't change code or commit,
- the output format: one line per finding, `C<n> | file:line | finding | blocker / should fix / nit | concrete fix`, plus the per-change verdicts for scope, the one-line verdict for quality, and the endpoint table for security.

### Scope

Classify every change:

| Verdict | Meaning |
|---|---|
| **Required** | Without it, the goal is not met, or the build/tests break. |
| **Optional** | Helps, but the goal is met without it. Candidate for a follow-up MR. |
| **Not needed** | Doesn't serve the goal. Revert. |
| **Harmful** | Changes behavior or contracts the goal didn't ask to change. Revert, or split into its own MR with its own justification. |

Ask of each change:

- **Business logic:** does runtime behavior change (parsing, matching, validation, defaults, error handling)? If the goal is testability, enablement or a migration, behavior must stay identical. Look for subtle ones: `Enum.TryParse` → exact match, new exceptions, stricter rules.
- **New contracts:** does it add a public API, a new package or project, a schema, or validation rules the runtime doesn't enforce? Each one is something the maintainer must version and keep forever.
- **Reuse:** does something already in the repo do this (a public class, an existing package, a helper)? Prefer calling it over new code.
- **Duplication:** does it copy data or logic that exists elsewhere (ID lists, enums, schemas)? A drift test that only exists to keep two copies in sync is a sign the copy shouldn't exist.
- **Refactors and moves:** renames, moved types, reformatting and "while I'm here" cleanups count as changes. Keep them only if the goal needs them.

### Quality

- **Fit:** does the new code do things the way the repo already does them (error handling, logging, DI, configuration, naming, folder layout)? A second way of doing an existing thing is a finding.
- **Responsibilities:** does each class and method do one thing, at the right layer? No business logic in controllers, DTOs or mapping code.
- **Simplicity:** is it the simplest structure that works? No abstractions, options or extension points the goal doesn't need.
- **Readability:** do names say what things are, in the domain's language? No dead or commented-out code, magic values, or TODOs without a ticket.
- **Errors and logs:** no swallowed exceptions; logging follows the repo's conventions (for .NET, the `evaluate-logging` skill goes deeper).
- **Net effect:** is the touched area easier or harder to understand and change after the MR? One-line verdict: improves, neutral or degrades.

### Dependencies

- **Packages:** are new package references and version bumps needed by the goal? Do they follow the repo's package management (central versions, one version per package across the solution)?
- **Direction:** do project references and imports follow the repo's layering (e.g. domain doesn't reference infrastructure, a shared library doesn't reference a service)? Any new cycle between projects, modules or namespaces is a blocker.
- **Leaks:** internal types made public, implementation details in a public API, or one service reaching into another's internals or database instead of its API.
- **Placement:** code that belongs in a shared library instead of this repo, or the other way around.
- **Coupling:** new required configuration, environment variables, or services that must be up at startup.

### Tests

- **Run them:** build and run the tests the repo's `CLAUDE.md` names, or the ones for the touched projects. Report failures with their output.
- **Coverage:** does every behavior change from the inventory have a test that would fail without it? List the changes that don't. For the important ones, check the assertion actually depends on the change.
- **Level and place:** unit, integration or end-to-end as the repo does it for similar code; location and naming follow the repo.
- **Test quality:** tests assert behavior, not implementation; no sleeps or dependence on time or order; the goal's edge and error cases are covered.
- **Changed existing tests:** a deleted test or a weakened assertion needs a reason. Without one, it's harmful.

### Security

Up to three passes, all defined in [references/security.md](references/security.md):

- **Endpoints:** every entry point the diff adds or changes (HTTP routes, GraphQL resolvers, gRPC methods, webhooks, message consumers, CLI commands, scheduled jobs) gets an authentication, authorization, input and exposure check against the OWASP API Security Top 10.
- **Vectors:** the whole diff, endpoints or not, scanned against the OWASP Top 10 categories, plus the dependency audit for new or bumped packages.
- **MCP:** when the diff adds or changes an MCP server, client or MCP configuration, its tools and context checked against the OWASP MCP Top 10.

Every finding carries its OWASP ID (e.g. `A01`, `API1`, `MCP03`). A reachable injection, missing authorization or exposed secret is a blocker.

## 5. Merge the findings

- Combine findings from all aspects and drop duplicates (same change, same problem).
- Check every blocker and should-fix against the code yourself. Drop what doesn't hold.
- Let quality, dependency, test and security findings inform the scope verdicts: a required change built by duplicating a helper stays required, with a fix; an optional change that opens an attack vector becomes harmful.
- Check that what stays is correct. A minimal change that gives wrong answers is not done. For a deeper bug hunt, suggest `/code-review`.

## 6. Report

```markdown
**Goal:** <one sentence>
**Scope:** <minimal | not minimal: about N of M changes can go>
**Codebase:** <improves | neutral | degrades>: <one sentence why>
**Tests:** <pass | fail>; <covered | N behavior changes untested>
**Security:** <no new attack surface | N endpoints checked>; <no findings | N findings, M blockers>

| Change | Verdict | Why |
|---|---|---|
| C1 ... | Required / Optional / Not needed / Harmful | ... |

## Endpoints
| Endpoint | AuthN | AuthZ | Input | Exposure |
|---|---|---|---|---|
| POST /orders/{id}/cancel | ok | missing ownership check (API1) | ok | ok |

## Fix before merge
- **F1** <file:line>: <what's wrong>. <concrete fix>

## Proposed minimal change
<what stays, what is reverted, what moves to a follow-up MR/PR>

## Follow-ups
- **F2** <file:line or C<n>>: <optional change, unrelated cleanup for its own MR/PR, or nit>

## Decisions for you
<only the trade-offs the maintainer or user has to make, each with a recommendation>
```

Keep it short: the tables and the fix list carry the review. Prefix security findings with their OWASP ID. Leave out empty sections. Number findings across both lists (F1, F2, ...) so the user can pick them in step 7. If everything is required, clean, tested and secure, say so in one line and stop.

If the review is of an MR/PR, end with one line: "Reply with the findings to post on the MR/PR as draft comments, e.g. F1, F3."

## 7. Post to the forge (only when the user asks)

The user picks the findings; you never post on your own initiative. Comments go out as **drafts**: only the user sees them until they submit the review on the forge, so the final call stays with them.

1. **Detect the forge** from `git remote get-url origin`: a `github.com` host is GitHub, a `gitlab` host is GitLab. For a self-hosted host that names neither, use whichever of `gh auth status` and `glab auth status` succeeds for that host; if both or neither do, ask. Then read the matching reference in full: [references/gitlab.md](references/gitlab.md) (`glab`) or [references/github.md](references/github.md) (`gh`). It gives the commands for each step below.
2. **Find the MR/PR** and its base and head commits. If the head isn't the commit you reviewed, tell the user and ask whether to continue: line numbers may have moved.
3. **Skip duplicates:** read the existing threads and drafts. Drop picked findings that one already raises, and say which.
4. **Draft the comments:** one per finding, written as the reviewer: what's wrong, why it matters, the concrete fix (a suggestion block when it's a small replacement). No review jargon (no C/F numbers, no "aspect", no verdict words); an OWASP ID stays, it tells the author where to read up. No AI label unless the user asks for one. Show all drafts and where each will land, and wait for the user's go-ahead or edits.
5. **Choose the position** from the diff between base and head:
   - added line: the new file's path and line,
   - removed line: the old file's path and line,
   - unchanged line inside a hunk: the new file's line (GitLab wants both old and new),
   - anything else (code outside the diff, a finding about the MR/PR as a whole): no position, a general comment that names the file:line in its text.
6. **Post** the drafts as the reference shows. If the forge rejects a position, post that one again as a general comment with the file:line in the text.
7. **Report:** what was posted inline, what went in as a general comment, what was skipped, and the MR/PR link. Remind the user to submit the review on the forge. Do not submit it yourself unless the user asks.
