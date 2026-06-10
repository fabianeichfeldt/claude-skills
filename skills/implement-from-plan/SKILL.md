---
name: implement-from-plan
description: Reads a markdown plan file and executes it step by step, cross-checking current code state, marking steps done as it goes, and running tests after each meaningful change. Use when the user says "implement the plan in X.md", "follow the plan", "let's implement now the plan of X.md", or points at a docs/ or plan file to execute.
---

# Implement From Plan

## Quick start

If the user named a plan file, read it immediately. If not, ask:
> "Which plan file should I implement? (e.g. `docs/gpx-segment-editor.md`)"

## Workflow

### 1. Read and parse the plan
- Read the plan file in full.
- Identify every discrete step (numbered list, `[ ]` checkboxes, `##` sections, or clearly bounded paragraphs).
- Print a one-line summary of each step and its status: **done**, **partial**, or **todo** — based on a quick scan of the codebase (grep for key symbols, check file existence).

### 2. Confirm before starting
- If any steps are already done, list them and skip them.
- If the plan is ambiguous or contradicts current code, flag it and ask before proceeding.
- Otherwise confirm: "Ready to implement N steps. Starting with step 1."

### 3. Execute steps sequentially

**≤ 6 todo steps — implement inline** (main agent retains accumulated context):

For each **todo** step:
1. Implement the change (edit files, create files, etc.).
2. After every step that touches logic or adds a function: run `npm test` (or the project's unit test command).
3. After every 3+ file changes: run the full test suite including e2e if available.
4. Mark the step done in your running output: `✓ Step N complete`.
5. If a step fails tests, stop and report — do not continue to the next step.

**> 6 todo steps — spawn one subagent** to own the full implementation:

Write a self-contained prompt that includes:
- The full text of the plan file
- The list of steps already done (from step 1 scan) so the agent skips them
- The test commands to run (`npm test`, `npm run test:e2e`, etc.)
- Instruction to mark checkboxes `- [x]` as steps complete and stop on test failure

Spawn via `Agent` tool (general-purpose). When it returns, report its summary to the user.

### 4. Finish
- Report: total steps done, any skipped (already done), any blocked.
- If all steps are done: suggest removing or archiving the plan file.

## Key rules
- Never skip test runs after logic changes — the plan file is a spec and tests are the verifier.
- If the plan file has checkboxes (`- [ ]`), update them to `- [x]` as steps complete.
- If a step is unclear, implement the most literal reading and note the assumption.
- Do not implement steps beyond what is written in the plan — scope creep is the enemy.
