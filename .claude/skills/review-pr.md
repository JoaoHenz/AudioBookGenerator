Review a pull request like a thoughtful reviewer: read the changes, leave inline comments and suggestions where useful, and submit a single review with a verdict (approve, request changes, or comment).

## Process

1. Resolve the target PR:
    - If the user passed a number (`/review-pr 42`), use it.
    - Otherwise resolve from the current branch: `gh pr view --json number,url,headRefOid,baseRefName,headRefName,author,title,body,files`.
    - If the PR author matches the active `gh` user (`gh api user --jq .login`), tell the user you will not approve your own changes and downgrade the final verdict to `COMMENT` even if nothing is wrong.

2. Fetch context:
    - `gh pr diff <num>` for the unified diff.
    - For each changed file, read the **post-state** from disk (the branch is checked out) so you have full surrounding context, not just the hunk.
    - Skim the PR title and body — the review should call out anywhere the diff diverges from what the description claims.

3. Analyze. Look for, in roughly this priority order:
    - **Bugs**: logic errors, off-by-one, null/undefined handling, wrong conditional, swapped args, broken edge cases.
    - **Security**: injection, secrets in code, auth bypass, unsafe deserialization, path traversal.
    - **Correctness vs description**: missing pieces, scope creep, wrong file touched.
    - **Tests**: new logic without coverage, tests that don't actually assert the new behavior.
    - **Readability**: misleading names, dead code, magic numbers, comments that lie, oversized functions.
    - **Performance**: obvious N+1, redundant work in a hot path. Skip micro-optimization nits.
    - Skip anything ruff/the linter would already flag.

4. Build inline comments — one per concrete issue, anchored on the changed line. When the fix is ≤ a few lines and unambiguous, include a GitHub suggestion block so the author can apply it in one click:
    ````
    ```suggestion
    <fixed line(s) of code>
    ```
    ````
    For larger refactors, describe the change in prose. Don't leave nit-only comments on every minor preference; pick the ones that matter.

5. Pick the verdict:
    - `APPROVE` — no blocking issues; minor nits are fine.
    - `REQUEST_CHANGES` — at least one bug, security issue, or correctness gap that must be fixed before merge.
    - `COMMENT` — observations only, or you're unsure of project conventions, or you authored the PR (see step 1).

6. Write the summary body. One sentence stating the verdict and why, then a short bullet list of the highest-signal findings (group by file if it helps). Keep it under ~200 words. End with `-by claude-code` on its own line.

7. Submit the review in a single API call so inline comments and the verdict post atomically. Build the JSON payload, then:
    ```bash
    gh api -X POST repos/<owner>/<repo>/pulls/<num>/reviews --input - <<'EOF'
    {
      "commit_id": "<headRefOid>",
      "event": "APPROVE | REQUEST_CHANGES | COMMENT",
      "body": "<summary ending with -by claude-code>",
      "comments": [
        {"path": "src/foo.py", "line": 42, "side": "RIGHT", "body": "<comment ending with -by claude-code>"},
        {"path": "src/bar.py", "start_line": 10, "start_side": "RIGHT", "line": 14, "side": "RIGHT", "body": "<multi-line comment>"}
      ]
    }
    EOF
    ```
    - `line` is the line number in the **post-state** file (the new file).
    - For multi-line comments, add `start_line` + `start_side`.
    - Every inline comment body must also end with `-by claude-code` on its own line.

8. Return the review URL (from the API response's `html_url`) to the user. Mention that any conversation threads the author resolves stay resolved; you do not auto-resolve your own threads.

## Notes

- Do not push commits, edit code, or open new PRs from this skill — review only.
- If CI is failing, mention it in the summary but don't block solely on CI; the human/CI gate handles that.
- Don't invent issues to justify `REQUEST_CHANGES`. If the PR is genuinely fine, approve it.
