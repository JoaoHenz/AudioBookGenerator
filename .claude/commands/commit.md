Commit all uncommitted changes using conventional commits format.

## Rules

Always format commit messages as:

```
<type>(<scope>): <description>
```

Allowed types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`, `build`, `revert`, `agent`

Use `agent` for commits concerning AI agent guidelines, skills, memory, or Claude instructions.

Scope is optional. For multi-word scopes, use PascalCase (e.g. `openPr`, `clientLayout`). Examples:

-   `feat: add row click navigation to triggers table`
-   `fix(badge): guard against undefined status`
-   `refactor(table): add onRowClick prop with pointer cursor`
-   `agent(openPr): return Jira ticket link alongside PR URL`

## Commit granularity

Each commit must contain exactly **one logical change**. If you need more than one sentence to describe what a commit does, it should probably be multiple commits. Common splits:

-   Refactors (e.g. changing a component's API) go in their own commit before the feature that uses them
-   Hook/service changes go separate from the page/component that consumes them
-   Bug fixes (e.g. overflow, null guards) are never bundled with features
-   Removing old/stubbed code is part of the commit that replaces it, not a separate one

Commit as you go — don't batch everything at the end.

## Process

1. Run format and lint fix from the repo root before anything else:
    ```
    npx prettier --write .
    npx eslint . --ext .ts,.tsx --fix
    ```
2. Run `git status` and `git diff` to see all changes
3. Identify each logical change in the diff and plan the commit sequence
4. For each logical change: stage only the relevant files, commit with a single-line message
5. Keep going until `git status` shows nothing left to commit
6. Each commit message must match: `^(feat|fix|docs|style|refactor|perf|test|chore|ci|build|revert|agent)(\(\w+\))?:\s.+`
7. Never include `Co-Authored-By` trailers in commit messages
8. **Never pull, rebase, or merge from remote** during the commit process. The remote state is irrelevant — just commit locally.
9. After all commits are done, run `git push`. If the push fails (e.g. diverged remote), don't force-push and don't try to pull/rebase — just let the user know
