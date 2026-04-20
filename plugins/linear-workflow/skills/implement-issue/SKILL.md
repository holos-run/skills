---
name: implement-issue
description: Implement a Linear ticket end-to-end. Handles both single issues (branch, code, PR, review, CI, merge) and parent issues with sub-tickets (agent-team orchestration over children). Use this skill when the user provides a Linear ticket (URL or identifier like PLA-287) and asks to implement, work on, fix, or resolve it. Triggers on phrases like "implement ticket", "work on this ticket", "fix this ticket", "implement linear plan", "execute linear plan", or when given a Linear ticket identifier.
version: 2.0.0
---

# Implement Issue

Implement a Linear ticket end-to-end. This skill self-detects whether the ticket is a leaf issue (no children) or a parent issue (has children) and adapts its behavior:

- **Leaf mode**: Branch, implement, open PR, run adversarial code review (up to 2 fix rounds), wait for CI, merge, and mark Done.
- **Parent mode**: Orchestrate implementation of all child issues using an agent-team, track results, sweep for follow-ups, and post a summary.

Implement the Linear ticket **{{SKILL_INPUT}}**.

## Linear Conventions

- **Ticket** = Linear issue. Fetched via `mcp__linear-server__*` tools.
- **PR** = GitHub pull request. Opened via `gh`.
- The PR body must contain `Fixes <IDENTIFIER>` so Linear auto-closes the ticket on merge.
- Always send real newlines in Linear `body` / `description` values — never `\n` escape sequences.

---

## Mode Detection

### 1. Parse Input and Fetch Ticket

Parse `{{SKILL_INPUT}}` to extract the Linear ticket identifier. Accept either form:

- Identifier: `APP-123`
- URL: `https://linear.app/<workspace>/issue/APP-123/<slug>`

Call `mcp__linear-server__get_issue` with `id: "<IDENTIFIER>"`.

Record:

- `TICKET_ID` — Linear UUID
- `TICKET_IDENTIFIER` — e.g., `APP-123`
- `TICKET_TITLE`
- `TICKET_URL`
- `TEAM_KEY` — e.g., `APP`
- `EXISTING_LABELS`
- `PARENT_IDENTIFIER` — if present, note it (this ticket may be part of an implementation plan)

### 2. Check for Children

Call `mcp__linear-server__list_issues` with `parentId: "<TICKET_ID>"`.

- **Children exist** → enter **Parent Mode** (jump to the Parent Mode section below)
- **No children** → enter **Leaf Mode** (continue to the next section)

---

## Leaf Mode

Full lifecycle for implementing a single ticket with no children.

### L1. Start Wall Clock Timer

```bash
TICKET_START_TIME=$(date +%s)
```

### L2. Create Branch

```bash
git fetch origin
git checkout main
git pull origin main
git checkout -b feat/<identifier-lowercased>-<slug>
```

Branch naming: `feat/<identifier-lowercased>-<slug>` where slug is the ticket title in lowercase, spaces replaced by hyphens, special characters stripped, truncated to ~40 chars.

### L3. Announce on the Ticket

Post a comment via `mcp__linear-server__save_comment`:

- `issue: "<TICKET_IDENTIFIER>"`
- `body`:

```
Working on this ticket.

- Branch: feat/<identifier-lowercased>-<slug>
```

Move the ticket to In Progress and add the `implementing` label. Ensure the label exists for the team; create it if missing.

Call `mcp__linear-server__save_issue`:

- `issue: "<TICKET_IDENTIFIER>"`
- `state: "In Progress"`
- `labels: ["implementing", ...existing labels]`

If the team does not have a state named exactly `In Progress`, call `mcp__linear-server__list_issue_statuses` and pick the `started`-type state.

### L4. Read Project Conventions

Before implementing:

1. Read `CLAUDE.md` if it exists — project conventions, build commands, test strategy
2. Read `AGENTS.md` if it exists — architecture, package structure, indexed map of context.
3. Read `CONTRIBUTING.md` if it exists — commit message format

Follow whatever conventions the project specifies. Do not assume specific build tools, test frameworks, or file structures.

### L5. Implement

Follow the project's conventions for implementation:

1. Understand the existing patterns before writing new code
2. Write tests appropriate to the project's testing strategy
3. Make regular commits — each commit should be a logical unit
4. Run the project's test commands to verify your implementation
5. Run any code generation commands if applicable (e.g., if proto or schema files changed)

Commit messages should follow the project's format (from CONTRIBUTING.md or CLAUDE.md). Include the Linear identifier in the trailer:

```
<type>(<scope>): <short description>

Refs: <TICKET_IDENTIFIER>
```

### L6. Final Cleanup

Before opening the PR, scan for:

- Dead code introduced or made stale by the implementation
- Obsolete comments or outdated references
- Unused imports

Commit cleanup separately.

### L7. Open the PR

```bash
gh pr create --title "<concise title under 70 chars>" --body "$(cat <<'EOF'
## Summary
- <bullet points describing changes>

Fixes <TICKET_IDENTIFIER>

## Test plan
- [ ] <specific things to verify>

## Deferred Acceptance Criteria
- [ ] <AC from the ticket NOT addressed in this PR>

EOF
)"
```

Use the Linear identifier — not a GitHub issue number. If this ticket is a sub-ticket dispatched from a parent plan, use the sub-ticket identifier, not the parent.

**Deferred Acceptance Criteria section**: Include only if at least one AC from the ticket was not satisfied. If every AC is addressed, omit the entire heading. Presence of this section with non-empty bullets blocks the Done transition in step L12.

### L8. Code Review Loop

Run adversarial code review on the PR. Up to 2 fix rounds, then a final gate check.

#### L8a. Resolve the Review Command

Look in the project's `CLAUDE.md` or `AGENTS.md` for a section headed `## Code Review` that contains a fenced code block with a shell command. The command is a template with these variables:

- `$PR_NUMBER` — the PR number
- `$BRANCH` — the current branch name
- `$REPO` — the repository in `owner/repo` format

Example from a project's CLAUDE.md:

<pre>
## Code Review

```bash
codex --approval-mode full-auto --full-context \
  "Review PR #$PR_NUMBER on branch $BRANCH in $REPO. \
   Report findings as [CRITICAL], [IMPORTANT], or [STYLE]. \
   Respond with APPROVE or REQUEST_CHANGES."
```
</pre>

If found, use that command. Detect the variables:

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
BRANCH=$(git rev-parse --abbrev-ref HEAD)
PR_NUMBER=$(gh pr list --state open --head "$BRANCH" --json number --jq '.[0].number')
```

**Fallback** — if no `## Code Review` section is found in project config:

Spawn a Claude sub-agent for review:

```
Agent(
  description: "Code review PR #$PR_NUMBER",
  prompt: "You are an adversarial code reviewer. Review the diff of PR #$PR_NUMBER in $REPO.

Run: gh pr diff $PR_NUMBER

Examine every changed file. Report findings using these severity levels:
- [CRITICAL] — security vulnerabilities, data loss, crashes, correctness bugs
- [IMPORTANT] — error handling gaps, race conditions, missing validation, test gaps
- [STYLE] — naming, formatting, dead code, minor improvements

At the end, state your verdict:
- APPROVE — if no critical or important findings
- REQUEST_CHANGES — if any critical or important findings exist

List each finding with file path, line number, severity, and description."
)
```

#### L8b. Round 1: Review and Fix

Run the review command (or fallback agent). Parse the output for:

- **Verdict**: APPROVE or REQUEST_CHANGES
- **Finding counts** by severity: CRITICAL, IMPORTANT, STYLE

**If APPROVE (no findings):** Skip to step L9.

**If any findings:** Fix ALL findings — critical, important, and style. For each:

1. Read the cited file and line
2. Understand the issue
3. Apply the fix
4. Run tests to verify

After all fixes:

- Run the project's test suite to ensure nothing is broken
- Commit: `fix: address code review round 1 findings`
- Push

#### L8c. Round 2: Re-Review and Fix

Run the review command again. Parse the output.

- **If APPROVE:** Proceed to step L9.
- **If style-only findings remain (no CRITICAL or IMPORTANT):** Proceed to step L9. Create a follow-up ticket for style findings after merge (step L11).
- **If CRITICAL or IMPORTANT findings remain:** Fix all findings, commit, push. Proceed to L8d.

#### L8d. Final Review (Gate Check)

Run the review command one final time. Parse the output.

- **If APPROVE or style-only:** Proceed to step L9.
- **If CRITICAL or IMPORTANT findings still remain:** Escalate.

**Escalation:**

1. Post a summary comment on the PR listing unresolved findings:

   ```bash
   gh pr comment $PR_NUMBER --body "$(cat <<'EOF'
   ## Unresolved Critical/Important Findings

   After 2 review rounds, the following findings remain unresolved:

   <list each finding with file, line, and description>

   This PR requires human review before merge.
   EOF
   )"
   ```

2. Add `needs-human-review` label on the PR:

   ```bash
   gh pr edit $PR_NUMBER --add-label "needs-human-review"
   ```

3. Add `needs-human-review` label on the Linear ticket (create if missing):

   Call `mcp__linear-server__save_issue` with `issue: "<TICKET_IDENTIFIER>"` and `labels: ["needs-human-review", ...existing]`.

4. Do NOT merge. Skip to step L13 with result ESCALATED.

### L9. Wait for CI and Fix Failures

After code review is complete, wait for CI checks.

```bash
gh pr checks $PR_NUMBER --watch --fail-level all
```

If CI fails:

1. Read the CI logs to understand the failure
2. Fix the issue locally, run the project's test suite
3. Commit and push

If CI still fails after one fix attempt, escalate (add `needs-human-review` per step L8d) and skip to step L13 with result ESCALATED.

### L10. Merge

Before merging, handle remaining style-only findings from round 2 by creating a follow-up ticket (step L11).

```bash
gh pr merge $PR_NUMBER --merge --delete-branch
```

If merge fails due to conflicts:

```bash
git fetch origin
git rebase origin/main
git push --force-with-lease
gh pr merge $PR_NUMBER --merge --delete-branch
```

### L11. Follow-Up Ticket for Style Findings

If style-only findings remain after round 2, create a follow-up Linear ticket:

Call `mcp__linear-server__save_issue`:

- `team: "<TEAM_KEY>"`
- `parentId: "<PARENT_ID>"` if this ticket has a parent (attach to same plan); otherwise omit
- `title: "fix: address review findings from PR #${PR_NUMBER}"`
- `description`:

```markdown
## Context

PR #<PR_NUMBER> (ticket <TICKET_IDENTIFIER>) was merged with style-only review findings remaining.

## Findings

<paste remaining style findings>
```

Record the follow-up identifier as `FOLLOW_UP_IDENTIFIER`.

### L12. AC-Gate and Close Ticket

Before marking Done, check whether the PR lists deferred acceptance criteria:

```bash
PR_BODY=$(gh pr view $PR_NUMBER --json body --jq .body)
```

Parse for a `## Deferred Acceptance Criteria` heading with non-empty bullets.

**If deferred ACs exist:**

- Do NOT move to Done
- Add `needs-human-review` label on the ticket
- Post a comment listing the deferred items
- Record result as MERGED_WITH_DEFERRED_ACS

**If no deferred ACs:**

Move to Done via `mcp__linear-server__save_issue`:

- `issue: "<TICKET_IDENTIFIER>"`
- `state: "Done"`
- `labels: [... existing labels minus "implementing"]`

### L13. Post Summary

Calculate elapsed time and post a summary comment:

```bash
TICKET_END_TIME=$(date +%s)
ELAPSED=$((TICKET_END_TIME - TICKET_START_TIME))
MINUTES=$((ELAPSED / 60))
SECONDS=$((ELAPSED % 60))
```

Call `mcp__linear-server__save_comment` with `issue: "<TICKET_IDENTIFIER>"` and body:

```
## Implementation Complete

- PR: #<PR_NUMBER>
- Result: <MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED>
- Review rounds: <count>
- Wall clock time: <MINUTES>m <SECONDS>s
- Follow-up: <FOLLOW_UP_IDENTIFIER> (if any)
```

---

## Parent Mode

Orchestrator for a parent ticket with children. Uses Claude Code agent-teams to dispatch each child issue to a teammate.

### P1. Start Wall Clock Timer

```bash
PLAN_START_TIME=$(date +%s)
```

### P2. List Children

Call `mcp__linear-server__list_issues` with `parentId: "<TICKET_ID>"`, sorted by creation time ascending.

For each child, record:

- `identifier` (e.g., `PLA-301`)
- `id` (UUID)
- `title`
- `statusType` (`triage`, `backlog`, `unstarted`, `started`, `completed`, `canceled`)

Skip any child whose status is `completed` or `canceled`.

If no open children remain, post a comment that all work is complete and stop.

### P3. Transition Labels

Replace `planning` with `implementing` on the parent ticket.

Ensure the `implementing` label exists. Then call `mcp__linear-server__save_issue`:

- `issue: "<TICKET_IDENTIFIER>"`
- `labels: ["implementing", ...existing labels minus "planning"]`

### P4. Create Agent Team and Dispatch

Create an agent team to process sub-issues. sub-issues are processed **sequentially** (each phase depends on the previous one).

For each open child issue, create a task in the team with a sequential dependency on the previous task and any tasks explicitly noted as blocking dependencies in the parent ticket.

For each task, spawn a teammate (Opus) with instructions to invoke this skill on the sub-issue:

```
Spawn a teammate to implement <SUB_IDENTIFIER>.
The teammate should invoke /linear-workflow:implement-issue <SUB_IDENTIFIER> to implement
the ticket end-to-end. The skill handles branching, implementation, code review, CI, merge, and ticket transitions. Run to completion.
```

Wait for your teammates to complete their tasks before proceeding.

After each teammate completes, detect the result by checking the sub-ticket's state in Linear:

| Sub-ticket state | Labels | Result |
|------------------|--------|--------|
| `completed`-type (Done) | — | MERGED |
| `started`-type (In Progress) | has `needs-human-review` | Check comments for "Deferred Acceptance Criteria" → MERGED_WITH_DEFERRED_ACS; otherwise → ESCALATED |
| Any other state | — | FAILED |

Record per-sub-ticket timing and results.

After each sub-ticket, the working directory should be clean on main:

```bash
git checkout main
git pull origin main
```

### P5. Sweep for Follow-Up Tickets

After all original children are processed, re-list children via `mcp__linear-server__list_issues` with `parentId: "<TICKET_ID>"`.

Compare against the original list. Any new open child is a follow-up created during review.

Process follow-ups the same way (add tasks to the team, dispatch teammates).

### P6. Handle Nesting Beyond Depth 2

If a follow-up is itself a parent (has its own children), the teammate cannot create a nested agent-team (agent-teams don't nest). In this case, the teammate should use the `Agent()` tool to spawn a sub-agent for deeper orchestration:

```
Agent(
  description: "Implement nested parent <IDENTIFIER>",
  model: "opus",
  prompt: "Invoke /linear-workflow:implement-issue <IDENTIFIER> to implement this parent ticket
  and all its children. Run to completion."
)
```

### P7. Post Summary

Calculate total elapsed time:

```bash
PLAN_END_TIME=$(date +%s)
PLAN_ELAPSED=$((PLAN_END_TIME - PLAN_START_TIME))
PLAN_MINUTES=$((PLAN_ELAPSED / 60))
PLAN_SECONDS=$((PLAN_ELAPSED % 60))
```

Post a summary comment on the parent ticket via `mcp__linear-server__save_comment`:

```
## Plan Execution Complete

Total wall clock time: <PLAN_MINUTES>m <PLAN_SECONDS>s

### Sub-Tickets

- <SUB_IDENTIFIER> <title>: MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED
  - PR: #<PR_NUMBER>
  - Wall clock time: <minutes>m <seconds>s
  - Follow-up: <FOLLOW_UP_IDENTIFIER> (if any)
[...repeat for each sub-ticket]

### Follow-Up Tickets

- <FOLLOW_UP_IDENTIFIER> <title>: MERGED | ESCALATED | FAILED
  - PR: #<PR_NUMBER>
  - Wall clock time: <minutes>m <seconds>s
[...or "No follow-up tickets were created."]
```

If any sub-ticket is ESCALATED or MERGED_WITH_DEFERRED_ACS, add:

```
**Action required**: Some sub-tickets need human attention before this plan can close.
```

### P8. Close Parent

Check all children via `mcp__linear-server__list_issues` with `parentId: "<TICKET_ID>"`.

**If every child is `completed` or `canceled`:**

Call `mcp__linear-server__save_issue`:

- `issue: "<TICKET_IDENTIFIER>"`
- `state: "Done"`
- `labels: [... existing minus "implementing"]`

**Otherwise** (children remain in `started` due to deferred ACs or escalation):

Leave the parent in its current state. Add `needs-human-review` alongside `implementing` so it surfaces in triage queues.

---

## Merge Authority

| Situation | Action |
|-----------|--------|
| Clean review, CI green, no deferred ACs | Merge, Done |
| All findings fixed, clean re-review, CI green, no deferred ACs | Merge, Done |
| Style-only findings after round 2, CI green, no deferred ACs | Merge, create follow-up, Done |
| PR lists deferred ACs (any mergeable scenario) | Merge PR, but leave ticket In Progress + `needs-human-review` |
| CRITICAL/IMPORTANT unresolved after 2 fix rounds + final gate | Do NOT merge, `needs-human-review` |
| CI failures unresolved after 1 fix attempt | Do NOT merge, `needs-human-review` |

## Prerequisites

- **Linear MCP**: `mcp__linear-server__*` tools configured
- **GitHub CLI**: `gh` authenticated with repo access
- **Git**: Clean working directory
- **Agent teams**: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` (for parent mode)
- **Code review tool**: Configured in project's CLAUDE.md (optional — falls back to Claude sub-agent)

## Linear API Cheat Sheet

| Action | Tool | Key arguments |
|--------|------|---------------|
| Fetch ticket | `mcp__linear-server__get_issue` | `id` |
| List children | `mcp__linear-server__list_issues` | `parentId` |
| Update issue | `mcp__linear-server__save_issue` | `issue`, `state` / `labels` / `description` |
| Create issue | `mcp__linear-server__save_issue` | `team`, `title`, `description` |
| Create sub-ticket | `mcp__linear-server__save_issue` | `team`, `parentId`, `title`, `description` |
| Post comment | `mcp__linear-server__save_comment` | `issue`, `body` |
| List comments | `mcp__linear-server__list_comments` | `issueId` |
| List statuses | `mcp__linear-server__list_issue_statuses` | `team` |
| List / create labels | `mcp__linear-server__list_issue_labels` / `mcp__linear-server__create_issue_label` | `team` |
