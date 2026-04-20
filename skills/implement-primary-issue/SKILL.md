---
name: implement-primary-issue
description: Execute a full implementation plan from a parent Linear ticket with sub-tickets. Iterates over each sub-ticket, dispatches to /implement-sub-issue (which handles implementation, review, CI, and merge end-to-end), tracks wall clock timing, sweeps for follow-up tickets, and posts a summary. Triggers on phrases like "implement linear plan", "execute linear plan", "run the linear plan", "implement parent ticket", or when given a parent Linear ticket identifier with sub-tickets.
version: 2.1.0
---

# Implement Linear Plan

Orchestrator for a parent Linear ticket containing sub-tickets (Linear native parent/child). For each sub-ticket, the orchestrator **hands off** to the implementer Cyrus agent by applying the `role/implementer` label, then monitors the sub-ticket's Linear state, labels, and comments to detect completion, escalation, or stall. This skill does **not** implement sub-tickets in-process — that work is done by a separate Cyrus agent (or a local `/implement-sub-issue` invocation if Cyrus is not available).

See [HOL-724](https://linear.app/holos-run/issue/HOL-724) for the architectural rationale behind the handoff model (Option A: single Cyrus identity, role labels drive `labelPrompts`/`routingLabels`).

## Linear vs GitHub

Tickets live in Linear; code ships through GitHub PRs. This skill uses:

- `mcp__linear-server__*` tools for ticket discovery, labels, comments, and state monitoring
- A separate implementer agent (Cyrus or local `/implement-sub-issue`) performs all GitHub PR operations (branching, PR creation, review, CI, merge)

Key principles:

- **Discover sub-tickets via Linear parent/child**, not by parsing markdown task lists. Call `mcp__linear-server__list_issues` with `parentId: "<PARENT_UUID>"`.
- **Completion state comes from Linear workflow state**, not from checking a `[x]` box. A sub-ticket is "already done" if its state is `Done` or equivalent `completed`-type state.
- **Follow-up tickets are Linear sub-tickets** (same `parentId`), not GitHub issues. Follow-ups are labeled `role/maintainer` — not `role/implementer` — so a dedicated maintainer agent picks them up.
- **Handoff, not in-process dispatch.** Apply `role/implementer` to the next sub-ticket and poll Linear until it transitions out of `started`. The implementer agent posts progress comments on the sub-ticket as it works.

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

### 3. Transition Labels on the Parent Ticket

Replace `planning` with `implementing`, and `role/planner` with `role/orchestrator`, on the parent ticket to signal that execution has begun and the orchestrator (this skill) owns the parent while sub-tickets are handed off.

Ensure the following labels exist for the team (create any missing ones via `mcp__linear-server__create_issue_label`):

- `implementing`
- `role/orchestrator`
- `role/implementer` (needed for handoff in step 6)
- `role/maintainer` (needed for follow-up routing in step 8)

Then update the parent ticket's labels via `mcp__linear-server__save_issue`:

- `issue: "<PARENT_IDENTIFIER>"`
- `labels: ["implementing", "role/orchestrator", ...existing labels minus "planning" and minus "role/planner"]`

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

### 6. Iterate Over Sub-Tickets — Hand Off and Monitor

For each open sub-ticket in the queue, hand off to the implementer agent by applying the `role/implementer` label, then poll Linear until the sub-ticket reaches a terminal state. Sub-tickets are processed **sequentially** — only one `role/implementer` label is outstanding at a time, which keeps the implementer agent pool doing one sub-ticket at a time and avoids parallel merges.

Before starting each sub-ticket, record the start time and update the session name:

```bash
SUB_START_TIME=$(date +%s)
```

```
/rename <SUB_IDENTIFIER> <sub title> (plan <PARENT_IDENTIFIER>)
```

#### 6a. Hand Off: Apply `role/implementer` to the Sub-Ticket

Applying the `role/implementer` label is the handoff signal. The implementer Cyrus agent matches this label via its `routingLabels` config and picks up the sub-ticket. See [README — Multi-agent handoff](../../README.md) for the Cyrus config shape.

Post an orchestrator comment announcing the handoff, then apply the label.

Call `mcp__linear-server__save_comment` with `issue: "<SUB_IDENTIFIER>"` and body (real newlines):

```
[role=orchestrator slot=<SLOT> gen=1] Handing off to implementer agent.

- Parent plan: <PARENT_IDENTIFIER>
- Queue position: <N> of <TOTAL>
- Handoff signal: label `role/implementer`
```

Then call `mcp__linear-server__save_issue`:

- `issue: "<SUB_IDENTIFIER>"`
- `labels: ["role/implementer", ...existing labels]`

Strip any pre-existing `role/*` label except `role/implementer` before applying, so the ticket is routed to exactly one role.

**If Cyrus is not available in this environment** (e.g., local development, no webhook listener), the orchestrator falls back to an in-session dispatch: run `/implement-sub-issue <SUB_IDENTIFIER>` directly in this session instead of polling Linear. Detect this case by checking whether a comment from the implementer agent appears within `HANDOFF_ACK_TIMEOUT` (default 2 minutes); if not, assume no implementer agent is listening and fall back.

#### 6b. Monitor: Poll Linear Until Terminal State

Poll the sub-ticket every `POLL_INTERVAL` seconds (default 60). On each poll:

1. Call `mcp__linear-server__get_issue` with `id: "<SUB_IDENTIFIER>"` — record `state.type` and `updatedAt`.
2. Call `mcp__linear-server__list_comments` with `issueId: "<SUB_IDENTIFIER>"` — record the most recent comment's `createdAt`.
3. If the sub-ticket's `state.type` is `completed` or `canceled`, **exit the poll loop** — see 6c for result detection.
4. If the sub-ticket has the `needs-human-review` label, **exit the poll loop** — the implementer has escalated.
5. Compute `LAST_ACTIVITY = max(updatedAt, latest-comment-createdAt)`.
6. If `now - LAST_ACTIVITY > STALL_THRESHOLD` (default 30 minutes), post a stall-escalation comment on the sub-ticket (see 6d) and **exit the poll loop** with result `STALLED`.
7. Otherwise, sleep `POLL_INTERVAL` and repeat.

**Timing knobs (defaults, override with env vars if needed):**

| Name | Default | Purpose |
|---|---|---|
| `HANDOFF_ACK_TIMEOUT` | 120s | If implementer hasn't posted a "Working on this ticket." comment by this time, fall back to in-session dispatch |
| `POLL_INTERVAL` | 60s | How often the orchestrator checks Linear |
| `STALL_THRESHOLD` | 30m | No state change + no new comments for this long → STALLED |

Keep the poll loop tight — it should make exactly 2 Linear API calls per tick. Do not re-fetch the full comment list each tick; use `list_comments` with a sort/filter if available, or cache prior comment IDs and only log new ones.

#### 6c. Detect the Result

Once the poll loop exits, determine the outcome by reading the sub-ticket's current state:

| Sub-ticket state | Labels | Result |
|------------------|--------|--------|
| `completed`-type (Done) | — | MERGED |
| `started`-type (In Progress) | has `needs-human-review` | MERGED_WITH_DEFERRED_ACS or ESCALATED |
| Any other state, still labeled `role/implementer` | — | STALLED |
| `canceled`-type | — | CANCELED |

To distinguish MERGED_WITH_DEFERRED_ACS from ESCALATED when the ticket is `In Progress` with `needs-human-review`, scan the ticket's comments for the phrase "Merged With Deferred Acceptance Criteria". If found, it's MERGED_WITH_DEFERRED_ACS; otherwise ESCALATED.

Detect the PR number from the implementer's summary comment (`PR: #<number>`).

Record `SUB_END_TIME`, `SUB_RESULT`, `SUB_PR`, and any `FOLLOW_UP_IDENTIFIER` mentioned in the summary.

```bash
SUB_END_TIME=$(date +%s)
SUB_ELAPSED=$((SUB_END_TIME - SUB_START_TIME))
```

#### 6d. Stall Escalation Comment

If the poll loop exits with result `STALLED`, post a comment on the sub-ticket before moving on. Call `mcp__linear-server__save_comment` with `issue: "<SUB_IDENTIFIER>"` and body:

```
[role=orchestrator slot=<SLOT> gen=1] Sub-ticket stalled.

- No state change or new comments for <elapsed> minutes.
- Last activity: <LAST_ACTIVITY>
- Plan: <PARENT_IDENTIFIER>

Marking this sub-ticket `needs-human-review` and removing `role/implementer` to take it out of the implementer queue.
```

Then call `mcp__linear-server__save_issue`:

- `issue: "<SUB_IDENTIFIER>"`
- `labels: ["needs-human-review", ...existing labels minus "role/implementer"]`

### 7. Reset for Next Sub-Ticket

After the sub-ticket is processed, the orchestrator's working directory is unchanged — all code work happened in the implementer agent's worktree. Just move on to the next queued sub-ticket.

Return to step 6 to process the next open sub-ticket.

### 8. Re-List Children for Follow-Up Tickets (Route to Maintainer)

After all original sub-tickets have been processed, re-call `mcp__linear-server__list_issues` with `parentId: "<PARENT_ID>"` to discover any follow-up tickets created during the review cycle.

Compare against the original sub-ticket list. Any **new** open sub-ticket (not in the original list) is a follow-up ticket.

Follow-up tickets are routed to the **maintainer** agent, not the implementer. Maintainer agents are tuned for mechanical cleanup: stricter `allowedTools`, smaller PRs, no proto or schema changes. Follow-up tickets created by `/implement-sub-issue` for style-only review findings are already labeled `role/maintainer`. If a follow-up ticket is missing that label, apply it here before handing off.

For each follow-up ticket found, run the same handoff + monitor cycle as step 6, but:

- Apply `role/maintainer` instead of `role/implementer` in step 6a.
- Use a wider `POLL_INTERVAL` (default 120s) since maintainer work is typically shorter and lower priority.
- Track timing and results the same way as original sub-tickets.

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
[role=orchestrator slot=<SLOT> gen=1] ## Plan Execution Complete

Total wall clock time: <PLAN_MINUTES>m <PLAN_SECONDS>s
Orchestrator slot: <SLOT>

### Wall-Clock by Role

- Implementer agent: <total_minutes_m <seconds>s across <N> sub-tickets
- Maintainer agent: <total_minutes>m <seconds>s across <N> follow-ups
- Orchestrator overhead (handoff + polling): <minutes>m <seconds>s

### Sub-Tickets (Implementer)

- <SUB_IDENTIFIER> <title>: MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED | STALLED | CANCELED
  - PR: #<PR_NUMBER>
  - Wall clock time: <minutes>m <seconds>s
  - Follow-up: <FOLLOW_UP_IDENTIFIER> (if any)
  - Deferred ACs: <bulleted list> (only if MERGED_WITH_DEFERRED_ACS)
[…repeat for each sub-ticket]

### Follow-Up Tickets (Maintainer)

- <FOLLOW_UP_IDENTIFIER> <title>: MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED | STALLED
  - PR: #<PR_NUMBER>
  - Wall clock time: <minutes>m <seconds>s
[…or "No follow-up tickets were created during review."]

[If any escalated OR merged with deferred ACs OR stalled:]
**Action required**: Some sub-tickets need human attention before this plan can close:
- Sub-tickets labeled `needs-human-review` were escalated — review the critical/important findings (or the stall) on each.
- Sub-tickets marked `MERGED_WITH_DEFERRED_ACS` were merged but left `In Progress`; a human must either complete the deferred ACs, spin out a follow-up sub-ticket, or explicitly accept the gap before transitioning them to `Done`.
- Sub-tickets marked `STALLED` had no Linear activity for the stall window; check whether the implementer agent is healthy.
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

- **One sub-ticket at a time**: Only one `role/implementer` label is outstanding at any moment. No parallel PRs. The orchestrator applies `role/implementer` to the next queued sub-ticket only after the previous one reaches a terminal state.
- **Handoff, not in-process implementation**: The orchestrator never branches, never codes, never opens a PR. It applies labels, polls Linear, and aggregates results. The implementer Cyrus agent (or a local `/implement-sub-issue` fallback) does the work.
- **Follow-up routing**: Follow-up tickets created during review (by `/implement-sub-issue`) are labeled `role/maintainer`, not `role/implementer`, so a dedicated maintainer agent handles them. The follow-up sweep in step 8 hands these off the same way.
- **Escalation is permanent**: Once a sub-ticket gets `needs-human-review` (either from the implementer's review loop or from a stall-escalation here), the orchestrator does not re-attempt it. A human must intervene.
- **Wall clock timing**: Every sub-ticket, every follow-up, and the overall plan track wall clock time. The closing summary breaks out implementer vs. maintainer vs. orchestrator overhead.
- **Comment tagging**: Every orchestrator comment starts with `[role=orchestrator slot=<SLOT> gen=<N>]` so the handoff chain is reconstructible. Implementer and maintainer agents use `role=implementer` / `role=maintainer` respectively. `gen` increments on each handoff iteration of the same ticket (e.g., if a stalled ticket is later retried).
- **Linear markdown**: Send real newlines in all Linear `body` / `description` values — never `\n` escape sequences.

### Commit Conventions

- Implementation commits follow CONTRIBUTING.md format and include `Refs: <SUB_IDENTIFIER>` in the trailer
- Each review round is tracked in `tmp/review-pr/pr-<N>/round-<R>.md`

## Prerequisites

- **Claude Code**: For the orchestrator session (this skill runs here)
- **Self-hosted Cyrus agent(s)**: At minimum an implementer agent whose `routingLabels` includes `role/implementer`. A separate maintainer agent whose `routingLabels` includes `role/maintainer` is strongly recommended — without it, follow-up tickets will sit unpicked. See [README — Multi-agent handoff](../../README.md) for the Cyrus config shape.
- **Linear MCP**: `mcp__linear-server__*` tools authorized in the orchestrator's Claude Code session (for label/state/comment operations)
- **Fallback Codex CLI / GitHub CLI / make**: Only needed if the orchestrator falls back to in-session `/implement-sub-issue` (no Cyrus implementer available). Implementer-agent environments have these already.
- **Git**: Orchestrator can run in any directory — it never touches code

## Examples

```
# Execute a plan with 3 sub-tickets
/implement-primary-issue HOL-525

# Execute from a URL
/implement-primary-issue https://linear.app/openinfra/issue/HOL-525/send-id-token-instead-of-access-token
```

The skill will:

1. Fetch HOL-525, transition labels (`planning` → `implementing`, `role/planner` → `role/orchestrator`), list its children: HOL-601, HOL-602, HOL-603
2. Apply `role/implementer` to HOL-601 → implementer Cyrus agent picks it up → orchestrator polls Linear every 60s → HOL-601 reaches Done → MERGED
3. Apply `role/implementer` to HOL-602 → implementer picks up → orchestrator polls → HOL-602 gets `needs-human-review` → ESCALATED
4. Apply `role/implementer` to HOL-603 → implementer picks up → orchestrator polls → HOL-603 reaches Done with follow-up HOL-610 created → MERGED
5. Re-list children of HOL-525, find HOL-610 (labeled `role/maintainer` by the implementer) → poll until maintainer agent picks it up and completes → Done
6. Post summary with wall-clock timing (by role) on HOL-525; move HOL-525 to Done only if every child is `completed` or `canceled`

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
