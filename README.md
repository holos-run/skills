# Linear Workflow

Claude Code plugin for planning and implementing software with Linear and
GitHub.  Intended to work equally well with vanilla claude code running locally
or as a self hosted Linear Agent driving the claude code sdk command line tool.

## What This Does

This plugin provides two skills that drive an **adversarial
plan-implement-review cycle**. Linear is the source of truth for work; code
ships through GitHub PRs. The cycle works like this:

1. A feature is described as a **Linear ticket**.
2. **Plan** — An agent explores the codebase, creates a new primary issue with a structured plan and phased sub-tickets, then marks the original ticket Done.
3. **Implement** — For each sub-ticket, an agent-team orchestrator dispatches teammates to:
   - Implement the sub-issue (branch, code, tests, PR).
   - Run adversarial code review (round 1).
   - Fix round 1 findings.
   - Run adversarial code review (round 2).
   - Fix round 2 findings.
   - Run a final review gate — if critical/important findings remain, the issue enters `needs-human-review` and the skill stops.
   - Wait for CI; fix failures
   - Merge to base branch.
4. **Repeat** for all remaining sub-tickets, then sweep for follow-ups.
5. **Summary** posted back to the primary issue with wall clock timing.

The skills are **project-agnostic** — they read conventions (build commands,
test strategy, commit format) from the target repo's `CLAUDE.md` or `AGENTS.md`
rather than encoding them in the plugin.

## Installation

```bash
claude plugin marketplace add holos-run/skills
```
```bash
claude plugin install linear-workflow@holos-run
```

## Skills

| Skill                              | Purpose                                                                                       |
| ---------------------------------- | --------------------------------------------------------------------------------------------- |
| `/linear-workflow:plan-issue`      | Explore codebase, create a new primary issue with phased sub-tickets, relate back to original |
| `/linear-workflow:implement-issue` | Implement any Linear ticket: leaf issues directly, parent issues via agent-team orchestration |

### Typical Workflow

```bash
# 1. Plan — creates a new primary issue with sub-tickets
/linear-workflow:plan-issue APP-123

# 2. Implement — orchestrates all sub-tickets under the new primary
/linear-workflow:implement-issue APP-234
```

`implement-issue` self-detects whether the ticket has children:

- **No children** → implements directly (branch, code, PR, review, CI, merge)
- **Has children** → spawns an agent-team, dispatches a teammate per sub-ticket

## Prerequisites

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
codex --approval-mode full-auto --full-context \
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
Original ticket (rough idea)
  ├── Agent explores codebase
  ├── Creates NEW primary issue with plan + sub-tickets
  │     ├── Sub-ticket: phase 1
  │     ├── Sub-ticket: phase 2
  │     └── Sub-ticket: phase 3
  ├── Links original ←related→ new primary
  └── Marks original Done ("planning complete")
```

The original ticket's task is "plan this feature." When planning completes, that task is done. The new primary issue tracks implementation.

### implement-issue (leaf mode)

```
Ticket
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
Parent ticket
  ├── List children (sub-tickets)
  ├── Create agent-team with sequential tasks
  │
  ├── Teammate 1: implement-issue sub-ticket-1 (leaf mode)
  │     └── branch → code → PR → review → fix → CI → merge
  ├── Teammate 2: implement-issue sub-ticket-2 (leaf mode)
  │     └── branch → code → PR → review → fix → CI → merge
  ├── Teammate 3: implement-issue sub-ticket-3 (leaf mode)
  │     └── ...
  │
  ├── Sweep for follow-up tickets
  ├── Post summary with wall clock timing
  └── Mark parent Done (if all children complete)
```
