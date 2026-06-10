---
name: audit-corrections
description: Scan all Claude conversation logs in ~/.claude/projects/ to find recurring correction patterns (user pushbacks, adaptations, repeated instructions) and suggest what should be added to CLAUDE.md files. Use when the user wants to audit past sessions for learnings, improve CLAUDE.md coverage, or run a periodic hygiene check.
---

Scan all conversation transcript files in `~/.claude/projects/` to find recurring patterns of corrections, adaptations, and user pushbacks. The goal is to surface what should be documented in CLAUDE.md files so the user does not need to repeat the same guidance.

## Step 1 — Collect all transcripts

Find every `.jsonl` file under `~/.claude/projects/`. These are conversation logs; each line is a JSON object with a `role` field (`user` or `assistant`) and content.

Spawn a subagent (subagent_type: general-purpose) to read the files and extract the raw corrections. Hand it this task:

> Read every .jsonl file under `~/.claude/projects/`. For each file, extract user turns that contain corrections, pushbacks, or explicit instructions — look for signals like "no", "don't", "actually", "instead", "stop doing", "not like that", "I want", "always", "never", "prefer", "rename", "rework", "change back". Also extract explicit rules the user stated. Return a structured list: project → file → list of extracted user statements with brief context.

## Step 2 — Identify recurring patterns

From the raw list, group by theme and count frequency (how many separate sessions the pattern appeared in). A pattern is worth surfacing only if it:
- Appeared in 2+ separate sessions, **or**
- Was an explicit rule stated verbatim by the user, **or**
- Caused multiple back-and-forth correction rounds in one session

Themes to look for:
- Code style / naming / formatting corrections
- Architecture / layering violations
- Over-engineering or generalization that was rolled back
- Scope creep (Claude doing work that belongs elsewhere)
- Workflow rules (test-first, lint gates, compile checks)
- Communication preferences (ask vs. assume, not suggesting "move on")
- Documentation / diagram format preferences
- Tool or command preferences (docker, package managers, etc.)

## Step 3 — Check against existing CLAUDE.md files

For each pattern found, check whether it is already captured in a CLAUDE.md in the relevant project directory or in `~/.claude/CLAUDE.md`. If it is already there, skip it.

List the existing CLAUDE.md files by running: `find ~/src -name "CLAUDE.md"` and `cat ~/.claude/CLAUDE.md`.

## Step 4 — Present findings and offer to apply

Present the results in this format:

### Already documented — skip
(brief list of patterns already in a CLAUDE.md)

### New patterns — candidates for CLAUDE.md

For each new pattern:
- **Pattern**: one-line description
- **Evidence**: which projects/sessions, approximate frequency
- **Suggested rule**: the exact text to add to CLAUDE.md
- **Target file**: which CLAUDE.md it belongs in (project-specific or global `~/.claude/CLAUDE.md`)

Then ask the user which patterns to apply, and write them into the appropriate CLAUDE.md files.

## Scope

If the user invokes this skill with a specific project name (e.g. `/audit-corrections project-name`), limit the scan to that project's logs only.
If no argument is given, scan all projects.
