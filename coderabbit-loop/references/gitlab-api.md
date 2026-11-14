# GitLab API Reference

GitLab REST calls for the coderabbit-loop workflow, using `glab api`. `glab api` resolves `:fullpath`
to the URL-encoded project path from the local git remote.

CodeRabbit on GitLab posts notes and discussions on the MR; it doesn't run a pipeline job. Its bot
username depends on the installation (usually `coderabbitai`). Check the first CodeRabbit note on
the MR to confirm it.

## Fetch MR details

```bash
glab mr view <MR_IID> --output json
```

Key fields: `iid` (use this, not `id`), `source_branch`, `sha` (head commit), `draft`.

## Ask for a review

Only when automatic reviews are paused or a rate-limit wait has passed. A push triggers a review by
itself.

```bash
glab mr note <MR_IID> --message "@coderabbitai review"
```

## Fetch CodeRabbit's notes (summary, review results, rate-limit notices)

```bash
glab api "projects/:fullpath/merge_requests/<MR_IID>/notes?per_page=100&sort=desc&order_by=updated_at" \
  | jq '[.[] | select(.author.username == "CODERABBIT_BOT_USERNAME")]'
```

Look in `body` for `Actionable comments posted: N`, `No actionable comments were generated in the
recent review.`, and `Review limit reached` with `Next included review available in N minutes.`

## Fetch unresolved discussions (inline comments)

```bash
glab api "projects/:fullpath/merge_requests/<MR_IID>/discussions?per_page=100"
```

Paginate with `&page=2`, etc. until the response array is shorter than `per_page`.

```bash
jq '[.[] | select(.resolved == false and (.notes[0].type == "DiffNote") and (.notes[0].author.username == "CODERABBIT_BOT_USERNAME"))]'
```

Each discussion has `id` (use it to resolve), `notes[0].body`, and `notes[0].position.new_path`.

## Resolve a discussion

```bash
glab api --method PUT \
  "projects/:fullpath/merge_requests/<MR_IID>/discussions/<DISCUSSION_ID>" \
  --field resolved=true
```

GitLab has no batch resolution. Send one PUT per discussion.
