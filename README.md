# issue-fixer

A Claude Code plugin that fixes GitHub issues end-to-end: investigates the codebase, implements a fix, verifies it, and opens a pull request.

## Install

**In this repo:** already wired up. `.claude/skills` and `.agents/skills` are both symlinks to `../skills` — anyone who clones this repo and opens Claude Code or Codex in it gets `issue-fixer` auto-discovered at project scope, no install step needed.

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

User-invoked only — Claude won't start it on its own, since it pushes branches and opens PRs without asking. Pass one or more GitHub links:

```
/issue-fixer https://github.com/owner/repo/issues/42
/issue-fixer https://github.com/owner/repo
/issue-fixer https://github.com/owner/repo/issues?q=label:ready-for-agent
/issue-fixer https://github.com/owner/repo https://github.com/owner/other/issues/7
```

- **Issue link** — that issue is a candidate.
- **Repo link** — every open issue in the repo is a candidate. An issues URL with a `q=` query narrows it to matching issues (e.g. by label).

It pools every candidate, drops those that are closed, blocked by another open issue, already have an open PR, or carry `human-review-needed`, then picks the most urgent one and fixes it. One issue per run. With no link, it stops and asks for one.

## Repo layout

```
issue-fixer/
├── .claude-plugin/
│   └── plugin.json           # Claude Code plugin manifest, for marketplace distribution
├── .codex-plugin/
│   └── plugin.json           # Codex plugin manifest, for marketplace distribution
├── .claude/skills -> ../skills   # project-scope auto-load (symlink), for Claude Code
├── .agents/skills -> ../skills    # project-scope auto-load (symlink), for Codex
├── skills/
│   └── issue-fixer/SKILL.md  # one folder per skill
├── claude-session/           # cron + tmux scheduler for unattended runs
└── .github/workflows/validate.yml
```

Skills live one-per-folder under `skills/`; Claude Code and Codex both auto-discover everything there, so adding a new skill is just `skills/<name>/SKILL.md` — no manifest changes needed. The `.claude/skills` and `.agents/skills` symlinks are what make that discovery apply automatically when you're working *in this repo*; the `.claude-plugin/plugin.json` and `.codex-plugin/plugin.json` manifests are separate and only matter when installing `issue-fixer` into *other* projects via the marketplace flow above.

## What it does

0. Verifies GitHub access — MCP server or `gh` CLI, plus read and push access to every linked repo. If any is missing, it stops and reports what.
1. Builds the candidate pool from the links and filters it as above.
2. Picks the issue most worth fixing and says why.
3. Reads the whole issue thread.
4. Clones the repo fresh into a scratch location (never touches your local checkouts).
5. Reproduces the bug before changing anything.
6. Implements the smallest correct fix, following the repo's conventions.
7. Runs relevant tests/build and adds a regression test where practical. If tests still fail, it punts; if there are no tests to run, the PR is opened as a draft.
8. Opens a pull request with `Fixes #<n>` and notes how it was verified.

It never stops to ask. If the picked issue turns out to be ambiguous or can't be finished safely, it **punts**: comments on the issue explaining why, adds the `human-review-needed` label (creating it if needed), and ends the run. Later runs skip punted issues until a human removes the label.

## Scheduled runs

`claude-session/` runs issue-fixer unattended on a cron schedule in a tmux session (set up by `claude-session/setup.sh`, from [claude-session](https://github.com/Yuzhouboat/claude-session)). `claude-session/claude-schedule.conf` sets the schedule and the prompt, including which links to pass. For unattended runs, pass a label-filtered link so only issues a maintainer has labeled get picked — issue text is written by anyone who can file an issue:

```bash
PROMPT="/issue-fixer https://github.com/Yuzhouboat/Y_Know/issues?q=label:ready-for-agent"
CRON_SCHEDULE="1 * * * *"
```

## Requirements

- GitHub access via one of: a connected GitHub MCP server, or the `gh` CLI installed and authenticated (`gh auth login`). Either works; the skill checks for one at the start.
- Read and push access to every linked repo. issue-fixer doesn't fork; without push access it stops.
- `git` able to clone the target repo.

## License

MIT — see [LICENSE](LICENSE).
