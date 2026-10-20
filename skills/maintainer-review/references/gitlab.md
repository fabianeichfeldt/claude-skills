# Posting to a GitLab MR

Commands for step 7 of the maintainer review on GitLab, with `glab`. GitLab's drafts are **draft notes**: only the author sees them until they click "Submit review".

## Find the MR

```bash
glab api "projects/:id/merge_requests?source_branch=<branch>&state=opened"
```

Or use the MR number you were given: `glab api "projects/:id/merge_requests/<iid>"`. Read its `iid`, `web_url` and `diff_refs` (`base_sha`, `start_sha`, `head_sha`). `head_sha` is the commit to compare against the one you reviewed.

## Existing threads and drafts

```bash
glab api --paginate "projects/:id/merge_requests/<iid>/discussions"
glab api --paginate "projects/:id/merge_requests/<iid>/draft_notes"
```

## Position

Lines come from the diff between `base_sha` and `head_sha`:

- added line: `new_path` and `new_line`,
- removed line: `old_path` and `old_line`,
- unchanged line inside a hunk: both `old_line` and `new_line`.

## Post

One draft note per finding, with the JSON body in a file:

```bash
glab api --method POST "projects/:id/merge_requests/<iid>/draft_notes" \
  -H "Content-Type: application/json" --input note.json
```

```json
{"note": "<comment>", "position": {"position_type": "text",
  "base_sha": "...", "start_sha": "...", "head_sha": "...",
  "old_path": "<path>", "new_path": "<path>", "new_line": 42}}
```

A general comment is the same call with only `note`. If GitLab rejects a position, post that note again without `position`, with the file:line in the text.
