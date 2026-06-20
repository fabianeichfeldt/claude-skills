# audit-corrections

A [Claude Code](https://claude.ai/code) skill that scans your conversation history to find recurring correction patterns and surfaces what should be added to your `CLAUDE.md` files — so you stop repeating yourself across sessions.

## Install

```sh
npx skills add fabianeichfeldt/claude-skills --skill audit-corrections
```

## What it does

1. Reads all `.jsonl` conversation logs under `~/.claude/projects/`
2. Extracts user turns containing corrections, pushbacks, and explicit rules (signals like "no", "don't", "actually", "instead", "stop doing", "always", "never")
3. Groups findings by theme and frequency
4. Cross-checks against existing `CLAUDE.md` files to skip patterns already documented
5. Presents new patterns with suggested rule text and the target file to add them to
6. Optionally writes the rules directly into the appropriate `CLAUDE.md`

## Usage

Run it any time you want to audit what Claude has been getting wrong repeatedly:

> "Audit past sessions for learnings"
> "Improve my CLAUDE.md coverage"
> "Run a hygiene check"
> "audit-corrections my-project" — scopes the scan to one project

## Why

CLAUDE.md is only as good as what's in it. Most teams discover what belongs there reactively — after Claude makes the same mistake three times. This skill flips that by mining your actual session history to tell you exactly what's missing.
