# GraphQL Queries Reference

GraphQL reports CodeRabbit's author login as `coderabbitai`; REST reports it as `coderabbitai[bot]`.

## Fetch unresolved review threads (paginated)

```graphql
query($cursor: String) {
  repository(owner: "OWNER", name: "REPO") {
    pullRequest(number: PR_NUMBER) {
      reviewThreads(first: 100, after: $cursor) {
        pageInfo { hasNextPage endCursor }
        nodes {
          id
          isResolved
          isOutdated
          comments(first: 3) {
            nodes {
              body
              path
              line
              author { login }
              createdAt
            }
          }
        }
      }
    }
  }
}
```

Keep threads where `isResolved` is false and the first comment's `author.login` is `coderabbitai`.
An `isOutdated` thread sits on code that has since changed. Check whether the change fixed it
before resolving it.

## Reply to a thread before resolving it

```graphql
mutation {
  addPullRequestReviewThreadReply(input: {pullRequestReviewThreadId: "ID1", body: "Not applying: <reason>"}) {
    comment { id }
  }
}
```

## Batch-resolve threads

```graphql
mutation {
  t1: resolveReviewThread(input: {threadId: "ID1"}) { thread { isResolved } }
  t2: resolveReviewThread(input: {threadId: "ID2"}) { thread { isResolved } }
}
```

## Fetch the walkthrough summary, edited in place (REST)

CodeRabbit updates one summary comment on every review, so select by `updated_at`, not `created_at`:

```bash
gh api --paginate "repos/{owner}/{repo}/issues/<PR_NUMBER>/comments?per_page=100" \
  | jq -s 'add
    | map(select(.user.login == "coderabbitai[bot]"))
    | sort_by(.updated_at)
    | last
    | {updated_at, body}'
```

## Fetch the CodeRabbit check run for a commit (REST)

```bash
gh api "repos/{owner}/{repo}/commits/<SHA>/check-runs" \
  --jq '.check_runs[] | select(.name == "CodeRabbit") | {status, conclusion, title: .output.title}'
```

Titles seen: `Finished` (`success`), `Review rate limited` (`neutral`), `Review skipped: <reason>`
(`skipped`).
