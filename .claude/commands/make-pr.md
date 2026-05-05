Create or update a pull request for the current branch using the project's title/description style.

## Title format

`<Type>: <lowercase imperative description>`

-   `<Type>` is capitalized: `Feat`, `Fix`, `Refactor`, `Chore`, `Docs`, `Style`, `Perf`, `Test`, `Build`, `CI`, `Revert`
-   No scope, no Jira ticket
-   Description is short (under ~70 chars) and reads as an imperative ("add X", "fix Y", "rename Z")

Examples (matching the style of PR #735 "Feat: add voice messages"):

-   `Feat: add voice messages`
-   `Fix: handle null sender in message thread`
-   `Refactor: split alert context into hooks`

## Body format

```
<One-line summary of what changed and why.>

- <Specific change>
- <Specific change>
- <Specific change>

<Optional `Also bundled:` line for unrelated work picked up in the branch, or a `Notes:` line for caveats reviewers should know.>
```

Rules:

-   Bullets must be concrete (mention the actual mechanism/file/behavior, not vague summaries).
-   No checklist, no testing template, no Jira section.
-   Keep the whole body shorter than the diff justifies — the reader should learn from this what they can't learn by reading the commits.

## Process

1. Run `git log develop..HEAD --oneline` and `git diff develop...HEAD --stat` to see the full scope of the branch.
2. Draft the title and body from the commit history and the diff.
3. Ensure the branch is pushed to remote (run `/ssh-load` first if needed, then `git push -u origin HEAD`).
4. Check whether a PR for the current branch already exists: `gh pr view --json number,url`.
    - If it exists: update it with `gh pr edit <num> --title "..." --body "$(cat <<'EOF' ... EOF)"`.
    - If it does not: create it with `gh pr create --title "..." --body "$(cat <<'EOF' ... EOF)"`.
5. Return the PR URL to the user. If the user wants a Copilot review, tell them to click "Re-request review" next to Copilot in the PR UI — you can't trigger it from here.
