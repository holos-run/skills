# holos-run/skills

Claude Code plugin marketplace hosting **`linear-workflow`** — a tight plan → implement → review loop for Linear tickets, designed to run under [Cyrus](https://github.com/ceedaragents/cyrus) or any Claude Code session with the Linear MCP server.

- **Repo:** https://github.com/holos-run/skills
- **Marketplace name:** `holos-run`
- **Plugin name:** `linear-workflow`
- **License:** MIT

## Installation

Install directly from this GitHub repo using Claude Code's built-in plugin commands:

```text
/plugin marketplace add holos-run/skills
/plugin install linear-workflow@holos-run
```

The first command registers this repository as a marketplace (reading `.claude-plugin/marketplace.json`). The second installs the `linear-workflow` plugin from it, which exposes three slash commands:

- `/plan-primary-issue`
- `/implement-primary-issue`
- `/implement-sub-issue`

> `/plugin marketplace add` accepts the `owner/repo` shorthand. Full `https://github.com/...` URLs and `git@github.com:...` URLs also work. You can pin a branch or tag with `holos-run/skills@main`.

### Update or uninstall

```text
/plugin marketplace update holos-run
/plugin update linear-workflow@holos-run

/plugin uninstall linear-workflow@holos-run
/plugin marketplace remove holos-run
```

### Prerequisites

- **Linear MCP server** — the skills call `mcp__linear-server__*` tools. Configure the Linear MCP server in Claude Code, or run the plugin under Cyrus, which provides it.
- **GitHub CLI (`gh`)** — authenticated against the repo you're shipping code into; used for branches, PRs, reviews, and merges.
- **A Linear team using native parent/child issues** — sub-tickets are linked via `parentId`, not markdown checkbox parsing.

### Setting up a persistent Linear MCP connection

The Linear MCP server is reachable over HTTP at `https://mcp.linear.app/mcp`. Wire it into Claude Code using `mcp-remote` as a local bridge, authenticated with a Linear API key.

1. **Install `mcp-remote`** globally:

   ```shell
   npm install -g mcp-remote
   ```

2. **Configure your Linear API key** (keys start with `lin_api_`) in `~/.cyrus/.env`:

   ```shell
   LINEAR_API_KEY=lin_api_xxxxxxxxxxxxxxxxxxxxxxxx
   ```

   Then restart Cyrus so it picks up the new environment:

   ```shell
   systemctl restart cyrus
   ```

3. **Register the MCP server** with Claude Code. If an existing `linear-server` entry is present, remove it first:

   ```shell
   claude mcp remove linear-server
   claude mcp add linear-server --scope user -- npx mcp-remote https://mcp.linear.app/mcp --header 'Authorization: Bearer ${LINEAR_API_KEY}'
   ```

## Repository layout

```
skills/
├── .claude-plugin/
│   ├── marketplace.json      # marketplace manifest (lists linear-workflow)
│   └── plugin.json           # plugin manifest for linear-workflow
├── skills/
│   ├── plan-primary-issue/SKILL.md
│   ├── implement-primary-issue/SKILL.md
│   └── implement-sub-issue/SKILL.md
└── README.md
```

`marketplace.json` and `plugin.json` both live under `.claude-plugin/`, with the plugin rooted at the repo itself (`"source": "./."`). This matches the Claude Code plugin marketplace spec for single-plugin repos — one `/plugin marketplace add` registers both the marketplace and the plugin for install.

## The `linear-workflow` plugin

Three skills cover the full lifecycle of a Linear ticket, from plan to merged PR. They're designed to be triggered by Cyrus when a human assigns a Linear ticket to the Cyrus agent user, but they also work as ordinary Claude Code slash commands in a local session.

| Skill | Role |
|---|---|
| `/plan-primary-issue <HOL-###>` | Explore the codebase and turn a Linear ticket into a phased plan. Writes the master plan into the parent ticket's description and creates one Linear sub-ticket per phase (`parentId` set natively). |
| `/implement-primary-issue <HOL-###>` | Orchestrator. Lists the parent ticket's Linear sub-tickets, dispatches each to `/implement-sub-issue` in order, tracks per-phase wall-clock timing, sweeps for follow-up tickets, and posts a summary. |
| `/implement-sub-issue <HOL-###>` | End-to-end lifecycle for a single ticket: branch from `origin/main`, comment on the Linear ticket with the agent slot, implement per repo conventions, open a GitHub PR linking back to Linear (`Fixes HOL-###`), run cross-model code review, fix findings, wait for CI, merge, then transition the Linear ticket to `Done`. |

See each skill's `SKILL.md` for the full workflow and triggering phrases.

### Typical flow

1. **Plan** — `/plan-primary-issue HOL-525` adds a `planning` label to the parent ticket, explores the codebase, writes a master plan into the parent ticket body, and creates one Linear sub-ticket per phase.
2. **Implement** — `/implement-primary-issue HOL-525` swaps `planning` for `implementing` on the parent, iterates over its Linear children, and dispatches each to `/implement-sub-issue`.
3. **Merge** — Each `/implement-sub-issue` invocation handles its sub-ticket end-to-end: branch, PR, cross-model review, CI, merge, and move the Linear ticket to `Done`. Follow-up tickets (things found during review) are created as new Linear sub-tickets under the same parent.

### How it works with Linear

Linear is the source of truth for tickets. The skills use `mcp__linear-server__*` MCP tools to:

- Fetch tickets (`get_issue`) and discover sub-tickets by `parentId` — not by parsing `[ ]` task lists.
- Apply lifecycle labels (`planning` → `implementing`) so operators can see which tickets an agent currently owns.
- Post structured comments including the agent slot (e.g. `agent-1`) derived from the worktree path, so you can tell which background agent is working on which ticket.
- Write the master plan directly into the parent ticket's body and create per-phase sub-tickets with `parentId` set, producing a native Linear parent/child tree.
- Move tickets to `Done` after merge, as a belt-and-suspenders safeguard alongside the `Fixes HOL-###` magic-word in the PR body.

Code still ships through GitHub — the PR is opened with `gh`, reviewed, CI-gated, and merged — but ticket state lives in Linear throughout.

### How it works with Cyrus

[Cyrus](https://github.com/ceedaragents/cyrus) is a background agent runner that watches Linear, spins up a Claude Code session in a git worktree for each assigned ticket, and streams the agent's activity back to the Linear ticket as comments. The `linear-workflow` skills are built for this flow:

- **Worktree-aware agent slots.** Cyrus places each session in a worktree named like `holos-console-agent-<N>`. Every skill extracts that slot from `$(pwd)` and includes it in Linear comments so humans can correlate a comment with a specific background agent.
- **One skill per Cyrus role.** Assign a parent (planning) ticket to Cyrus → `/plan-primary-issue` fires. Re-assign that parent to run the plan → `/implement-primary-issue` fires. Assign a single sub-ticket → `/implement-sub-issue` fires. The three skills map cleanly onto the three common Cyrus entry points.
- **Linear round-trips, not ad-hoc chatter.** Everything meaningful — plan body, phase list, agent slot, merge status, follow-up tickets — lands on the Linear ticket. This keeps Cyrus's Linear-native activity stream the durable record of the work.
- **Safe handoff between skills.** `/implement-primary-issue` only orchestrates; it delegates the full branch/PR/review/CI/merge lifecycle to `/implement-sub-issue` per phase, so any single sub-ticket can be re-run independently without duplicating logic.

Cyrus is not required — these skills run fine in a plain `claude` session — but the conventions (agent slot, Linear-first state, parent/child sub-tickets) are chosen to make Cyrus-driven work legible.

## License

MIT.
