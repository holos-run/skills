---
name: implement-issue
description: Implement a Linear issue end-to-end. Handles both single issues (branch, code, PR, review, CI, merge) and parent issues with sub-issues (sub-agent orchestration over children). Use this skill when the user provides a Linear issue (URL or identifier like PLA-287) and asks to implement, work on, fix, or resolve it. Triggers on phrases like "implement issue", "work on this issue", "fix this issue", "implement linear plan", "execute linear plan", or when given a Linear issue identifier.
version: 2.6.0
---

# Implement Issue

Implement a Linear issue end-to-end. This skill self-detects whether the issue is a leaf issue (no children) or a parent issue (has children) and adapts its behavior:

- **Leaf mode**: Branch, implement, open PR, run adversarial code review (label-aware reviewer selection — Sonnet by default, Opus or Codex when explicitly labeled), up to 2 fix rounds, wait for CI, merge, and mark Done.
- **Parent mode**: Orchestrate implementation of all child issues with label-aware dispatch: Sonnet by default, Opus when explicitly labeled, Codex CLI when explicitly labeled, then track results, sweep for follow-ups, and post a summary.

Implement the Linear issue **{{SKILL_INPUT}}**.

## Linear Conventions

- **Issue** = Linear issue. Fetched via `mcp__linear-server__*` tools.
- **PR** = GitHub pull request. Opened via `gh`.
- The PR body must contain `Fixes <IDENTIFIER>` so Linear auto-closes the issue on merge.
- Always send real newlines in Linear `body` / `description` values — never `\n` escape sequences.

---

## Mode Detection

### 1. Parse Input and Fetch Issue

Parse `{{SKILL_INPUT}}` to extract the Linear issue identifier. Accept either form:

- Identifier: `APP-123`
- URL: `https://linear.app/<workspace>/issue/APP-123/<slug>`

Call `mcp__linear-server__get_issue` with `id: "<IDENTIFIER>"`.

Record:

- `ISSUE_ID` — Linear UUID
- `ISSUE_IDENTIFIER` — e.g., `APP-123`
- `ISSUE_TITLE`
- `ISSUE_URL`
- `TEAM_KEY` — e.g., `APP`
- `EXISTING_LABELS`
- `PARENT_IDENTIFIER` — if present, note it (this issue may be part of an implementation plan)

### 1a. Check for Blocking Issues

Before entering any implementation mode, check whether the issue is blocked by other open issues.

Call `mcp__linear-server__get_issue` with `id: "<ISSUE_ID>"` and inspect the response for blocking relations. Linear returns these as a `relations` array; each entry has a `type` field (`"blocked_by"` or `"blocks"`) and a `relatedIssue` object.

Collect every related issue whose type indicates it **blocks** the current issue (type is `"blocked_by"`, `"blocking"`, or equivalent as returned by the API). For each, record:

- `BLOCKER_IDENTIFIER` — e.g., `APP-99`
- `BLOCKER_ID` — UUID
- `BLOCKER_TITLE`
- `BLOCKER_STATUS_TYPE` — `triage`, `backlog`, `unstarted`, `started`, `completed`, or `canceled`

Filter to **active blockers**: those whose `BLOCKER_STATUS_TYPE` is NOT `completed` and NOT `canceled`.

**If active blockers exist:**

1. Post a comment on the issue via `mcp__linear-server__save_comment`:

   ```
   This issue is blocked by the following open issues and cannot be implemented yet:

   - <BLOCKER_IDENTIFIER>: <BLOCKER_TITLE> (status: <status>)
   [... one line per active blocker]

   Waiting for all blockers to reach Done or Canceled before proceeding.
   ```

2. **Poll until unblocked.** Repeat the following loop until all active blockers are resolved:

   a. Wait 60 seconds (`sleep 60`).
   b. For each remaining active blocker, call `mcp__linear-server__get_issue` with `id: "<BLOCKER_ID>"` and read its `statusType`.
   c. Remove any blocker from the active list whose `statusType` is now `completed` or `canceled`.
   d. If the active list is empty, break out of the loop.

3. After the loop exits (all blockers resolved), post a follow-up comment via `mcp__linear-server__save_comment`:

   ```
   All blocking issues are now Done or Canceled. Proceeding with implementation.
   ```

**If no active blockers:** continue immediately to step 2.

### 2. Check for Children

Call `mcp__linear-server__list_issues` with `parentId: "<ISSUE_ID>"`.

- **Children exist** → enter **Parent Mode** (jump to the Parent Mode section below)
- **No children** → enter **Leaf Mode** (continue to the next section)

---

## Leaf Mode

Full lifecycle for implementing a single issue with no children.

### L1. Start Wall Clock Timer

```bash
ISSUE_START_TIME=$(date +%s)
```

### L2. Create Branch

```bash
git fetch origin
git checkout main
git pull origin main
git checkout -b feat/<identifier-lowercased>-<slug>
```

Branch naming: `feat/<identifier-lowercased>-<slug>` where slug is the issue title in lowercase, spaces replaced by hyphens, special characters stripped, truncated to ~40 chars.

### L3. Announce on the Issue

Post a comment via `mcp__linear-server__save_comment`:

- `issue: "<ISSUE_IDENTIFIER>"`
- `body`:

```
Working on this issue.

- Branch: feat/<identifier-lowercased>-<slug>
```

Move the issue to In Progress and add the `implementing` label. Ensure the label exists for the team; create it if missing.

Call `mcp__linear-server__save_issue`:

- `issue: "<ISSUE_IDENTIFIER>"`
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

Refs: <ISSUE_IDENTIFIER>
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

Fixes <ISSUE_IDENTIFIER>

## Test plan
- [ ] <specific things to verify>

## Deferred Acceptance Criteria
- [ ] <AC from the issue NOT addressed in this PR>

EOF
)"
```

Use `ISSUE_IDENTIFIER` — the identifier of the issue currently being implemented. **Never add a `Fixes` line for the parent issue.** If this is a sub-issue, include only `Fixes <ISSUE_IDENTIFIER>` (the sub-issue), never `Fixes <PARENT_IDENTIFIER>`.

**Deferred Acceptance Criteria section**: Include only if at least one AC from the issue was not satisfied. If every AC is addressed, omit the entire heading. Presence of this section with non-empty bullets blocks the Done transition in step L12.

### L8. Code Review Loop

Run adversarial code review on the PR. Up to 2 fix rounds, then a final gate check.

#### L8a. Resolve the Review Command

Resolve which reviewer L8b will invoke. **L8a does not run the review** — it only selects the path and verifies prerequisites. L8b actually executes it.

Detect variables (used by all paths below). All shell snippets in this section are meant to be expanded by the runner's shell — `$PR_NUMBER`, `$BRANCH`, `$REPO` are real shell variables, not placeholders:

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
BRANCH=$(git rev-parse --abbrev-ref HEAD)
PR_NUMBER=$(gh pr list --state open --head "$BRANCH" --json number --jq '.[0].number')
```

**Reviewer selection (in priority order).** Lowercase the issue's `EXISTING_LABELS` (recorded in step 1) before matching:

1. **Issue labeled `codex`** → Codex CLI reviewer.
2. **Issue labeled `opus`** → Claude sub-agent with `model: "opus"`.
3. **Issue labeled `sonnet`** → Claude sub-agent with `model: "sonnet"`.
4. **No routing label, and project's `CLAUDE.md` or `AGENTS.md` contains a `## Code Review` section** with a fenced shell command → use that command.
5. **No routing label and no project config** → Claude sub-agent on `sonnet`.

If conflicting routing labels are present, prefer in this order: `codex` > `opus` > `sonnet`. **Routing labels always win over the project's `## Code Review` config.** When a routing label is present, the project config is not consulted — even if both happen to invoke the same backend (e.g., issue labeled `codex` and project config also runs `codex exec`), the routing-label path uses the prompt below, not the project's command.

**Codex reviewer (when issue is labeled `codex`):**

Verify the Codex CLI is on `PATH`:

```bash
command -v codex >/dev/null
```

If `codex` is **not** found, do NOT fall back to a Claude sub-agent. Surface an explicit error and escalate now (do not proceed to L8b–L8d):

```bash
gh pr comment $PR_NUMBER --body "$(cat <<'EOF'
## Code Review Cannot Proceed

This issue is labeled `codex`, but the `codex` CLI is not available on PATH in this environment. The reviewer cannot silently downgrade to another model. Marking for human review.
EOF
)"
gh pr edit $PR_NUMBER --add-label "needs-human-review"
```

Then call `mcp__linear-server__save_issue` with `issue: "<ISSUE_IDENTIFIER>"` and `labels: ["needs-human-review", ...existing]`. Skip directly to step L13 with result `ESCALATED`.

If `codex` is available, the command L8b will run is:

```bash
codex exec --dangerously-bypass-approvals-and-sandbox \
  "You are an adversarial code reviewer. Review the diff of PR #$PR_NUMBER in $REPO.

Run: gh pr diff $PR_NUMBER

Examine every changed file. Report findings using these severity levels:
- [CRITICAL] — security vulnerabilities, data loss, crashes, correctness bugs
- [IMPORTANT] — error handling gaps, race conditions, missing validation, test gaps
- [STYLE] — naming, formatting, dead code, minor improvements

At the end, state your verdict:
- APPROVE — if no critical or important findings
- REQUEST_CHANGES — if any critical or important findings exist

List each finding with file path, line number, severity, and description."
```

**Project-configured reviewer (no routing label, but project config provides a command):**

The project's `CLAUDE.md` or `AGENTS.md` may contain a section headed `## Code Review` with a fenced code block. The command is a template with these variables:

- `$PR_NUMBER` — the PR number
- `$BRANCH` — the current branch name
- `$REPO` — the repository in `owner/repo` format

Example from a project's CLAUDE.md:

<pre>
## Code Review

```bash
codex exec --dangerously-bypass-approvals-and-sandbox \
  "Review PR #$PR_NUMBER on branch $BRANCH in $REPO. \
   Report findings as [CRITICAL], [IMPORTANT], or [STYLE]. \
   Respond with APPROVE or REQUEST_CHANGES."
```
</pre>

If found, that command is what L8b will run, with variables resolved from the shell.

**Claude sub-agent reviewer (default fallback or explicit `sonnet`/`opus` label):**

L8b will spawn a Claude sub-agent for review:

```
Agent(
  description: "Code review PR #$PR_NUMBER",
  model: "<sonnet | opus>",
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
- **If style-only findings remain (no CRITICAL or IMPORTANT):** Proceed to step L9. Create a follow-up issue for style findings after merge (step L11).
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

3. Add `needs-human-review` label on the Linear issue (create if missing):

   Call `mcp__linear-server__save_issue` with `issue: "<ISSUE_IDENTIFIER>"` and `labels: ["needs-human-review", ...existing]`.

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

Before merging, handle remaining style-only findings from round 2 by creating a follow-up issue (step L11).

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

### L11. Follow-Up Issue for Style Findings

If style-only findings remain after round 2, create a follow-up Linear issue:

Call `mcp__linear-server__save_issue`:

- `team: "<TEAM_KEY>"`
- `parentId: "<PARENT_ID>"` if this issue has a parent (attach to same plan); otherwise omit
- `title: "fix: address review findings from PR #${PR_NUMBER}"`
- `description`:

```markdown
## Context

PR #<PR_NUMBER> (issue <ISSUE_IDENTIFIER>) was merged with style-only review findings remaining.

## Findings

<paste remaining style findings>
```

Record the follow-up identifier as `FOLLOW_UP_IDENTIFIER`.

### L12. AC-Gate and Close Issue

Before marking Done, check whether the PR lists deferred acceptance criteria:

```bash
PR_BODY=$(gh pr view $PR_NUMBER --json body --jq .body)
```

Parse for a `## Deferred Acceptance Criteria` heading with non-empty bullets.

**If deferred ACs exist:**

- Do NOT move to Done
- Add `needs-human-review` label on the issue
- Post a comment listing the deferred items
- Record result as MERGED_WITH_DEFERRED_ACS

**If no deferred ACs:**

Move to Done via `mcp__linear-server__save_issue`:

- `issue: "<ISSUE_IDENTIFIER>"`
- `state: "Done"`
- `labels: [... existing labels minus "implementing"]`

### L13. Post Summary

Calculate elapsed time and post a summary comment:

```bash
ISSUE_END_TIME=$(date +%s)
ELAPSED=$((ISSUE_END_TIME - ISSUE_START_TIME))
MINUTES=$((ELAPSED / 60))
SECONDS=$((ELAPSED % 60))
```

Call `mcp__linear-server__save_comment` with `issue: "<ISSUE_IDENTIFIER>"` and body:

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

Orchestrator for a parent issue with children. Uses label-aware dispatch to route each child issue to either a Claude Code sub-agent or the Codex CLI.

### P1. Start Wall Clock Timer

```bash
PLAN_START_TIME=$(date +%s)
```

### P2. List Children

Call `mcp__linear-server__list_issues` with `parentId: "<ISSUE_ID>"`, sorted by creation time ascending.

For each child, record:

- `identifier` (e.g., `PLA-301`)
- `id` (UUID)
- `title`
- `statusType` (`triage`, `backlog`, `unstarted`, `started`, `completed`, `canceled`)
- `labels` (names, lowercased for routing)

Skip any child whose status is `completed` or `canceled`.

If no open children remain, post a comment that all work is complete and stop.

### P3. Transition Labels

Replace `planning` with `implementing` on the parent issue.

Ensure the `implementing` label exists. Then call `mcp__linear-server__save_issue`:

- `issue: "<ISSUE_IDENTIFIER>"`
- `labels: ["implementing", ...existing labels minus "planning"]`

### P4. Dispatch Sub-Issues

Process sub-issues **sequentially** (each phase depends on the previous one). Before dispatching each open child, the orchestrator (this session) must inspect that sub-issue's labels and choose the implementation runner from those labels. The dispatched worker runs the full leaf lifecycle for that one sub-issue and returns a short result summary.

#### Pre-Dispatch: Detect and Discard Partial Work

Before choosing a runner for each sub-issue, check for leftover state from any prior attempt. Partial work is present when a local branch, remote branch, or open PR exists for the sub-issue.

Derive the branch prefix from the sub-issue identifier in lowercase (e.g., `app-123` → prefix `feat/app-123-`):

```bash
SUB_PREFIX="feat/<sub-identifier-lowercased>-"
SUB_BRANCH=$(git branch --list "${SUB_PREFIX}*" | head -1 | xargs)
SUB_OPEN_PR=$(gh pr list --state open \
  --json number,headRefName \
  --jq ".[] | select(.headRefName | startswith(\"${SUB_PREFIX}\")) | .number" \
  | head -1)
```

If partial work exists (`SUB_BRANCH` or `SUB_OPEN_PR` is non-empty):

1. Close the open PR without merging (if one exists):
   ```bash
   gh pr close "$SUB_OPEN_PR" --comment "Discarding partial work — restarting implementation from scratch." 2>/dev/null || true
   ```
2. Delete the remote branch:
   ```bash
   git push origin --delete "$SUB_BRANCH" 2>/dev/null || true
   ```
3. Delete the local branch:
   ```bash
   git branch -D "$SUB_BRANCH" 2>/dev/null || true
   ```
4. Return to a clean main:
   ```bash
   git checkout main && git pull origin main
   ```
5. Reset the sub-issue to its unstarted state and remove the `implementing` label:
   Call `mcp__linear-server__save_issue` with `issue: "<SUB_IDENTIFIER>"`, `state: "<unstarted-type state>"`, removing `implementing` from labels.
6. Post a comment on the sub-issue via `mcp__linear-server__save_comment`:
   ```
   Discarding partial work from a previous attempt. Starting a clean implementation.
   ```

**Dispatch selection per sub-issue:**

- Lowercase the child label names before matching.
- If the child has a `codex` label, use the **Codex CLI** instead of `Agent()`.
- Else if the child has an `opus` label, spawn a Claude sub-agent with model `opus`.
- Else if the child has a `sonnet` label, spawn a Claude sub-agent with model `sonnet`.
- Else default to **Sonnet**.

If conflicting routing labels are present (`codex`, `opus`, `sonnet`), prefer the most explicit non-Claude path first: `codex` > `opus` > `sonnet`.

If routing selects Claude, spawn a sub-agent:

```
Agent(
  description: "Implement <SUB_IDENTIFIER>",
  model: "<sonnet | opus>",
  prompt: "Invoke /linear-workflow:implement-issue <SUB_IDENTIFIER> to implement this sub-issue
  end-to-end. The skill handles branching, implementation, code review, CI, merge, and issue
  transitions. Run to completion. Return a short summary: result (MERGED |
  MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED), PR number, and any follow-up issue identifier."
)
```

If routing selects Codex, run the Codex CLI directly:

```bash
codex exec --dangerously-bypass-approvals-and-sandbox \
  "Invoke /linear-workflow:implement-issue <SUB_IDENTIFIER> to implement this sub-issue end-to-end.
The skill handles branching, implementation, code review, CI, merge, and issue transitions.
Run to completion. Return a short summary: result (MERGED | MERGED_WITH_DEFERRED_ACS |
ESCALATED | FAILED), PR number, and any follow-up issue identifier."
```

Wait for the dispatched worker to complete before starting the next one.

#### Handling Usage Limits

Usage limits apply when a runner's output indicates capacity is exhausted — for example output containing phrases like "usage limit reached", "rate limit exceeded", "quota exceeded", or "you have reached your usage limit" — without returning a valid implementation result (`MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED`). Do not count usage-limit events as stuck-worker retry attempts.

**If Codex hits a usage limit:**

1. Do not increment the retry counter.
2. Switch the runner for this sub-issue to **Opus** and dispatch a Claude sub-agent with `model: "opus"` using the same sub-issue identifier and prompt.

**If Opus hits a usage limit (either as the primary route or as a Codex fallback):**

1. Do not increment the retry counter.
2. Parse the earliest "retry after" time from all usage-limit messages (e.g., "available again at HH:MM UTC", "retry in N minutes", "resets at HH:MM"). Convert to seconds until that time.
3. If no retry time is parseable, default to 15 minutes (`900` seconds).
4. Wait (`sleep <seconds_until_retry>`) — do not prompt the user.
5. Re-dispatch the sub-issue using its original routing label selection (Codex if labeled `codex`, otherwise Opus, otherwise Sonnet).

Never prompt the user for guidance on usage limits — resolve autonomously.

#### Orchestrator Constraint: Never Take Over Implementation Work

**The orchestrator must never implement sub-issue work directly.** It does not write code, create commits, create branches, or perform any leaf-mode steps itself — not even to "help" a stuck worker finish. If a worker fails or gets stuck, the orchestrator's only permitted response is to clean up and dispatch a replacement worker using the same routing rules.

#### Detecting a Stuck Worker

A worker has **failed to complete** if it returns without a valid result summary (MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED) — for example, it got stuck mid-implementation and did not finish.

Use a retry loop with up to **3 total attempts** per sub-issue:

1. If the worker's returned output does not contain a valid result summary, it got stuck.
2. Note where it got stuck and why (e.g. "wrote files but did not commit", "opened PR but did not wait for CI").
3. Clean up:
   ```bash
   git checkout main
   git pull origin main
   ```
4. Re-check the sub-issue's labels, then dispatch a replacement worker with the same sub-issue identifier plus a warning. Do not describe implementation steps — just provide context and let the skill decide what to do.

   If the route is Claude:
   ```
   Agent(
     description: "Implement <SUB_IDENTIFIER> (retry <N>)",
     model: "<sonnet | opus>",
     prompt: "Invoke /linear-workflow:implement-issue <SUB_IDENTIFIER>.

     Warning: A previous attempt did not complete. Point: <e.g. 'wrote files but did not commit'>. Reason: <e.g. 'worker returned without a result summary'>.

     Invoke the skill and let it run to completion. Return a short result summary: result (MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED), PR number, and any follow-up issue identifier."
   )
   ```

   If the route is Codex:
   ```bash
   codex exec --dangerously-bypass-approvals-and-sandbox \
     "Invoke /linear-workflow:implement-issue <SUB_IDENTIFIER>.

Warning: A previous attempt did not complete. Point: <e.g. 'wrote files but did not commit'>.
Reason: <e.g. 'worker returned without a result summary'>.

Invoke the skill and let it run to completion. Return a short result summary: result (MERGED |
MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED), PR number, and any follow-up issue identifier."
   ```
5. Repeat until a valid result summary is returned or 3 attempts are exhausted.

**If all 3 attempts fail to return a valid result summary:**

a. Add `needs-human-review` to the sub-issue:
   Call `mcp__linear-server__save_issue` with `issue: "<SUB_IDENTIFIER>"` and `labels: ["needs-human-review", ...existing]`.

b. Add `needs-human-review` to the parent issue:
   Call `mcp__linear-server__save_issue` with `issue: "<ISSUE_IDENTIFIER>"` and `labels: ["needs-human-review", ...existing]`.

c. Post a comment on the parent issue via `mcp__linear-server__save_comment`:
   ```
   Sub-issue APP-123 could not be completed after 3 attempts — each attempt got stuck before finishing. Marked for human review.
   ```

d. **Abort** — stop the skill immediately. Do not process further sub-issues.

After each worker completes successfully, detect the result by checking the sub-issue's state in Linear (cross-check against the worker's returned summary):

| Sub-issue state | Labels | Result |
|------------------|--------|--------|
| `completed`-type (Done) | — | MERGED |
| `started`-type (In Progress) | has `needs-human-review` | Check comments for "Deferred Acceptance Criteria" → MERGED_WITH_DEFERRED_ACS; otherwise → ESCALATED |
| Any other state | — | FAILED |

Record per-sub-issue timing and results.

After each sub-issue, the working directory should be clean on main:

```bash
git checkout main
git pull origin main
```

### P5. Sweep for Follow-Up Issues

After all original children are processed, re-list children via `mcp__linear-server__list_issues` with `parentId: "<ISSUE_ID>"`.

Compare against the original list. Any new open child is a follow-up created during review.

Process follow-ups the same way — dispatch one worker per follow-up using the label-routing rules from P4.

### P6. Nested Parent Issues

If a sub-issue or follow-up is itself a parent (has its own children), the dispatched worker invoked in P4 will simply re-enter this skill in Parent Mode and orchestrate its own children with the same label-routing rules. Nested orchestration composes naturally — no special handling is needed for depth beyond re-checking labels at each parent.

By default, nested parent orchestrators should also run on **Sonnet**. Use **Opus** only when the nested parent issue is explicitly labeled `opus`; use **Codex CLI** when it is explicitly labeled `codex`.

### P7. Post Summary

Calculate total elapsed time:

```bash
PLAN_END_TIME=$(date +%s)
PLAN_ELAPSED=$((PLAN_END_TIME - PLAN_START_TIME))
PLAN_MINUTES=$((PLAN_ELAPSED / 60))
PLAN_SECONDS=$((PLAN_ELAPSED % 60))
```

Post a summary comment on the parent issue via `mcp__linear-server__save_comment`:

```
## Plan Execution Complete

Total wall clock time: <PLAN_MINUTES>m <PLAN_SECONDS>s

### Sub-Issues

- <SUB_IDENTIFIER> <title>: MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED
  - PR: #<PR_NUMBER>
  - Wall clock time: <minutes>m <seconds>s
  - Follow-up: <FOLLOW_UP_IDENTIFIER> (if any)
[...repeat for each sub-issue]

### Follow-Up Issues

- <FOLLOW_UP_IDENTIFIER> <title>: MERGED | ESCALATED | FAILED
  - PR: #<PR_NUMBER>
  - Wall clock time: <minutes>m <seconds>s
[...or "No follow-up issues were created."]
```

If any sub-issue is ESCALATED or MERGED_WITH_DEFERRED_ACS, add:

```
**Action required**: Some sub-issues need human attention before this plan can close.
```

### P8. Close Parent

Check all children via `mcp__linear-server__list_issues` with `parentId: "<ISSUE_ID>"`.

**If every child is `completed` or `canceled`:**

Call `mcp__linear-server__save_issue`:

- `issue: "<ISSUE_IDENTIFIER>"`
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
| PR lists deferred ACs (any mergeable scenario) | Merge PR, but leave issue In Progress + `needs-human-review` |
| CRITICAL/IMPORTANT unresolved after 2 fix rounds + final gate | Do NOT merge, `needs-human-review` |
| CI failures unresolved after 1 fix attempt | Do NOT merge, `needs-human-review` |

## Prerequisites

- **Linear MCP**: `mcp__linear-server__*` tools configured
- **GitHub CLI**: `gh` authenticated with repo access
- **Git**: Clean working directory
- **Code review tool**: Configured in project's CLAUDE.md (optional — falls back to Claude sub-agent)
- **Codex CLI**: Required only for sub-issues labeled `codex`

## Linear API Cheat Sheet

| Action | Tool | Key arguments |
|--------|------|---------------|
| Fetch issue (incl. blocking relations) | `mcp__linear-server__get_issue` | `id` |
| List children | `mcp__linear-server__list_issues` | `parentId` |
| Update issue | `mcp__linear-server__save_issue` | `issue`, `state` / `labels` / `description` |
| Create issue | `mcp__linear-server__save_issue` | `team`, `title`, `description` |
| Create sub-issue | `mcp__linear-server__save_issue` | `team`, `parentId`, `title`, `description` |
| Post comment | `mcp__linear-server__save_comment` | `issue`, `body` |
| List comments | `mcp__linear-server__list_comments` | `issueId` |
| List statuses | `mcp__linear-server__list_issue_statuses` | `team` |
| List / create labels | `mcp__linear-server__list_issue_labels` / `mcp__linear-server__create_issue_label` | `team` |
