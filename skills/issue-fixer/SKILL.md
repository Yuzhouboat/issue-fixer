---
name: issue-fixer
description: Fix a GitHub issue end-to-end — investigate the codebase, implement a fix, verify it, and open a pull request. Use when the user gives a GitHub issue URL or number and asks to fix it, resolve it, or send a PR for it (e.g. "fix issue #42", "can you resolve this issue: <url>", "open a PR for this bug report").
disable-model-invocation: true
---

# issue-fixer

Fix a GitHub issue end-to-end: understand it, implement a fix, verify it, and open a PR.

This runs unattended, start to finish — never pause mid-run to ask the user a question or wait on their input. Each run handles exactly one issue. When you hit something that genuinely needs a human's judgment on that issue (it's ambiguous, blocked, already has a PR, or otherwise can't be safely finished), don't ask — **punt it**: see "Punting an issue" below. Punting ends the run — don't automatically pick another candidate and try again; report what happened and stop. A later run can pick a different candidate on its own (the punted one is now excluded, having a `human-review-needed` label).

## Punting an issue

When a specific issue (the explicit target, or one you're actively working, not one merely dropped during candidate pooling) can't be safely carried through to a PR, do this instead of stopping to ask:

1. Leave a comment on the issue explaining concretely why you stopped — what's ambiguous, what it's blocked on, which PR already covers it, etc.
2. Add the `human-review-needed` label. If the repo doesn't have that label yet, create it first (matches the flow used for any other label the repo is missing).
3. Stop the run here — don't pick another candidate and retry within the same run, whether this issue was explicit, config-pinned, or the one candidate you selected. Report what you found and that you punted it. A later invocation's candidate pooling will skip this issue automatically now that it carries `human-review-needed`.

Note the one exception: a **closed** issue isn't punted this way — there's nothing to review, it's just not actionable. Skip it and say so in your report; don't comment or relabel it.

## Inputs

Run step 0 under "Steps" below first — verify GitHub tooling and access before doing anything else, including the candidate pooling described here, since pooling itself reads from GitHub.

Resolve the target in this order, stopping at the first that succeeds:

1. **Explicit input.** The user gives a GitHub issue URL (`https://github.com/<owner>/<repo>/issues/<n>`) or a `#<n>` plus repo context.
2. **Config file.** If no issue was given, look for `.issue-fixer.json` in the repo root (of the current working directory). It holds either a specific issue, or a list of repos (each with its own label filter) to pick a candidate from — see `references/config.example.json` for the shape. Read it yourself rather than parsing it with a strict JSON tool; it may contain `//` comments explaining each field.

   - `issue`, if present, wins outright — treat it like explicit input (skip candidate selection below).
   - Otherwise use `repos` — a list of `{ "repo": "owner/name", "labels": [...] }` entries — to find a candidate across all of them. `labels` is optional per entry; if omitted, consider all open issues in that repo.
3. **Abort.** If there's no explicit issue and no usable config file (missing, or has neither `issue` nor a non-empty `repos`), stop and tell the user you need one of the two — don't guess a repo from the current directory's git remote.

### Picking a candidate issue (repos-list resolution)

When resolution lands on a `repos` list rather than a specific issue:

1. For each repo entry, list its open issues and, if that entry has `labels`, keep only issues carrying at least one of them. Pool candidates across all entries, tagged with their source repo.
2. Drop any candidate that's **blocked by another open issue** — scan the body/comments for references like "blocked by #N", "depends on #N", or a reverse "blocks #N" on another open issue, and check for a `blocked` label. An issue referencing a now-closed blocker isn't blocked.
3. Drop any candidate that **already has an open pull request linked to it** — check the issue's linked PRs, or search open PRs in that repo for one referencing the issue number (e.g. a body containing "Fixes #N" / "Closes #N" / "Resolves #N", or a branch/title naming it). Someone's already working the fix; don't duplicate it. A PR that's closed/merged without closing the issue doesn't count as covering it.
4. Drop any candidate already carrying the `human-review-needed` label — a previous run already punted it (see "Punting an issue"); don't pick it again until a human has cleared that label.
5. If no candidates survive, abort and tell the user why (no matching labels anywhere, or everything remaining is blocked, closed, already has an open PR, or already punted).
6. From what's left, judge which one is most worth fixing first — weigh severity/impact, priority-style labels, how well-scoped and actionable it is, and signal like comment/reaction volume. This is a judgment call, not a mechanical rule (e.g. not just "oldest"), and it spans repos — the most urgent issue wins regardless of which repo entry it came from. Briefly tell the user which repo and issue you picked and why before continuing.

## Tools

Two ways to talk to GitHub work: the `github` MCP server's tools (`issue_read`, `get_file_contents`, `create_pull_request`, etc.), or the `gh` CLI. Prefer the MCP tools when connected — they're more structured — and fall back to `gh` when they're not. Don't assume either is present; step 0 below checks.

## Steps

0. **Verify tools & access.** Do this before reading the issue or touching any code — fail fast rather than discovering a missing permission after the work is done.
   - **Tooling available.** Check whether `github` MCP tools are connected. If not, check the `gh` CLI is installed and authenticated (`gh --version`, `gh auth status`). If neither works, abort and tell the user what to set up (connect a GitHub MCP server, or `gh auth login`).
   - **Read access.** Use whichever tool is available to fetch the target repo (or, for config-driven runs, every repo listed) — this confirms both connectivity and read permission before you invest effort understanding the issue.
   - **Write access.** Confirm the authenticated identity can open a PR against the repo (or, for config-driven runs, every repo listed) — e.g. `gh api repos/<owner>/<repo> --jq .permissions` (needs `push: true`) or the MCP equivalent. If push access is missing, fall back to forking the repo and opening the PR from the fork automatically — no need to check with the user first, just note the fork in your final report.
   - If tooling, or read/write access entirely (not just push — see above), is missing, stop and report exactly what's missing. This is an environment-level blocker, not a per-issue judgment call, so there's no issue to punt it to — just report it plainly to the user.
1. **Fetch the issue.** Pull the issue title, body, labels, and comments with the available tool. Read the whole thread — reproduction steps and constraints often live in comments, not just the body.
2. **Check status.** Confirm the issue is open — if it's closed, skip it per "Punting an issue" above (the closed-issue exception: just report, no comment/label). Check for blockers the same way as in candidate selection; if this issue (given explicitly, or via the config's `issue` field) is blocked by another open issue, punt it — a fix built against unresolved dependent work may not hold up. Also check the same way as in candidate selection whether an open PR already links to this issue; if one does, punt it too rather than assuming your fresh attempt should supersede someone else's in-flight fix — that call needs a human.
3. **Locate the repo.** If it's already checked out locally (the current working directory, or a path the user has already mentioned), use that. Otherwise just clone it into a scratch/tmp location (e.g. the session's scratchpad directory) — don't ask the user where their checkout lives, and don't edit via the API.
4. **Reproduce/understand first.** Before writing a fix, confirm you understand the bug or request — find the relevant code, and where practical reproduce the failure (failing test, repro script, or manual trace).
5. **Implement the smallest correct fix.** Follow the repo's existing conventions. Don't refactor unrelated code.
6. **Verify.** Run the project's tests/build relevant to the change. Add or update a test that would have caught the bug, if the repo has a test suite.
7. **Open a PR.** Create a branch, commit with a message describing the fix and referencing the issue (`Fixes #<n>`), push (or push to your fork and open cross-repo, if step 0 fell back to a fork), and open a pull request with the available tool. Summarize what changed and why, link the issue, and note its status from step 2.

## Guardrails

- Run end-to-end without pausing for the user — push branches and open PRs directly once you've reached step 7; the config file and/or the user's invocation is the authorization. Don't re-confirm mid-run.
- If the issue is ambiguous, underspecified, or the fix requires a product decision you can't make from the issue thread and codebase alone, punt it (see "Punting an issue") rather than guessing.
- Never force-push or rewrite history on branches you didn't create.
- Never silently drop an issue you started actively working (explicit target, config `issue`, or the one candidate you picked) — punt it per "Punting an issue" so there's a durable trace on GitHub, not just something said to the user. Issues dropped earlier during candidate *pooling* (blocked, closed, already-has-a-PR, wrong labels) don't each need this — summarizing them in your final report is enough.
- Never skip step 0 — don't discover a missing credential or permission midway through a fix.
