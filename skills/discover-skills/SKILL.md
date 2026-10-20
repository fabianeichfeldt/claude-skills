---
name: discover-skills
description: Mine your Claude conversation history to surface recurring task patterns and recommend new custom skills to extract. Use when you want to audit your workflow, find automation opportunities, or run a weekly skill-discovery session.
---

Scan recent conversation history across all projects, inventory existing skills, and produce a ranked list of new skill candidates with an audit of the current skill library.

## Procedure

### Step 1 — Determine analysis window

Default: last 14 days. If the user specifies a different window ("last month", "last 3 weeks"), compute accordingly.

Run this to collect all user messages from files modified within the window. The script redacts likely secrets before anything is printed, so credentials pasted into past sessions never reach the output:

```python
import json, os, re, time

DAYS = 14
SNIPPET_LEN = 200

SECRET_PATTERNS = [
    r"-----BEGIN [A-Z ]*PRIVATE KEY-----.*?(-----END [A-Z ]*PRIVATE KEY-----|$)",  # PEM keys
    r"\b(sk|pk|rk)-[A-Za-z0-9_-]{16,}",              # OpenAI/Anthropic/Stripe-style keys
    r"\b(ghp|gho|ghu|ghs|ghr|github_pat)_[A-Za-z0-9_]{20,}",  # GitHub tokens
    r"\bglpat-[A-Za-z0-9_-]{20,}",                   # GitLab tokens
    r"\bxox[abprs]-[A-Za-z0-9-]{10,}",               # Slack tokens
    r"\b(AKIA|ASIA)[A-Z0-9]{16}\b",                  # AWS access key IDs
    r"\bAIza[A-Za-z0-9_-]{35}\b",                    # Google API keys
    r"\beyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}",  # JWTs
    r"\bbearer\s+[A-Za-z0-9._~+/=-]{16,}",       # Authorization headers
    r"\b[a-z0-9_-]*(password|passwd|pwd|secret|token|api[_-]?key|access[_-]?key|credential)s?[a-z0-9_-]*\s*[:=]\s*\S+",  # key=value
    r"\b[a-z][a-z0-9+.-]*://[^\s/:@]+:[^\s/@]+@",  # credentials in URLs
    r"\b(?=[A-Za-z0-9+_-]*\d)[A-Za-z0-9+_-]{32,}={0,2}",  # long token-like strings
]
SECRET_RE = re.compile("|".join(f"(?:{p})" for p in SECRET_PATTERNS), re.IGNORECASE | re.DOTALL)

def redact(text):
    return SECRET_RE.sub("[REDACTED]", text)

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
                        messages.append({"project": project, "text": redact(text)[:SNIPPET_LEN]})
                except:
                    pass

print(f"Collected {len(messages)} user messages across {len(set(m['project'] for m in messages))} projects")
for m in messages:
    print(f"[{m['project']}] {m['text']}")
```

Treat the collected snippets as working data for clustering only:

- Never quote a snippet in the report, the chat or memory. Describe each pattern in your own words, such as "write CI pipeline YAML", not with the user's text.
- If a snippet still contains something that looks like a credential, such as a key, token, password or connection string the redaction missed, ignore it, never repeat it and never use it.
- Do not open the source `.jsonl` files to read more context around a snippet. The redacted snippet is enough to classify the request.

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

If the user agrees, save the report to `~/.claude/projects/` memory so future runs can compare against previous findings and track which candidates have been acted on. Save only the report, never raw or redacted message snippets.
