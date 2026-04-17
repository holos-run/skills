# linear-workflow

Claude Code plugin providing a tight orchestration loop for plan, implement, and review when developing with GitHub + Linear + Slack.

## Installation

Add to your user settings (`~/.claude/settings.json`):

```json
{
  "plugins": [
    "/path/to/linear-workflow"
  ]
}
```

Or load on-the-fly:

```bash
claude --plugin-dir /path/to/linear-workflow
```

## Skills

| Skill | Purpose |
|-------|---------|
| `/linear-workflow:plan-primary-issue` | Break a Linear ticket into phased sub-tickets with a master plan |
| `/linear-workflow:implement-primary-issue` | Orchestrate sequential implementation of all sub-tickets under a parent |
| `/linear-workflow:implement-sub-issue` | Implement a single Linear ticket end-to-end: branch, code, PR, review, CI, merge |

### Typical workflow

1. **Plan** — `/linear-workflow:plan-primary-issue HOL-525` explores the codebase and creates phased sub-tickets under the parent.
2. **Implement** — `/linear-workflow:implement-primary-issue HOL-525` iterates over each sub-ticket, dispatching to `implement-sub-issue` which handles the full lifecycle.
3. **Review** — Each sub-ticket PR is reviewed automatically, findings are fixed in-PR, and follow-up tickets are created for anything that persists.

### Prerequisites

- **Linear MCP server** configured in your Claude Code settings
- **GitHub CLI** (`gh`) authenticated with repo access
- **Git** with a clean working directory on `main`
