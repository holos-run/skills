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
| `/implement-sub-issue <HOL-###>` | End-to-end lifecycle for a single ticket: branch from `origin/main`, comment on the Linear ticket, implement per repo conventions, open a GitHub PR linking back to Linear (`Fixes HOL-###`), run cross-model code review, fix findings, wait for CI, merge, then transition the Linear ticket to `Done`. |

See each skill's `SKILL.md` for the full workflow and triggering phrases.

### Typical flow

1. **Plan** — `/plan-primary-issue HOL-525` applies `role/planner` to the parent ticket, explores the codebase, writes a master plan into the parent body, creates one Linear sub-ticket per phase, and finally hands off by swapping `role/planner` for `role/orchestrator` on the parent.
2. **Implement** — `/implement-primary-issue HOL-525` picks up the parent already carrying `role/orchestrator`, iterates over its Linear children, and hands each off to `/implement-sub-issue` via `role/implementer`.
3. **Merge** — Each `/implement-sub-issue` invocation handles its sub-ticket end-to-end: branch, PR, cross-model review, CI, merge, and move the Linear ticket to `Done`. Follow-up tickets (things found during review) are created as new Linear sub-tickets under the same parent.

### How it works with Linear

Linear is the source of truth for tickets. The skills use `mcp__linear-server__*` MCP tools to:

- Fetch tickets (`get_issue`) and discover sub-tickets by `parentId` — not by parsing `[ ]` task lists.
- Apply canonical `role/*` labels (`role/planner` → `role/orchestrator` → `role/implementer` / `role/maintainer`) so operators can see which role currently owns each ticket.
- Post structured comments tagged with the role (and optional `agent=<AGENT_NAME>` when the environment exports one), so you can tell which role handled each handoff.
- Write the master plan directly into the parent ticket's body and create per-phase sub-tickets with `parentId` set, producing a native Linear parent/child tree.
- Move tickets to `Done` after merge, as a belt-and-suspenders safeguard alongside the `Fixes HOL-###` magic-word in the PR body.

Code still ships through GitHub — the PR is opened with `gh`, reviewed, CI-gated, and merged — but ticket state lives in Linear throughout.

### How it works with Cyrus

[Cyrus](https://github.com/ceedaragents/cyrus) is a background agent runner that watches Linear, spins up a Claude Code session in a git worktree for each assigned ticket, and streams the agent's activity back to the Linear ticket as comments. The `linear-workflow` skills are built for this flow:

- **Task-scoped worktrees.** Cyrus places each session in a worktree dedicated to one task (e.g., `~/.cyrus/worktrees/HOL-724`). The skills treat the worktree as task-scoped only — they do not parse the path for an agent slot. If you want comments tagged with a reusable agent identity, export `AGENT_NAME` in the session environment and the skills will include `agent=<AGENT_NAME>` in the comment tag.
- **One skill per Cyrus role.** Assign a parent (planning) ticket to Cyrus → `/plan-primary-issue` fires. Re-assign that parent to run the plan → `/implement-primary-issue` fires. Assign a single sub-ticket → `/implement-sub-issue` fires. The three skills map cleanly onto the three common Cyrus entry points.
- **Linear round-trips, not ad-hoc chatter.** Everything meaningful — plan body, phase list, role handoffs, merge status, follow-up tickets — lands on the Linear ticket. This keeps Cyrus's Linear-native activity stream the durable record of the work.
- **Label-based handoff between agents.** `/implement-primary-issue` does not implement sub-tickets in-process. It applies a `role/*` label to each sub-ticket — the implementer Cyrus agent matches that label via its `routingLabels` config, picks up the ticket, and runs `/implement-sub-issue`. The orchestrator then polls Linear until the sub-ticket reaches a terminal state.

Cyrus is not required — the skills still run in a plain `claude` session and fall back to in-session dispatch when no implementer agent answers within the handoff-ack timeout — but the full multi-agent flow needs at least one implementer and one maintainer Cyrus instance.

### Multi-agent handoff architecture

The three skills split Linear tickets across **role-specialized Cyrus agents**, all running as a single Linear identity. Role selection is driven by labels on the ticket, matched against Cyrus's per-repository `labelPrompts` and `routingLabels`.

| Role | Triggering label | Cyrus prompt mode | Typical `allowedTools` | Applied by |
|---|---|---|---|---|
| Planner | `role/planner` | `scoper` | `readOnly` + MCP writes | Human (assigns master ticket) |
| Orchestrator | `role/orchestrator` | `scoper` or custom | `readOnly` + MCP writes | `/plan-primary-issue` (as its final handoff step on the master ticket) |
| Implementer | `role/implementer` | `builder` | `all` | `/implement-primary-issue`, one sub-ticket at a time |
| Maintainer | `role/maintainer` | `builder` | `safe` (no `Bash` beyond allow-list) | `/implement-sub-issue` when creating a follow-up from review |

**Flow.**

1. A human assigns a master ticket to Cyrus (the master ticket has `role/planner` — or the planner skill applies it at entry).
2. `/plan-primary-issue` writes the plan body, creates phase sub-tickets (unlabeled by default), swaps `role/planner` for `role/orchestrator` on the master as its final step, and exits.
3. A human re-assigns the master ticket to Cyrus to kick off execution. `/implement-primary-issue` picks it up already carrying `role/orchestrator`.
4. For each phase sub-ticket, the orchestrator applies `role/implementer`, then polls Linear (`get_issue` + `list_comments`, every 60s) until the sub-ticket transitions to a completed state, gets `needs-human-review`, or stalls (no activity for 30 minutes).
5. The implementer Cyrus agent (matched by `routingLabels: ["role/implementer"]`) picks up the labeled sub-ticket, runs `/implement-sub-issue` end-to-end (branch, code, PR, Codex review, CI, merge), and posts a final summary comment.
6. If the review produced style-only follow-ups, `/implement-sub-issue` creates a follow-up Linear sub-ticket labeled `role/maintainer`. The maintainer Cyrus agent picks those up autonomously — the orchestrator's follow-up sweep polls them but does not reapply any label.
7. After the queue drains, `/implement-primary-issue` posts a summary on the master ticket with per-role wall-clock time and moves the master to `Done` if every child is `completed` or `canceled`.

**Sample `~/.cyrus/config.json` stanza** (single Cyrus process, three roles via label routing):

```json
{
  "repositories": [
    {
      "name": "holos-console",
      "repositoryPath": "/home/jeff/workspace/holos-run/holos-console",
      "baseBranch": "main",
      "routingLabels": ["role/implementer", "role/maintainer", "role/orchestrator", "role/planner"],
      "labelPrompts": {
        "scoper": { "labels": ["role/planner", "role/orchestrator"], "allowedTools": "readOnly" },
        "builder": { "labels": ["role/implementer"], "allowedTools": "all" },
        "debugger": { "labels": ["role/maintainer"], "allowedTools": "safe" }
      }
    }
  ]
}
```

A fully separated **multi-identity** deployment (one OAuth user per role) is possible but not required — it buys a distinct audit trail per role at the cost of 3× the OAuth and webhook surface. See [HOL-724](https://linear.app/holos-run/issue/HOL-724) for the rationale behind picking the single-identity pattern as the default.

**Comment tagging for cross-hop observability.** Every comment the skills post on a Linear ticket is prefixed with `[role=<role> gen=<N>]`. `role` is `planner`, `orchestrator`, `implementer`, or `maintainer`; `gen` increments each time the same ticket is re-handed-off (typically `1`, `2` after a stall retry). When the session environment exports `AGENT_NAME`, the tag also includes `agent=<AGENT_NAME>` (e.g., `[role=implementer agent=builder-1 gen=1]`) so a reusable agent identity can be correlated across tasks. Agent identity is **not** derived from the worktree path — worktrees are task-scoped, not agent-scoped.

## License

MIT.
