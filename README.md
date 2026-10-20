# claude-skills

A collection of reusable [Claude Code](https://claude.ai/code) skills that extend your AI assistant with structured workflows for common development tasks.

## Install all skills

```sh
npx skills add fabianeichfeldt/claude-skills
```

## Skills

### `implement-from-plan`

Reads a markdown plan file and executes it step by step — cross-checking current code state, running tests after each change, and marking steps complete as it goes. Use it when you have a `docs/*.md` plan and want Claude to follow it precisely without scope creep.

```sh
npx skills add fabianeichfeldt/claude-skills --skill implement-from-plan
```

**Trigger phrases:** "implement the plan in X.md", "follow the plan", "let's implement now the plan of X.md"

---

### `audit-corrections`

Scans all your Claude conversation logs (`~/.claude/projects/`) to find recurring correction patterns — pushbacks, repeated instructions, explicit rules — and suggests what to add to your `CLAUDE.md` files so you never have to give the same guidance twice.

```sh
npx skills add fabianeichfeldt/claude-skills --skill audit-corrections
```

**Trigger phrases:** "audit past sessions for learnings", "improve CLAUDE.md coverage", "run a hygiene check"

---

### `discover-skills`

Mines your Claude conversation history to surface recurring task patterns and recommends new custom skills worth extracting. Inventories your existing skills and flags unused or overlapping ones. Great as a weekly workflow audit.

```sh
npx skills add fabianeichfeldt/claude-skills --skill discover-skills
```

**Trigger phrases:** "find automation opportunities", "what skills should I create", "audit my workflow"

---

### `maintainer-review`

Reviews a branch, diff, GitLab MR or GitHub PR as the repository's maintainer: is it the minimal change for the business goal, does it leave the codebase cleaner, keep dependencies straight, test what it changes, and stay secure? A dedicated security agent checks every new endpoint and scans for open vectors against the OWASP Top 10, the OWASP API Security Top 10 and, when MCP servers are involved, the OWASP MCP Top 10. On request, it posts the findings you pick as draft comments, using `glab` or `gh` depending on the forge.

```sh
npx skills add fabianeichfeldt/claude-skills --skill maintainer-review
```

**Trigger phrases:** "review this MR/PR", "is this change minimal?", "is this secure?", "make it surgical"

---

## Requirements

- [Claude Code](https://claude.ai/code) CLI
- Node.js (for `npx skills`)
- `glab` or `gh` (only for `maintainer-review` to post comments)
