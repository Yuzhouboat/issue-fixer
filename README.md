# issue-fixer

A Claude Code plugin that fixes GitHub issues end-to-end: investigates the codebase, implements a fix, verifies it, and opens a pull request.

## Install

**In this repo:** already wired up. `.claude/skills` is a symlink to `../skills` — anyone who clones this repo and opens Claude Code in it gets `issue-fixer` auto-discovered at project scope, no install step needed.

**In another project**, install it from the published marketplace:

```
/plugin marketplace add Yuzhouboat/yuzhou-agent-toolkit
/plugin install issue-fixer
```

Or for local development against a checkout: `claude --plugin-dir /path/to/issue-fixer`.

**Codex**, via the same marketplace:

```bash
codex plugin marketplace add git@github.com:Yuzhouboat/yuzhou-agent-toolkit.git
codex plugin add issue-fixer@yuzhou-agent-toolkit
```

## Usage

Just give Claude a GitHub issue and ask it to fix it — the `issue-fixer` skill loads automatically:

```
fix issue #42
resolve this issue: https://github.com/owner/repo/issues/42
```

If you don't name an issue, it falls back to a `.issue-fixer.json` file in the repo root:

```json
{
  "repos": [
    { "repo": "owner/repo", "labels": ["bug", "priority:high"] },
    { "repo": "owner/other-repo", "labels": ["bug"] }
  ]
}
```

With `repos` (no `issue`), it pools open issues across every listed repo — filtered by that entry's `labels`, if given — skips anything blocked by another open issue, and picks the most urgent candidate itself, from whichever repo it came from. Set `issue` in the config instead to pin a specific one. If there's no explicit issue and no usable config file, it aborts rather than guessing.

## Repo layout

```
issue-fixer/
├── .claude-plugin/
│   └── plugin.json           # Claude Code plugin manifest, for marketplace distribution
├── .codex-plugin/
│   └── plugin.json           # Codex plugin manifest, for marketplace distribution
├── .claude/skills -> ../skills   # project-scope auto-load (symlink)
├── skills/
│   └── issue-fixer/SKILL.md  # one folder per skill
└── .github/workflows/validate.yml
```

Skills live one-per-folder under `skills/`; Claude Code auto-discovers everything there, so adding a new skill is just `skills/<name>/SKILL.md` — no manifest changes needed. The `.claude/skills` symlink is what makes that discovery apply automatically when you're working *in this repo*; `.claude-plugin/plugin.json` is separate and only matters when installing `issue-fixer` into *other* projects via the marketplace flow above.

## What it does

0. Verifies it has working GitHub access — the right tooling, plus read and PR-create permission on the target repo — before doing anything else.
1. Fetches the issue (title, body, comments, labels).
2. Confirms the issue is open and checks whether it's blocked by another open issue.
3. Locates and understands the relevant code.
4. Reproduces the bug or confirms the request before changing anything.
5. Implements the smallest correct fix, following the repo's conventions.
6. Runs relevant tests/build; adds a regression test where practical.
7. Opens a pull request referencing the issue.

## Requirements

- GitHub access via one of: a connected GitHub MCP server, or the `gh` CLI installed and authenticated (`gh auth login`). Either works; the skill checks for one at the start.
- Push/PR-create permission on the target repo (or issue-fixer will tell you it can only work from a fork).
- A local checkout of the target repo, or the ability to clone it.

## License

MIT — see [LICENSE](LICENSE).
