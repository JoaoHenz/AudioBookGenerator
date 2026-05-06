Iterate on the latest review feedback for the PR on the current branch. Address each unresolved comment, push fixes, reply, and resolve threads.

## Process

1. Find the PR for the current branch (`gh pr view --json number,url,headRefName`).
2. List unresolved review threads via GraphQL:
    ```
    gh api graphql -f query='query { repository(owner:"<owner>",name:"<repo>") { pullRequest(number:<num>) { reviewThreads(first:100) { nodes { id isResolved comments(first:50) { nodes { databaseId author { login } body path line } } } } } } }'
    ```
    Filter to `isResolved == false`.
3. For each unresolved thread, decide: **take**, **partially take**, or **skip with motivation**. Lean toward taking — skip only with a clear, defensible reason. State each verdict to the user before implementing.
4. For each comment to take: implement the fix in code, then create a **separate commit per logical fix** using the conventional-commits format (see `/commit`). Group fixes that are the same kind of change in different files (e.g. an a11y attribute on two switches) into one commit; otherwise keep one commit per comment.
5. Run `/ssh-load`, then `git push`.
6. Reply to each thread by posting to `repos/<owner>/<repo>/pulls/<num>/comments/<comment-id>/replies`:
    - `Implemented.` for taken comments.
    - A short motivation paragraph for skipped ones.
    - Always end with `-by claude-code` on its own line.
7. Resolve each thread after replying via the GraphQL mutation:
    ```
    gh api graphql -f query='mutation { resolveReviewThread(input:{threadId:"<id>"}) { thread { isResolved } } }'
    ```
    Reply and resolve as a pair — never reply without resolving (Copilot does not auto-resolve).
8. Tell the user the iteration is done and remind them that if they want a fresh Copilot review they have to click "Re-request review" next to Copilot in the PR UI — you cannot trigger it from here.
