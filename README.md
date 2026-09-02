# issue-fixer

A Claude Code plugin that fixes GitHub issues end-to-end: investigates the codebase, implements a fix, verifies it, and opens a pull request.

## Install

```
/plugin marketplace add <owner>/issue-fixer
/plugin install issue-fixer
```

(Replace `<owner>/issue-fixer` with this repo's path once published, or use a local path during development: `claude --plugin-dir /path/to/issue-fixer`.)

## Usage

Give Claude a GitHub issue and ask it to fix it:

```
fix issue #42
resolve this issue: https://github.com/owner/repo/issues/42
```

Or invoke explicitly:

```
/issue-fixer:fix-issue https://github.com/owner/repo/issues/42
```

## Repo layout

```
issue-fixer/
├── .claude-plugin/plugin.json   # plugin manifest
├── skills/
│   └── issue-fixer/SKILL.md     # one folder per skill
├── commands/
│   └── fix-issue.md             # explicit slash-command entry points
└── .github/workflows/validate.yml
```

Skills live one-per-folder under `skills/`; Claude Code auto-discovers everything there, so adding a new skill is just `skills/<name>/SKILL.md` — no manifest changes needed.

## What it does

1. Fetches the issue (title, body, comments, labels) via the GitHub MCP tools.
2. Locates and understands the relevant code.
3. Reproduces the bug or confirms the request before changing anything.
4. Implements the smallest correct fix, following the repo's conventions.
5. Runs relevant tests/build; adds a regression test where practical.
6. Opens a pull request referencing the issue.

## Requirements

- A GitHub MCP server connected (for issue lookup and PR creation).
- A local checkout of the target repo, or the ability to clone it.

## License

MIT — see [LICENSE](LICENSE).
