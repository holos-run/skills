# Simple Workflow

Claude Code plugin for planning and implementing software with plain markdown
plan files in the repository instead of an external issue tracker. It is the
`linear-workflow` plan/implement/review loop with every Linear call replaced
by a file edit that is committed and pushed to `main` as it happens.

## Quick Start

```bash
claude plugin install simple-workflow@holos-run
```

```bash
/simple-workflow:plan-issue "Add login with email and password"
/simple-workflow:implement-issue 001
```

## Where plans live

The skills resolve the plans folder in this order:

1. A `Plans directory: <path>` (or `Plans folder:` / `plans_dir:`) line, or a
   `## Plans` section naming a directory, in the repository's `CLAUDE.md` or
   `AGENTS.md`.
2. A top-level `plans/` directory, created on first use.

Layout inside the plans folder:

```
plans/
├── TODO.md                          # one line per item, ticked when done
├── 001-add-login.md                 # primary plan (id 001)
└── 001-add-login/
    ├── 01-user-schema.md            # phase (id 001-01)
    ├── 02-login-handler.md          # phase (id 001-02)
    └── 03-cleanup.md                # phase (id 001-03)
```

Every item is a markdown file with YAML front matter carrying `id`, `title`,
`status` (`todo` | `in-progress` | `done` | `canceled`), `labels`
(`planning`, `implementing`, `needs-human-review`, `escalation`), `parent`,
`blocked_by`, `related`, `branch`, `pr`, `created`, and `updated`. A `## Activity`
section at the end of each file holds timestamped entries that replace tracker
comments; each entry ends with an agent-attribution footer.

## How status reaches git

Plan files are only ever modified on `main`. Each status change is a small
commit built in a temporary detached worktree at `origin/main` and pushed
straight to `main` (`git push origin HEAD:main`). Feature branches never touch
the plans folder, so code PRs stay clean and progress is visible on `main` the
moment it happens. When branch protection rejects direct pushes the skill
falls back to a `plans/status-*` branch and a pull request it merges itself.

Typical commits during one phase:

```
chore(plans): start 001-02
chore(plans): 001-02 opened PR #42
chore(plans): 001-02 done
```

`TODO.md` is updated in the same commit as the item it describes.

## Skills

| Skill | Purpose |
| --- | --- |
| `/simple-workflow:plan-issue` | Explore the codebase and write a primary plan file plus one phase file per phase; update `TODO.md` |
| `/simple-workflow:implement-issue` | Implement a plan item: leaf items directly (branch, PR, adversarial review, CI, merge, mark done), parent items by orchestrating one worker per phase |

Both skills accept `--model <name>` to pin the implementation model within the
invoking harness. `implement-issue` also accepts `--reviewer` to override the
cross-runtime code reviewer. Review behaviour, harness rules, and escalation
are identical to `linear-workflow`, except that reviewer failures produce an
escalation plan item (`NNN-reviewer-failure-pr-<N>.md`) instead of a Linear
issue and document.

## Prerequisites

- **GitHub CLI** (`gh`) authenticated with repo access
- **Git** push access to `main` for status commits, or PR merge rights for the
  fallback path
- **Agent teams** enabled for parent-mode orchestration:
  `claude config set -g env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS 1`
- **Codex CLI** and `jq` when implementing from Claude Code (the reviewer),
  or the **Claude CLI** when implementing from Codex
