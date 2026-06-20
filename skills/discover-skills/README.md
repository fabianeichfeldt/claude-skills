# discover-skills

A [Claude Code](https://claude.ai/code) skill that mines your conversation history to surface recurring task patterns and recommends new custom skills worth extracting. Also audits your existing skill library for unused or overlapping entries.

## Install

```sh
npx skills add fabianeichfeldt/claude-skills --skill discover-skills
```

## What it does

1. Collects all user messages from the last 14 days (configurable) across all projects
2. Inventories every installed skill — custom local, plugin, and system skills
3. Clusters recurring patterns by multi-session recurrence and iteration count
4. Gaps analysis: checks which patterns are already covered by existing skills (and whether those skills are actually being invoked)
5. Flags unused skills and near-duplicate skills worth merging
6. Prints a ranked report of new skill candidates with session counts and project coverage
7. Optionally drafts the skill file for any candidate you pick

## Usage

> "Find automation opportunities in my workflow"
> "What skills should I create?"
> "Run a weekly skill discovery session"
> "Audit my workflow for the last month"

## Output format

```
## Skill Discovery Report — Jun 5–19 2026

### New Skill Candidates (ranked by recurrence)
| Rank | Suggested Name     | Pattern               | Sessions | Projects |
|------|--------------------|-----------------------|----------|----------|
| 1    | write-pipeline-yaml| Write CI pipeline YAML| 6        | 3        |

### Existing Skills Audit
| Skill              | Source | Status              |
|--------------------|--------|---------------------|
| implement-from-plan| custom | Active              |
| old-debug-helper   | custom | Unused (0 invocations)|

### Top 3 Recommendations
1. **`write-pipeline-yaml`** — appeared in 6 sessions across 3 projects, always manual
```

## Why

Skills pay off when they encode something you do repeatedly. But it's hard to know what to extract until you look at the pattern. This skill does that analysis for you on a cadence, so your skill library grows with your actual workflow rather than with good intentions.
