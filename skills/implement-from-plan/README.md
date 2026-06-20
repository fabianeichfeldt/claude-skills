# implement-from-plan

A [Claude Code](https://claude.ai/code) skill that reads a markdown plan file and executes it step by step — checking current code state, running tests after each change, and marking steps complete as it goes.

## Install

```sh
npx skills add fabianeichfeldt/claude-skills --skill implement-from-plan
```

## What it does

1. Reads the plan file you point it at
2. Scans the codebase to determine which steps are already done, partial, or todo
3. Executes each pending step sequentially
4. Runs tests after every logic change
5. Marks steps complete in your output (and updates `- [ ]` checkboxes if present)
6. Stops and reports on any test failure — never silently continues

For plans with more than 6 steps it spawns a subagent so the main context stays clean.

## Usage

Just point Claude at your plan file:

> "Implement the plan in docs/gpx-editor.md"
> "Follow the plan"
> "Let's implement now the plan of docs/auth-refactor.md"

## Why

Plan files drift from code when Claude implements them piecemeal across sessions. This skill keeps the plan as the single source of truth and uses tests as the verifier — no half-done steps, no scope creep beyond what's written.
