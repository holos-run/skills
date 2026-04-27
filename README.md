# Linear Workflow

Claude Code plugin for planning and implementing software with Linear and
GitHub.  Intended to work equally well with vanilla claude code running locally
or as a self hosted Linear Agent driving the claude code sdk command line tool.

## Quick Start

Install the required assistant dependencies first:

```bash
npx skills add shadcn/ui
```

Then install this plugin:

```bash
claude plugin install linear-workflow@holos-run
```

Enable agent teams for parent issue orchestration:

```bash
claude config set -g env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS 1
```

Run the normal plan/implement loop:

```bash
/linear-workflow:plan-issue APP-123
/linear-workflow:implement-issue APP-234
```

## Model Selection via Labels

Label any Linear sub-issue to control which model implements it:

| Label | Runner |
| --- | --- |
| _(none)_ | Claude Sonnet (default) |
| `sonnet` | Claude Sonnet |
| `opus` | Claude Opus (complex architecture, deep reasoning) |
| `codex` | Codex CLI (adversarial implementation via a separate model) |

The orchestrator reads these labels before dispatching each sub-agent, so you
can mix models within a single parent issue — assign `opus` to the hardest
sub-issues and leave the rest unlabeled for Sonnet. When conflicting labels are
present, precedence is: `codex` > `opus` > `sonnet`.

## Per-Sub-Issue Base-Branch Routing

By default, when Cyrus picks up a sub-issue under a parent issue, its
`determineBaseBranch` rule resolves the base branch in this priority order:

1. `[repo=<name>#<branch>]` override tag at the top of the issue description
2. Graphite "blocked-by" relations
3. **Parent issue's branch** (`parent-issue` rule)
4. Repository default branch

Without an override tag, rule 3 wins for any sub-issue with a parent — so
Cyrus's activity message reads `→ cyrus/hol-1028-…` and the sub-issue's
worktree is branched off the parent issue's branch instead of `main`. That
breaks the per-phase PR model: each sub-issue's commits stack on the parent's
branch and cannot merge to `main` independently.

`plan-issue` (v2.2.0+) prevents this by emitting a `[repo=<repo-name>#main]`
override on the very first line of every sub-issue description. The override
is the priority-0 `commit-ish` input to `GitService` and forces the sub-issue
worktree to branch from `main`.

`implement-issue` (v2.7.0+) defends against the same misrouting from the
other side. Before dispatching each child, the orchestrator inspects the
child's description and patches in the override if it is missing or points
somewhere other than `main`. Manually-created sub-issues, plans created by
older `plan-issue` versions, and follow-ups generated during review are
covered automatically.

### Worktree-Friendly Branching

The skill always runs in a Git worktree (Cyrus provisions a worktree per
issue). A given branch can only be checked out in one worktree at a time, so
**the workflow never checks out `main` locally** — `main` belongs to the
repo's primary worktree, not the issue worktree.

Instead, every leaf branch is created directly from `origin/main`:

```bash
git fetch origin
git checkout -b feat/<id>-<slug> origin/main
```

That single `checkout -b … origin/main` form creates the new branch from the
fetched `origin/main` tip and sets the new branch's upstream tracking to
`origin/main`. Subsequent pushes use `git push origin HEAD` (no `-u`) so the
upstream stays on `origin/main`; the PR is opened against `main` regardless.

Between sub-issue dispatches, the orchestrator stays on its own primary issue
branch and only runs `git fetch origin` to refresh remotes — it never moves
to `main`.

## What This Does

This plugin provides workflow skills that drive an **adversarial
plan-implement-review cycle**, plus targeted `holos-console` frontend skills
for dense resource UI and TanStack server-state work. Linear is the source of
truth for workflow-managed implementation; code ships through GitHub PRs. The
cycle works like this:

1. A feature is described as a **Linear issue**.
2. **Plan** — An agent explores the codebase, creates a new primary issue with a structured plan and phased sub-issues, then marks the original issue Done.
3. **Implement** — For each sub-issue, an agent-team orchestrator dispatches teammates to:
   - Implement the sub-issue (branch, code, tests, PR).
   - Run adversarial code review (round 1).
   - Fix round 1 findings.
   - Run adversarial code review (round 2).
   - Fix round 2 findings.
   - Run a final review gate — if critical/important findings remain, the issue enters `needs-human-review` and the skill stops.
   - Wait for CI; fix failures
   - Merge to base branch.
4. **Repeat** for all remaining sub-issues, then sweep for follow-ups.
5. **Summary** posted back to the primary issue with wall clock timing.

The skills are **project-agnostic** — they read conventions (build commands,
test strategy, commit format) from the target repo's `CLAUDE.md` or `AGENTS.md`
rather than encoding them in the plugin.

## Installation

```bash
claude plugin install linear-workflow@holos-run
```

## Skills

| Skill | Purpose |
| --- | --- |
| `/linear-workflow:plan-issue` | Explore codebase, create a new primary issue with phased sub-issues, relate back to original |
| `/linear-workflow:implement-issue` | Implement any Linear issue: leaf issues directly, parent issues via sub-agent orchestration |
| `/linear-workflow:enterprise-k8s-frontend` | Build dense operator-facing Vite + React resource list/detail workflows in `holos-console` using TanStack Router/Query/Table, ResourceGrid, shadcn/Tailwind, and ConnectRPC |
| `/linear-workflow:tanstack-query-and-grid` | Implement or refactor TanStack Query hooks, key factories, mutations, invalidation, and ResourceGrid toolbar/search/filter/sort server-state coordination |

### Typical Workflow

```bash
# 1. Plan — creates a new primary issue with sub-issues
/linear-workflow:plan-issue APP-123

# 2. Implement — orchestrates all sub-issues under the new primary
/linear-workflow:implement-issue APP-234
```

`implement-issue` self-detects whether the issue has children:

- **No children** → implements directly (branch, code, PR, review, CI, merge)
- **Has children** → spawns an agent-team, dispatches a teammate per sub-issue

## Prerequisites

- **shadcn/ui assistant skill** installed with `npx skills add shadcn/ui`.
  The `enterprise-k8s-frontend` and `tanstack-query-and-grid` skills rely on
  it for current shadcn project context, component docs, CLI behavior, and
  composition rules.
- **Linear MCP server** configured (see below)
- **GitHub CLI** (`gh`) authenticated with repo access
- **Git** with a clean working directory
- **Agent teams** enabled for parent-mode orchestration (see below)

### Enable Agent Teams

Parent-mode orchestration requires Claude Code agent-teams (experimental):

```bash
claude config set -g env.CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS 1
```

## Linear MCP Configuration

Both Claude and any external review tools need persistent access to Linear. The
key pattern is `mcp-remote` with a Linear API key managed through `direnv` and
your system keychain.

Load the API key securely via `direnv`. Add to your project `.envrc`:

```bash
export LINEAR_API_KEY="$(ks show LINEAR_API_KEY)"
```

This uses `ks` (keychain-store) to retrieve the key at shell init rather than storing it in plaintext.

Copy this to ~/.local/bin/linear-server and make executable.

```bash
#! /bin/bash
# Avoid reauth interruptions with an API key. install with:
#   claude mcp remove linear-server
#   claude mcp add linear-server --scope user -- ~/.local/bin/linear-server
set -euo pipefail
if [[ -z "${LINEAR_API_KEY}" ]]; then
  echo "empty LINEAR_API_KEY" >&2
  exit 1
fi
exec npx mcp-remote https://mcp.linear.app/mcp --header "Authorization: Bearer ${LINEAR_API_KEY}"
```

Then configure the MCP:

```bash
claude mcp add linear-server --scope user -- ~/.local/bin/linear-server
```

## Configuring Code Review

The `implement-issue` skill runs adversarial code review on every PR (up to 2 fix rounds + a final gate check). It looks for a review command in the target repo's `CLAUDE.md` or `AGENTS.md`. If none is found, it falls back to a Claude sub-agent review.

### How the skill resolves the review command

1. Look in `CLAUDE.md` or `AGENTS.md` for a section headed `## Code Review`
2. Find a fenced code block containing a shell command
3. Substitute `$PR_NUMBER`, `$BRANCH`, and `$REPO` into the command
4. Execute and parse stdout for: `APPROVE`, `REQUEST_CHANGES`, `[CRITICAL]`, `[IMPORTANT]`, `[STYLE]`

If no `## Code Review` section is found, the skill spawns a Claude sub-agent to
review the PR diff. This works everywhere but is less adversarial than using a
separate model.

### Example: Codex CLI for adversarial review (recommended)

Add this to your project's `CLAUDE.md`:

<pre>
## Code Review

```bash
codex --approval-mode full-auto --full-auto \
  "Review PR #$PR_NUMBER on branch $BRANCH in $REPO. \
   Focus on: security vulnerabilities, correctness bugs, error handling, \
   race conditions, missing validation, and test coverage gaps. \
   Report each finding on its own line as: \
   [SEVERITY] file:line — description \
   where SEVERITY is CRITICAL, IMPORTANT, or STYLE. \
   At the end, state APPROVE or REQUEST_CHANGES."
```
</pre>

This gives you true adversarial review — a separate model (Codex/GPT) reviewing
code written by Claude, catching blind spots that self-review misses.

#### Codex CLI setup

Install:

```bash
npm install -g @openai/codex
```

Configure `~/.codex/config.toml` for your LLM provider (e.g., LiteLLM, OpenAI
directly, Azure). See the [Codex CLI docs](https://github.com/openai/codex) for
provider-specific configuration.

The following example works with LiteLLM:

```toml
personality = "pragmatic"
model = "gpt-5.3-codex"
model_provider = "litellm"
model_reasoning_effort = "medium"

[notice.model_migrations]
"gpt-5.3-codex" = "gpt-5.4"

[model_providers.litellm]
name = "LiteLLM"
base_url = "https://litellm.example.com/v1"
env_key = "LITELLM_API_KEY"
wire_api = "responses"
```

### Example: Claude CLI for self-review

If you don't have Codex but want explicit review configuration:

<pre>
## Code Review

```bash
claude --print "Review the diff of PR #$PR_NUMBER in $REPO. \
  Run: gh pr diff $PR_NUMBER \
  Report findings as [CRITICAL], [IMPORTANT], or [STYLE]. \
  State APPROVE or REQUEST_CHANGES."
```
</pre>

This is less adversarial (same model family reviewing its own code) but still
provides structured review output.

### Fallback behavior

When no `## Code Review` section exists in the project, the skill spawns a Claude sub-agent that:

1. Reads the PR diff via `gh pr diff $PR_NUMBER`
2. Reviews every changed file
3. Reports findings using `[CRITICAL]`, `[IMPORTANT]`, `[STYLE]` severity levels
4. Returns `APPROVE` or `REQUEST_CHANGES`

This ensures the review loop works out of the box in any repo, with the adversarial quality improving when you configure an external reviewer.

## How It Works

### plan-issue

```
Original issue (rough idea)
  ├── Agent explores codebase
  ├── Creates NEW primary issue with plan + sub-issues
  │     ├── Sub-issue: phase 1
  │     ├── Sub-issue: phase 2
  │     └── Sub-issue: phase 3
  ├── Links original ←related→ new primary
  └── Marks original Done ("planning complete")
```

The original issue's task is "plan this feature." When planning completes, that task is done. The new primary issue tracks implementation.

### implement-issue (leaf mode)

```
Issue
  ├── Branch + implement + open PR
  ├── Code review round 1 → fix
  ├── Code review round 2 → fix
  ├── Code review FINAL → escalate or proceed
  ├── CI checks → fix or escalate
  ├── Merge
  └── Mark Done (or needs-human-review)
```

### implement-issue (parent mode)

```
Parent issue
  ├── List children (sub-issues)
  ├── Create agent-team with sequential tasks
  │
  ├── Teammate 1: implement-issue sub-issue-1 (leaf mode)
  │     └── branch → code → PR → review → fix → CI → merge
  ├── Teammate 2: implement-issue sub-issue-2 (leaf mode)
  │     └── branch → code → PR → review → fix → CI → merge
  ├── Teammate 3: implement-issue sub-issue-3 (leaf mode)
  │     └── ...
  │
  ├── Sweep for follow-up issues
  ├── Post summary with wall clock timing
  └── Mark parent Done (if all children complete)
```
