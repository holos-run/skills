---
name: implement-primary-issue
description: Execute a full implementation plan from a parent Linear ticket with sub-tickets. Iterates over each sub-ticket, dispatches to /implement-sub-issue (which handles implementation, review, CI, and merge end-to-end), tracks wall clock timing, sweeps for follow-up tickets, and posts a summary. Triggers on phrases like "implement linear plan", "execute linear plan", "run the linear plan", "implement parent ticket", or when given a parent Linear ticket identifier with sub-tickets.
version: 2.0.0
---

# Implement Linear Plan

Orchestrator for a parent Linear ticket containing sub-tickets (Linear native parent/child). Each sub-ticket is dispatched to `/implement-sub-issue` which handles the full lifecycle end-to-end: branching, coding, testing, PR, code review, fix loops, CI, merge, and ticket state transitions. This skill only orchestrates: list children, dispatch, track timing, sweep for follow-ups, and post a summary.

## Linear vs GitHub

Tickets live in Linear; code ships through GitHub PRs. This skill uses:

- `mcp__linear-server__*` tools for ticket discovery, comments, state, and labels
- `/implement-sub-issue` handles all GitHub PR operations (branching, PR creation, review, CI, merge)

Key principles:

- **Discover sub-tickets via Linear parent/child**, not by parsing markdown task lists. Call `mcp__linear-server__list_issues` with `parentId: "<PARENT_UUID>"`.
- **Completion state comes from Linear workflow state**, not from checking a `[x]` box. A sub-ticket is "already done" if its state is `Done` or equivalent `completed`-type state.
- **Follow-up tickets are Linear sub-tickets** (same `parentId`), not GitHub issues.
- **Dispatch `/implement-sub-issue`** for each sub-ticket — it runs end-to-end including review, CI, merge, and ticket transitions.

Always send real newlines in Linear `body` / `description` values — never `\n` escape sequences.

## Arguments

`{{ARGUMENTS}}` is a Linear ticket identifier or URL:

- Identifier: `/implement-primary-issue HOL-525`
- URL: `/implement-primary-issue https://linear.app/<workspace>/issue/HOL-525/<slug>`

## Workflow

### 1. Start Wall Clock Timer and Resolve the Parent Ticket

Record the plan start time and parse `{{ARGUMENTS}}` to extract the Linear identifier.

```bash
PLAN_START_TIME=$(date +%s)
```

Call `mcp__linear-server__get_issue` with `id: "<PARENT_IDENTIFIER>"`.

Record:

- `PARENT_ID` — Linear UUID (used for `parentId` lookups on children)
- `PARENT_IDENTIFIER` — e.g., `HOL-525`
- `PARENT_TITLE`
- `TEAM_KEY` — e.g., `HOL`
- `EXISTING_LABELS` — current labels on the parent

Initialize a tracking structure for per-sub-ticket timing and results. For each sub-ticket processed, record:

- `SUB_START_TIME` — when work began on this sub-ticket
- `SUB_END_TIME` — when the sub-ticket was completed/escalated/failed
- `SUB_RESULT` — MERGED, MERGED_WITH_DEFERRED_ACS, ESCALATED, or FAILED
- `SUB_PR` — the PR number (if any)
- `FOLLOW_UP_IDENTIFIER` — follow-up Linear ticket identifier (if any)

### 2. Determine the Agent Slot

Worktrees are named `holos-console-agent-<N>`. Extract the slot:

```bash
SLOT=$(basename "$(pwd)" | sed -n 's/.*agent-\([0-9]\+\).*/agent-\1/p')
```

If `$SLOT` is empty, fall back to `SLOT="agent-unknown-$(basename "$(pwd)")"`.

### 3. Transition Labels: Remove `planning`, Add `implementing`

Replace the `planning` label with `implementing` on the parent ticket to signal that execution has begun.

Ensure the `implementing` label exists for the team. Call `mcp__linear-server__list_issue_labels` with `team: "<TEAM_KEY>"` to check. If missing, create it via `mcp__linear-server__create_issue_label` with `name: "implementing"` and `team: "<TEAM_KEY>"`.

Then update the parent ticket's labels via `mcp__linear-server__save_issue`:

- `issue: "<PARENT_IDENTIFIER>"`
- `labels: ["implementing", ...existing labels minus "planning"]`

### 4. Name the Session

Rename the current session so the human operator can see what this agent is working on:

```
/rename Plan <PARENT_IDENTIFIER> <parent ticket title>
```

For example: `/rename Plan HOL-525 Send id_token instead of access_token`.

### 5. List Sub-Tickets via Linear Parent/Child

Call `mcp__linear-server__list_issues` with:

- `parentId: "<PARENT_ID>"`
- Sort order: creation time ascending, so phases are processed in the order the planning skill created them.

For each returned sub-ticket, record:

- `identifier` (e.g., `HOL-601`)
- `id` (UUID)
- `title`
- `state.type` (`triage`, `backlog`, `unstarted`, `started`, `completed`, `canceled`)

Skip any sub-ticket whose `state.type` is `completed` or `canceled` — that work is already done (or abandoned). Build the ordered queue from the remaining sub-tickets.

If no sub-tickets are returned, the parent is not a plan — fall back to delegating directly to `/implement-sub-issue` with the parent identifier.

If every sub-ticket is already `completed`, post a comment on the parent noting that all work is complete and stop.

### 6. Iterate Over Sub-Tickets

For each open sub-ticket, dispatch to `/implement-sub-issue` which handles the full lifecycle. Process sub-tickets **sequentially** in creation order.

Before starting each sub-ticket, record the start time and update the session name:

```bash
SUB_START_TIME=$(date +%s)
```

```
/rename <SUB_IDENTIFIER> <sub title> (plan <PARENT_IDENTIFIER>)
```

#### 6a. Dispatch to /implement-sub-issue

Launch an Opus sub-agent to implement the sub-ticket end-to-end. The sub-agent invokes `/implement-sub-issue` which handles branching, coding, testing, opening a PR, code review, fix loops, CI, merge, and ticket state transitions.

```
Agent(
  description: "Implement ticket <SUB_IDENTIFIER>",
  model: "opus",
  prompt: "You are working in <working-directory>. Invoke the /implement-sub-issue skill with argument '<SUB_IDENTIFIER>' to implement Linear ticket <SUB_IDENTIFIER>. Follow all repository conventions in AGENTS.md. The skill handles the full lifecycle including review, CI, merge, and ticket transitions — run it to completion."
)
```

**Wait for the sub-agent to complete before proceeding.**

#### 6b. Detect the Result

After the sub-agent finishes, check the sub-ticket's state to determine the outcome:

Call `mcp__linear-server__get_issue` with `id: "<SUB_IDENTIFIER>"`.

Interpret the result:

| Sub-ticket state | Labels | Result |
|------------------|--------|--------|
| `completed`-type (Done) | — | MERGED |
| `started`-type (In Progress) | has `needs-human-review` | MERGED_WITH_DEFERRED_ACS or ESCALATED |
| Any other state | — | FAILED |

To distinguish MERGED_WITH_DEFERRED_ACS from ESCALATED when the ticket is `In Progress` with `needs-human-review`, check the ticket's comments (via `mcp__linear-server__list_comments`) for the phrase "Merged With Deferred Acceptance Criteria". If found, it's MERGED_WITH_DEFERRED_ACS; otherwise ESCALATED.

Also detect the PR number from the sub-ticket's comments (the `/implement-sub-issue` skill posts a summary comment with `PR: #<number>`).

Record `SUB_END_TIME`, `SUB_RESULT`, `SUB_PR`, and any `FOLLOW_UP_IDENTIFIER` mentioned in the summary.

```bash
SUB_END_TIME=$(date +%s)
SUB_ELAPSED=$((SUB_END_TIME - SUB_START_TIME))
```

### 7. Reset for Next Sub-Ticket

After the sub-ticket is processed, prepare the working directory for the next sub-ticket:

```bash
git checkout main
git pull origin main
```

Return to step 6 to process the next open sub-ticket.

### 8. Re-List Children for Follow-Up Tickets

After all original sub-tickets have been processed, re-call `mcp__linear-server__list_issues` with `parentId: "<PARENT_ID>"` to discover any follow-up tickets created during the review cycle.

Compare against the original sub-ticket list. Any **new** open sub-ticket (not in the original list) is a follow-up ticket.

For each follow-up ticket found, execute the same dispatch cycle (step 6). Track timing and results the same way as original sub-tickets.

Update the session name when processing follow-ups:

```
/rename <FOLLOW_UP_IDENTIFIER> <title> (follow-up, plan <PARENT_IDENTIFIER>)
```

If no follow-up tickets are found, proceed directly to step 9.

### 9. Completion — Post Summary with Wall Clock Timing

After all sub-tickets and follow-up tickets have been processed, calculate total elapsed time and post a comprehensive summary on the parent ticket. Call `mcp__linear-server__save_comment` with `issue: "<PARENT_IDENTIFIER>"` and body:

```bash
PLAN_END_TIME=$(date +%s)
PLAN_ELAPSED=$((PLAN_END_TIME - PLAN_START_TIME))
PLAN_MINUTES=$((PLAN_ELAPSED / 60))
PLAN_SECONDS=$((PLAN_ELAPSED % 60))
```

Body (real newlines, not `\n`):

```
## Plan Execution Complete

Total wall clock time: <PLAN_MINUTES>m <PLAN_SECONDS>s
Agent slot: <SLOT>

### Sub-Tickets

- <SUB_IDENTIFIER> <title>: MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED
  - PR: #<PR_NUMBER>
  - Wall clock time: <minutes>m <seconds>s
  - Follow-up: <FOLLOW_UP_IDENTIFIER> (if any)
  - Deferred ACs: <bulleted list> (only if MERGED_WITH_DEFERRED_ACS)
[…repeat for each sub-ticket]

### Follow-Up Tickets

- <FOLLOW_UP_IDENTIFIER> <title>: MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED
  - PR: #<PR_NUMBER>
  - Wall clock time: <minutes>m <seconds>s
[…or "No follow-up tickets were created during review."]

[If any escalated OR merged with deferred ACs:]
**Action required**: Some sub-tickets need human attention before this plan can close:
- PRs labeled `needs-human-review` were not merged — review the critical/important findings.
- Sub-tickets marked `MERGED_WITH_DEFERRED_ACS` were merged but left `In Progress`; a human must either complete the deferred ACs, spin out a follow-up sub-ticket, or explicitly accept the gap before transitioning them to `Done`.
```

Move the parent ticket to `Done` **only** when every child is `completed` or `canceled`. Explicitly, any child in `started` state (including sub-tickets with `SUB_RESULT=MERGED_WITH_DEFERRED_ACS`) blocks the parent transition — deferred ACs on a child mean the plan is not truly done, even if every PR merged.

Check remaining open children:

```bash
# Re-list children and count non-completed
# via mcp__linear-server__list_issues parentId=<PARENT_ID>
```

If every child is `completed` or `canceled`, call `mcp__linear-server__save_issue` with:

- `issue: "<PARENT_IDENTIFIER>"`
- `state: "Done"`
- `labels: [...existing minus "implementing"]`

Otherwise (children remain in `started` due to deferred ACs or escalation), leave the parent in its current state and add the `needs-human-review` label alongside `implementing` so it surfaces in triage queues.

## Guardrails

### Safety

- **One sub-ticket at a time**: Sub-tickets are processed sequentially. No parallel PRs.
- **Full delegation**: Each sub-ticket is handled end-to-end by `/implement-sub-issue`. The orchestrator does not interfere with review, CI, or merge decisions.
- **Follow-up sweep**: After all original sub-tickets are processed, the plan re-lists children and implements any follow-up tickets that were added during review.
- **Follow-up attachment**: Follow-up tickets are created as Linear sub-tickets of the same parent so step 8 can discover and implement them.
- **Escalation is permanent**: Once a sub-ticket gets `needs-human-review`, the orchestrator does not re-attempt it. A human must intervene.
- **Wall clock timing**: Every sub-ticket and the overall plan track wall clock time. The closing summary on the parent ticket includes timing for each item.
- **Linear markdown**: Send real newlines in all Linear `body` / `description` values — never `\n` escape sequences.

### Commit Conventions

- Implementation commits follow CONTRIBUTING.md format and include `Refs: <SUB_IDENTIFIER>` in the trailer
- Each review round is tracked in `tmp/review-pr/pr-<N>/round-<R>.md`

## Prerequisites

- **Claude Code**: With Agent tool and Opus model access
- **Codex CLI**: Required by the `/review-pr` skill (`npm install -g @openai/codex`)
- **GitHub CLI**: `gh` authenticated with repo access
- **Linear MCP**: `mcp__linear-server__*` tools authorized in the Claude Code settings
- **Git**: Clean working directory on `main` branch
- **make**: Build toolchain for `make generate`, `make test`

## Examples

```
# Execute a plan with 3 sub-tickets
/implement-primary-issue HOL-525

# Execute from a URL
/implement-primary-issue https://linear.app/openinfra/issue/HOL-525/send-id-token-instead-of-access-token
```

The skill will:

1. Fetch HOL-525, remove `planning` label, add `implementing` label, list its children: HOL-601, HOL-602, HOL-603
2. Dispatch HOL-601 to `/implement-sub-issue` → (implements, reviews, fixes, merges end-to-end) → Done
3. Dispatch HOL-602 to `/implement-sub-issue` → (implements, reviews, escalates) → needs-human-review
4. Dispatch HOL-603 to `/implement-sub-issue` → (implements, reviews, merges with follow-up HOL-610) → Done
5. Re-list children of HOL-525, find HOL-610 → dispatch to `/implement-sub-issue` → Done
6. Post summary with wall clock timing on HOL-525, move HOL-525 to Done (if all children done)

## Linear API Cheat Sheet

| Action | Tool | Key arguments |
|--------|------|---------------|
| Fetch a ticket | `mcp__linear-server__get_issue` | `id: "<IDENTIFIER>"` |
| List children of a ticket | `mcp__linear-server__list_issues` | `parentId: "<UUID>"` |
| Update state / labels / body | `mcp__linear-server__save_issue` | `issue`, `state` or `labels` or `description` |
| Create sub-ticket / follow-up | `mcp__linear-server__save_issue` | `team`, `parentId`, `title`, `description` |
| Post comment | `mcp__linear-server__save_comment` | `issue`, `body` |
| List comments | `mcp__linear-server__list_comments` | `issueId: "<IDENTIFIER>"` |
| Look up workflow states | `mcp__linear-server__list_issue_statuses` | `team: "<TEAM_KEY>"` |
| List / create labels | `mcp__linear-server__list_issue_labels` / `mcp__linear-server__create_issue_label` | `team: "<TEAM_KEY>"` |
