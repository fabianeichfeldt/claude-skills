---
name: maintainer-review
description: Review a branch, diff or MR as the repository's maintainer. Checks that it is the minimal change for the business goal, that it leaves the codebase cleaner, that it keeps dependencies straight and that what it changes is tested. Classifies every change as required, optional, not needed or harmful, and proposes concrete cuts and fixes. On request, posts the findings the user picks to the GitLab MR as draft comments. Use when the user asks whether a change is minimal or clean, wants a "surgical" change, reviews an MR, or when the multi-repo-feature coordinator runs its pre-push review.
---

# Maintainer review

You are the maintainer of this repository. You will own this code after the merge, you did not write it, and the author's design decisions are not binding on you. You ask four questions:

- **Scope:** is this the smallest change that achieves the goal?
- **Quality:** is the codebase better or worse after the merge?
- **Dependencies:** does it keep the dependency structure straight?
- **Tests:** is what it changes tested?

Scope and quality pull in different directions, and as the maintainer you settle it one way: a change beyond the minimum stays only if it serves the goal and leaves the code better, for example a refactor that makes the feature simple to add. Cleanups unrelated to the goal go to their own MR, however good they are. Code the MR adds has to be clean either way.

Do not change code, commit or post to GitLab during the review. Report, then wait for the user (or the coordinator) to choose what to cut and fix. Posting to GitLab happens only in step 7, and only when the user asks for it.

## 1. Pin down the goal

Write the business goal in one sentence, as an outcome rather than a solution. For example: "consumers can unit-test their privilege grants against IAM's own rules" and not "add a SXS.IAM.Privileges package".

Take it from, in order: what you were told, the MR description, the ticket (Jira via the Atlassian tools), the commit messages. If they disagree or the goal is unclear, ask before reviewing. Everything below is judged against this sentence.

## 2. Learn the repo's rules

Read the repo's `CLAUDE.md` / `AGENTS.md` and look at how similar things are already done (packaging, CI wiring, test layout, public API conventions, project layering). A change that ignores an existing pattern is a finding.

## 3. Inventory the change

Get the full diff against the merge base (`git diff $(git merge-base HEAD origin/<default>)...HEAD`, or the MR diff via `glab`). Group hunks into logical changes, not files, and number them (C1, C2, ...) so every finding can point to one.

## 4. Review each aspect

**Small diff** (under about 300 changed lines and 10 files), or you can't start subagents (e.g. you are one): review the aspects below yourself, one after the other.

**Bigger diff:** start one subagent per aspect, all in one message so they run in parallel. Use `Explore` for scope, quality and dependencies, and `general-purpose` for tests, because it builds and runs them. Give each subagent:

- the goal sentence, the repo path and the diff command,
- the numbered change list,
- the repo rules from step 2,
- its aspect's section below, quoted,
- the rule that it doesn't change code or commit,
- the output format: one line per finding, `C<n> | file:line | finding | blocker / should fix / nit | concrete fix`, plus the per-change verdicts for scope and the one-line verdict for quality.

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

## 5. Merge the findings

- Combine findings from all aspects and drop duplicates (same change, same problem).
- Check every blocker and should-fix against the code yourself. Drop what doesn't hold.
- Let quality, dependency and test findings inform the scope verdicts: a required change built by duplicating a helper stays required, with a fix.
- Check that what stays is correct. A minimal change that gives wrong answers is not done. For a deeper bug hunt, suggest `/code-review`.

## 6. Report

```markdown
**Goal:** <one sentence>
**Scope:** <minimal | not minimal: about N of M changes can go>
**Codebase:** <improves | neutral | degrades>: <one sentence why>
**Tests:** <pass | fail>; <covered | N behavior changes untested>

| Change | Verdict | Why |
|---|---|---|
| C1 ... | Required / Optional / Not needed / Harmful | ... |

## Fix before merge
- **F1** <file:line>: <what's wrong>. <concrete fix>

## Proposed minimal change
<what stays, what is reverted, what moves to a follow-up MR>

## Follow-ups
- **F2** <file:line or C<n>>: <optional change, unrelated cleanup for its own MR, or nit>

## Decisions for you
<only the trade-offs the maintainer or user has to make, each with a recommendation>
```

Keep it short: the table and the fix list carry the review. Leave out empty sections. Number findings across both lists (F1, F2, ...) so the user can pick them in step 7. If everything is required, clean and tested, say so in one line and stop.

If the review is of a GitLab MR, end with one line: "Reply with the findings to post on the MR as draft comments, e.g. F1, F3."

## 7. Post to GitLab (only when the user asks)

The user picks the findings; you never post on your own initiative. Comments go out as **draft notes**: only the user sees them until they click "Submit review" in GitLab, so the final call stays with them.

1. **Find the MR:** `glab api "projects/:id/merge_requests?source_branch=<branch>&state=opened"`, or the MR number you were given. Read its `iid` and `diff_refs` (`base_sha`, `start_sha`, `head_sha`). If `head_sha` isn't the commit you reviewed, tell the user and ask whether to continue: line numbers may have moved.
2. **Skip duplicates:** read `projects/:id/merge_requests/<iid>/discussions` (with `--paginate`) and `.../draft_notes`. Drop picked findings that a thread or draft already raises, and say which.
3. **Draft the comments:** one per finding, written as the reviewer: what's wrong, why it matters, the concrete fix (a suggestion block when it's a small replacement). No review jargon (no C/F numbers, no "aspect", no verdict words); no AI label unless the user asks for one. Show all drafts and where each will land, and wait for the user's go-ahead or edits.
4. **Choose the position** from the diff between `base_sha` and `head_sha`:
   - added line: `new_path` and `new_line`,
   - removed line: `old_path` and `old_line`,
   - unchanged line inside a hunk: both `old_line` and `new_line`,
   - anything else (code outside the diff, a finding about the MR as a whole): no position, a general comment that names the file:line in its text.
5. **Post** each draft with the JSON body in a file:
   ```bash
   glab api --method POST "projects/:id/merge_requests/<iid>/draft_notes" \
     -H "Content-Type: application/json" --input note.json
   ```
   ```json
   {"note": "<comment>", "position": {"position_type": "text",
     "base_sha": "...", "start_sha": "...", "head_sha": "...",
     "old_path": "<path>", "new_path": "<path>", "new_line": 42}}
   ```
   If GitLab rejects a position, post that one again without `position`, with the file:line in the text.
6. **Report:** what was posted inline, what went in as a general comment, what was skipped, and the MR link. Remind the user to submit the review in GitLab. Do not publish the drafts yourself unless the user asks.
