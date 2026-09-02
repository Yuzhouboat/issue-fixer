# issue-fixer

A Claude Code plugin that fixes GitHub issues end-to-end: investigates the codebase, implements a fix, verifies it, and opens a pull request.

## Install

**In this repo:** already wired up. `.claude/settings.json` registers this directory as a self-contained marketplace and enables the plugin at project scope — anyone who clones this repo and opens Claude Code in it gets `issue-fixer` automatically, no install step needed.

**In another project**, install it from the published marketplace:

```
/plugin marketplace add Yuzhouboat/claude-marketplace
/plugin install issue-fixer
```

Or for local development against a checkout: `claude --plugin-dir /path/to/issue-fixer`.

## Usage

Just give Claude a GitHub issue and ask it to fix it — the `issue-fixer` skill loads automatically:

```
fix issue #42
resolve this issue: https://github.com/owner/repo/issues/42
```

## Repo layout

```
issue-fixer/
├── .claude-plugin/
│   └── plugin.json           # plugin manifest
├── .claude/settings.json     # registers + enables the plugin at project scope
├── skills/
│   └── issue-fixer/SKILL.md  # one folder per skill
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
