# issue-fixer

A Claude Code plugin that fixes GitHub issues end-to-end: investigates the codebase, implements a fix, verifies it, and opens a pull request.

## Install

**In this repo:** already wired up. `.claude/skills` and `.agent/skills` are both symlinks to `../skills` — anyone who clones this repo and opens Claude Code or Codex in it gets `issue-fixer` auto-discovered at project scope, no install step needed.

**In another project**, install it from the published marketplace:

```
/plugin marketplace add Yuzhouboat/yuzhou-agent-toolkit
/plugin install issue-fixer -s user       # every project on this machine (default)
/plugin install issue-fixer -s project    # this repo only, via .claude/settings.json
```

Or for local development against a checkout: `claude --plugin-dir /path/to/issue-fixer`.

**Codex**, via the same marketplace:

```bash
codex plugin marketplace add git@github.com:Yuzhouboat/yuzhou-agent-toolkit.git
codex plugin add issue-fixer@yuzhou-agent-toolkit
```

Codex has no project-scope install — `codex plugin add` always installs machine-wide, unlike
Claude Code's `-s user|project`. Both install paths above were verified with a real install
(`claude plugin install` / `codex plugin add`); see the
[yuzhou-agent-toolkit README](https://github.com/Yuzhouboat/yuzhou-agent-toolkit#claude-code-plugin-marketplace)
for the full verification notes.

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
├── .claude/skills -> ../skills   # project-scope auto-load (symlink), for Claude Code
├── .agent/skills -> ../skills    # project-scope auto-load (symlink), for Codex
├── skills/
│   └── issue-fixer/SKILL.md  # one folder per skill
└── .github/workflows/validate.yml
```

Skills live one-per-folder under `skills/`; Claude Code and Codex both auto-discover everything there, so adding a new skill is just `skills/<name>/SKILL.md` — no manifest changes needed. The `.claude/skills` and `.agent/skills` symlinks are what make that discovery apply automatically when you're working *in this repo*; the `.claude-plugin/plugin.json` and `.codex-plugin/plugin.json` manifests are separate and only matter when installing `issue-fixer` into *other* projects via the marketplace flow above.

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
