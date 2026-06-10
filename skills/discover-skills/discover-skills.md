---
name: discover-skills
description: Mine your Claude conversation history to surface recurring task patterns and recommend new custom skills to extract. Use when you want to audit your workflow, find automation opportunities, or run a weekly skill-discovery session.
---

Scan recent conversation history across all projects, inventory existing skills, and produce a ranked list of new skill candidates with an audit of the current skill library.

## Procedure

### Step 1 — Determine analysis window

Default: last 14 days. If the user specifies a different window ("last month", "last 3 weeks"), compute accordingly.

Run this to collect all user messages from files modified within the window:

```python
import json, os, time

DAYS = 14
cutoff = time.time() - (DAYS * 86400)
projects_dir = os.path.expanduser("~/.claude/projects")
messages = []

for root, dirs, files in os.walk(projects_dir):
    for fname in files:
        if not fname.endswith(".jsonl"):
            continue
        fpath = os.path.join(root, fname)
        if os.path.getmtime(fpath) < cutoff:
            continue
        project = root.replace(projects_dir + "/", "").split("/")[0]
        with open(fpath) as f:
            for line in f:
                try:
                    obj = json.loads(line)
                    msg = obj.get("message", {})
                    if msg.get("role") != "user":
                        continue
                    content = msg.get("content", "")
                    text = ""
                    if isinstance(content, list):
                        for c in content:
                            if isinstance(c, dict) and c.get("type") == "text":
                                text = c["text"].strip()
                                break
                    elif isinstance(content, str):
                        text = content.strip()
                    if text and len(text) > 40 and not text.startswith("<"):
                        messages.append({"project": project, "text": text[:400]})
                except:
                    pass

print(f"Collected {len(messages)} user messages across {len(set(m['project'] for m in messages))} projects")
for m in messages:
    print(f"[{m['project']}] {m['text'][:200]}")
```

### Step 2 — Inventory existing skills

Collect all skills from three sources:

1. **Custom local skills**: `~/.claude/skills/` — list `.md` files and subdirectory `SKILL.md` files. For each, read the `description:` frontmatter line.
2. **Installed plugin skills**: read `~/.claude/settings.json` for `enabledPlugins`. Check `~/.claude/plugins/cache/` for each plugin's skill list.
3. **System skills**: read the skills listed in the `<system-reminder>` tag visible in the current session context.

Build a flat list: `{ name, description, source }` for every skill.

### Step 3 — Cluster recurring patterns

Group the user messages by recurring theme. Strong signals:

- **Multi-session recurrence**: same type of request appears in 2+ separate JSONL files — strongest signal
- **High iteration count**: many back-and-forth turns on the same sub-task within one session
- **Repeated action + object combos**: e.g. `(debug, flaky test)`, `(mock, backend service)`, `(convert, e2e to component)`, `(update, CLAUDE.md)`, `(write, pipeline yaml)`

For each cluster, note:
- How many distinct sessions/files it appears in
- Which projects it spans
- Whether it's self-contained (could run without heavy context)

### Step 4 — Gap analysis

For each cluster:
1. Does an existing skill already cover this? Match by name and description.
2. If yes — is it being invoked (look for `<command-name>/skill-name</command-name>` in history) or is the user re-prompting manually every time?
3. If no — frequency > 2 sessions is a strong extraction candidate.

Also flag:
- **Overlapping skills**: two skills with near-identical descriptions → merge candidate
- **Unused skills**: in `~/.claude/skills/` but no invocation found in recent history

### Step 5 — Output report

Print a concise markdown report:

```
## Skill Discovery Report — <date range>

### New Skill Candidates (ranked by recurrence)

| Rank | Suggested Name | Pattern | Sessions | Projects |
|------|---------------|---------|----------|----------|
| 1    | ...           | ...     | N        | ...      |

### Existing Skills Audit

| Skill | Source | Status |
|-------|--------|--------|
| ...   | custom / plugin | Active / Unused / Overlaps with X |

### Top 3 Recommendations

1. **`skill-name`** — one-sentence justification
2. ...
3. ...
```

End with: **"Which of these would you like me to draft as a skill file now?"**

### Step 6 — Optional: save report to memory

If the user agrees, save the report to `~/.claude/projects/` memory so future runs can compare against previous findings and track which candidates have been acted on.
