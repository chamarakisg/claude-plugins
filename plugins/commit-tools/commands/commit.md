---
description: Stage all changes, commit with the given message, and push to the current branch
argument-hint: <commit message>
allowed-tools: Bash(git add:*), Bash(git commit:*), Bash(git push:*), Bash(git rev-parse:*), Bash(git status:*)
---

Commit and push all current changes.

Commit message: $ARGUMENTS

Steps:

1. Stage everything: `git add .`
2. Commit using the message above: `git commit -m "$ARGUMENTS"`
   - If `$ARGUMENTS` is empty, ask the user for a commit message before proceeding.
3. Determine the current branch: `git rev-parse --abbrev-ref HEAD`
4. Push to that branch. Use `-u` so the first push of a new branch also sets upstream:
   `git push -u origin <current-branch>`

Report the resulting commit hash and confirm the push succeeded.
