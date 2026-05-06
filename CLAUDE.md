# Project instructions

## Commits

When the user asks to commit (e.g. "commit this", "commit stuff", "commit changes"), always run `/ssh-load` first, then use the `/commit` skill.

## Pull Requests

When the user asks to open/make/fix a PR (e.g. "open a pr", "create pr", "make pr", "fix pr"), always run `/ssh-load` first, then use the `/make-pr` skill.

## PR review iteration

When the user asks to iterate on PR review (e.g. "iterate with copilot on pr", "check copilot review", "check pr review", "address pr feedback"), always run `/ssh-load` first, then use the `/iterate-pr` skill.

## Reviewing PRs

When the user asks you to review a PR (e.g. "review the pr", "review this pr", "do a review", "review pr 42"), use the `/review-pr` skill. No `/ssh-load` needed — the review is posted via the `gh` HTTPS token, not git over SSH.

## Copilot review requests

You cannot request Copilot reviews. The public REST/GraphQL APIs do not expose a working re-request path for Bot reviewers (`gh api .../requested_reviewers` is a no-op once Copilot has reviewed; `requestReviews` GraphQL only accepts user/team IDs). If the user asks for a Copilot review, tell them plainly that you can't do it from here and that they need to click the "Re-request review" arrow next to Copilot in the PR UI themselves. Do not attempt `gh` API or GraphQL workarounds — they don't work.

## PR comment replies

When the user asks you to answer/reply to a comment on a PR:

-   End your reply with the line `-by claude-code` on its own line.
-   After posting the reply, resolve the conversation thread (Copilot does not auto-resolve). Use the GraphQL `resolveReviewThread` mutation via `gh api graphql`, since the REST API does not expose thread resolution.
