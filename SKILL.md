---
name: issue-fixer
description: Fix a GitHub issue end-to-end — investigate the codebase, implement a fix, verify it, and open a pull request. Use when the user gives a GitHub issue URL or number and asks to fix it, resolve it, or send a PR for it (e.g. "fix issue #42", "can you resolve this issue: <url>", "open a PR for this bug report").
---

# issue-fixer

Fix a GitHub issue end-to-end: understand it, implement a fix, verify it, and open a PR.

## Inputs

The user provides a GitHub issue as a URL (`https://github.com/<owner>/<repo>/issues/<n>`) or a `#<n>` plus repo context. If neither the repo nor issue number can be determined, ask.

## Steps

1. **Fetch the issue.** Use the `github` MCP tools (`issue_read` / `get_file_contents` etc.) to pull the issue title, body, labels, and comments. Read the whole thread — reproduction steps and constraints often live in comments, not just the body.
2. **Locate the repo.** If it's not already checked out locally, clone it (or ask the user where their local checkout lives) rather than editing via the API.
3. **Reproduce/understand first.** Before writing a fix, confirm you understand the bug or request — find the relevant code, and where practical reproduce the failure (failing test, repro script, or manual trace).
4. **Implement the smallest correct fix.** Follow the repo's existing conventions. Don't refactor unrelated code.
5. **Verify.** Run the project's tests/build relevant to the change. Add or update a test that would have caught the bug, if the repo has a test suite.
6. **Open a PR.** Create a branch, commit with a message describing the fix and referencing the issue (`Fixes #<n>`), push, and open a pull request via the `github` MCP tools. Summarize what changed and why, and link the issue.

## Guardrails

- Confirm with the user before pushing branches or opening PRs on a repo you haven't touched before in this session — see the project's general risky-action rules.
- If the issue is ambiguous, underspecified, or the fix requires a product decision, ask the user rather than guessing.
- Never force-push or rewrite history on branches you didn't create.
