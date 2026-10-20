# Posting to a GitHub PR

Commands for step 7 of the maintainer review on GitHub, with `gh`. GitHub's drafts are a **pending review**: a review created without an `event` stays visible only to its author until they click "Submit review". Each user can hold one pending review per PR, so all findings go into one review.

`gh api` fills `{owner}` and `{repo}` from the current repo's remote.

## Find the PR

```bash
gh pr list --head <branch> --state open --json number,url
gh pr view <number> --json number,url,headRefOid
```

`headRefOid` is the commit to compare against the one you reviewed.

## Existing threads and reviews

```bash
gh api --paginate "repos/{owner}/{repo}/pulls/<number>/comments"   # inline threads
gh api --paginate "repos/{owner}/{repo}/issues/<number>/comments"  # general comments
gh api --paginate "repos/{owner}/{repo}/pulls/<number>/reviews"    # review bodies; state PENDING is yours
```

If you already have a `PENDING` review, read its comments (`.../reviews/<review_id>/comments`) for duplicates and add to it as shown under "Add to an existing pending review" instead of creating a second one.

## Position

Lines come from `gh pr diff <number>`, and must sit inside one of its hunks:

- added line, or unchanged line inside a hunk: `path`, `line` (the new file's line), `side: "RIGHT"`,
- removed line: `path`, `line` (the old file's line), `side: "LEFT"`.

## Post

All findings in one pending review, with the JSON body in a file. General comments (no position) go in the review's `body`, each naming its file:line. Leave out `event`: that is what keeps the review pending.

```bash
gh api --method POST "repos/{owner}/{repo}/pulls/<number>/reviews" --input review.json
```

```json
{"commit_id": "<headRefOid>",
 "body": "<general comments, or empty>",
 "comments": [
   {"path": "<path>", "line": 42, "side": "RIGHT", "body": "<comment>"}
 ]}
```

GitHub rejects the whole review (HTTP 422) if any one position is invalid. Check each against the hunks of `gh pr diff`, move the doubtful ones into `body` with their file:line, and post again.

## Add to an existing pending review

Take the review's `node_id` from the reviews listing, then add one thread per finding:

```bash
gh api graphql -f query='
  mutation($review: ID!, $path: String!, $line: Int!, $side: DiffSide!, $body: String!) {
    addPullRequestReviewThread(input: {pullRequestReviewId: $review, path: $path,
      line: $line, side: $side, body: $body}) { thread { id } }
  }' -f review=<node_id> -f path=<path> -F line=42 -f side=RIGHT -f body='<comment>'
```

A thread that fails goes into the review body instead: `gh api --method PUT "repos/{owner}/{repo}/pulls/<number>/reviews/<review_id>" -f body='<existing body + comment>'`.
