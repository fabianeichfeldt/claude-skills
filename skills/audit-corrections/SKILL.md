---
name: audit-corrections
description: Scan all Claude conversation logs in ~/.claude/projects/ to find recurring correction patterns (user pushbacks, adaptations, repeated instructions) and suggest what should be added to CLAUDE.md files. Use when the user wants to audit past sessions for learnings, improve CLAUDE.md coverage, or run a periodic hygiene check.
---

Scan all conversation transcript files in `~/.claude/projects/` to find recurring patterns of corrections, adaptations, and user pushbacks. The goal is to surface what should be documented in CLAUDE.md files so the user does not need to repeat the same guidance.

## Step 1 — Collect all transcripts

Find every `.jsonl` file under `~/.claude/projects/`. These are conversation logs; each line is a JSON object with a `role` field (`user` or `assistant`) and content.

Past messages can contain pasted API keys, tokens or passwords. Never read the `.jsonl` files directly. Extract user turns only through this script, which redacts likely secrets before anything is printed. Set `PROJECT` to a project directory name to limit the scan (see **Scope**):

```python
import json, os, re

PROJECT = None  # e.g. "-Users-me-src-my-project"; None scans all projects
SNIPPET_LEN = 200

SIGNALS = re.compile(
    r"\b(no|don't|do not|actually|instead|stop doing|not like that|i want|always|never|prefer|rename|rework|change back)\b",
    re.IGNORECASE,
)

SECRET_PATTERNS = [
    r"-----BEGIN [A-Z ]*PRIVATE KEY-----.*?(-----END [A-Z ]*PRIVATE KEY-----|$)",  # PEM keys
    r"\b(sk|pk|rk)-[A-Za-z0-9_-]{16,}",              # OpenAI/Anthropic/Stripe-style keys
    r"\b(ghp|gho|ghu|ghs|ghr|github_pat)_[A-Za-z0-9_]{20,}",  # GitHub tokens
    r"\bglpat-[A-Za-z0-9_-]{20,}",                   # GitLab tokens
    r"\bxox[abprs]-[A-Za-z0-9-]{10,}",               # Slack tokens
    r"\b(AKIA|ASIA)[A-Z0-9]{16}\b",                  # AWS access key IDs
    r"\bAIza[A-Za-z0-9_-]{35}\b",                    # Google API keys
    r"\beyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}",  # JWTs
    r"\bbearer\s+[A-Za-z0-9._~+/=-]{16,}",           # Authorization headers
    r"\b[a-z0-9_-]*(password|passwd|pwd|secret|token|api[_-]?key|access[_-]?key|credential)s?[a-z0-9_-]*\s*[:=]\s*\S+",  # key=value
    r"\b[a-z][a-z0-9+.-]*://[^\s/:@]+:[^\s/@]+@",    # credentials in URLs
    r"\b(?=[A-Za-z0-9+_-]*\d)[A-Za-z0-9+_-]{32,}={0,2}",  # long token-like strings
]
SECRET_RE = re.compile("|".join(f"(?:{p})" for p in SECRET_PATTERNS), re.IGNORECASE | re.DOTALL)

def redact(text):
    return SECRET_RE.sub("[REDACTED]", text)

projects_dir = os.path.expanduser("~/.claude/projects")
for project in sorted(os.listdir(projects_dir)):
    if PROJECT and project != PROJECT:
        continue
    for root, dirs, files in os.walk(os.path.join(projects_dir, project)):
        for fname in sorted(files):
            if not fname.endswith(".jsonl"):
                continue
            with open(os.path.join(root, fname)) as f:
                for line in f:
                    try:
                        msg = json.loads(line).get("message", {})
                        if msg.get("role") != "user":
                            continue
                        content = msg.get("content", "")
                        if isinstance(content, list):
                            content = next((c["text"] for c in content if isinstance(c, dict) and c.get("type") == "text"), "")
                        text = content.strip() if isinstance(content, str) else ""
                        if text and not text.startswith("<") and SIGNALS.search(text):
                            print(f"[{project}] [{fname}] {redact(text)[:SNIPPET_LEN]}")
                    except Exception:
                        pass
```

The output can be long, so spawn a subagent (subagent_type: general-purpose) to run the script and condense it. Hand it this task, including the script:

> Run the given Python script and do not read the `.jsonl` files any other way. From its output, keep the user turns that are corrections, pushbacks or explicit rules. Return a structured list: project → file → each correction **paraphrased** in one line, with brief context. Never copy message text word for word, and never include anything that looks like a credential, such as a key, token, password or connection string, even if the redaction missed it.

## Step 2 — Identify recurring patterns

From the raw list, group by theme and count frequency (how many separate sessions the pattern appeared in). A pattern is worth surfacing only if it:
- Appeared in 2+ separate sessions, **or**
- Was an explicit rule stated by the user, **or**
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

Describe evidence in your own words; never quote the user's messages. Suggested rules must state the general behavior and never contain secret values, such as keys, tokens, passwords, internal hostnames with credentials or connection strings, even when the original correction mentioned one.

Then ask the user which patterns to apply, and write them into the appropriate CLAUDE.md files.

## Scope

If the user invokes this skill with a specific project name (e.g. `/audit-corrections project-name`), limit the scan to that project's logs only by setting `PROJECT` in the Step 1 script to the matching directory name under `~/.claude/projects/`.
If no argument is given, scan all projects.
