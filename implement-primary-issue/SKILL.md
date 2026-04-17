---
name: implement-primary-issue
description: Execute a full implementation plan from a parent Linear ticket with sub-tickets. Iterates over each sub-ticket, implements it with a Claude Opus sub-agent via /implement-sub-issue, runs a fix-first code review loop using the review-pr skill, and merges or escalates. Implements follow-up tickets created during review. Posts wall clock timing summaries to each ticket. Triggers on phrases like "implement linear plan", "execute linear plan", "run the linear plan", "implement parent ticket", or when given a parent Linear ticket identifier with sub-tickets.
version: 1.0.0
---

# Implement Linear Plan

Automated implementation cycle for a parent Linear ticket containing sub-tickets (Linear native parent/child). Each sub-ticket is implemented by an Opus sub-agent via the `/implement-sub-issue` skill, reviewed by the `/review-pr` skill (Codex backend) in a sub-agent, fixed in-PR if findings exist, and merged or escalated — all without human intervention unless critical or important findings persist. After all sub-tickets are processed, the plan re-lists the parent's children to pick up follow-up tickets created during review and implements those too. Wall clock timing is tracked per sub-ticket and overall.

## Linear vs GitHub

Tickets live in Linear; code still ships through GitHub PRs. This skill uses:

- `mcp__linear-server__*` tools for ticket discovery, comments, state, and labels
- `gh` for PR operations (opened and merged by `/implement-sub-issue` and `/review-pr`)

Key differences from the GitHub `/implement-plan`:

- **Discover sub-tickets via Linear parent/child**, not by parsing markdown task lists. Call `mcp__linear-server__list_issues` with `parentId: "<PARENT_UUID>"`.
- **Completion state comes from Linear workflow state**, not from checking a `[x]` box. A sub-ticket is "already done" if its state is `Done` or equivalent `completed`-type state.
- **Follow-up tickets are Linear sub-tickets** (same `parentId`), not GitHub issues.
- **Dispatch `/implement-sub-issue`** (not `/implement-issue`).

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

Initialize a tracking structure for per-sub-ticket timing and results. For each sub-ticket processed, record:

- `SUB_START_TIME` — when work began on this sub-ticket
- `SUB_END_TIME` — when the sub-ticket was merged/escalated/failed
- `SUB_RESULT` — MERGED, MERGED_WITH_DEFERRED_ACS, ESCALATED, or FAILED
- `SUB_PR` — the PR number (if any)
- `FOLLOW_UP_IDENTIFIER` — follow-up Linear ticket identifier (if any)
- `DEFERRED_ACS` — array of deferred-AC bullets parsed from the PR body (empty if none)

### 2. Determine the Agent Slot

Worktrees are named `holos-console-agent-<N>`. Extract the slot:

```bash
SLOT=$(basename "$(pwd)" | sed -n 's/.*agent-\([0-9]\+\).*/agent-\1/p')
HOSTNAME=$(hostname)
```

If `$SLOT` is empty, fall back to `SLOT="agent-unknown-$(basename "$(pwd)")"`.

### 3. Name the Session

Rename the current session so the human operator can see what this agent is working on:

```
/rename Plan <PARENT_IDENTIFIER> <parent ticket title>
```

For example: `/rename Plan HOL-525 Send id_token instead of access_token`.

### 4. List Sub-Tickets via Linear Parent/Child

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

### 5. Iterate Over Sub-Tickets

For each open sub-ticket, execute the full implement-review-merge cycle (steps 6–10). Process sub-tickets **sequentially** in creation order.

Before starting each sub-ticket, record the start time and update the session name:

```bash
SUB_START_TIME=$(date +%s)
```

```
/rename <SUB_IDENTIFIER> <sub title> (plan <PARENT_IDENTIFIER>)
```

After each sub-ticket completes (merged or escalated), record the end time and result before moving to the next one:

```bash
SUB_END_TIME=$(date +%s)
SUB_ELAPSED=$((SUB_END_TIME - SUB_START_TIME))
```

### 6. Implement the Sub-Ticket

Launch an Opus sub-agent to implement the sub-ticket. The sub-agent invokes `/implement-sub-issue` which handles branching, coding, testing, and opening a draft PR.

```
Agent(
  description: "Implement ticket <SUB_IDENTIFIER>",
  model: "opus",
  prompt: "You are working in <working-directory>. Invoke the /implement-sub-issue skill with argument '<SUB_IDENTIFIER>' to implement Linear ticket <SUB_IDENTIFIER>. Follow all repository conventions in AGENTS.md. Do not merge the PR — stop after opening it."
)
```

**Wait for the sub-agent to complete before proceeding.**

After the sub-agent finishes, detect the PR number by filtering on the current branch name to avoid selecting an unrelated PR:

```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
PR_NUMBER=$(gh pr list --state open --head "$BRANCH" --json number --jq '.[0].number')
```

If no PR was found, the implementation sub-agent may have failed. Fall back to searching PRs by Linear identifier in their body:

```bash
PR_NUMBER=$(gh pr list --state open --json number,body --jq '.[] | select(.body | test("(Fixes|Closes|Resolves)\\s+<SUB_IDENTIFIER>")) | .number' | head -1)
```

If still not found, post a comment on the sub-ticket via `mcp__linear-server__save_comment` that implementation failed, mark the ticket result as FAILED, and continue to the next sub-ticket.

### 7. Wait for CI

After the PR is open, assess whether E2E tests are relevant to this change, then wait for the appropriate CI checks.

#### 7a. Assess E2E Relevance

Examine the PR's changed files:

```bash
CHANGED_FILES=$(gh pr diff $PR_NUMBER --name-only)
```

Apply the following decision table:

| Changed files match | E2E relevant? | Reason |
|---------------------|---------------|--------|
| `frontend/src/routes/**` | YES | Changes to existing UI routes affect user-facing behavior |
| `frontend/src/components/**` | YES | Changes to shared UI components may affect rendered pages |
| `frontend/src/lib/**` | YES | Changes to auth, transport, or query logic affect runtime behavior |
| `console/oidc/**` | YES | Changes to OIDC provider affect the login flow tested by E2E |
| Existing files in `console/rpc/**` | YES | Changes to existing RPC handlers may affect UI data flow |
| `console/console.go` | YES | Changes to server setup or route registration affect E2E |
| `frontend/e2e/**` | YES | Changes to E2E tests themselves should run E2E |
| `proto/**` (new messages only, no field changes to existing messages) | NO | New proto definitions do not affect existing UI behavior |
| `gen/**`, `frontend/src/gen/**` | NO | Generated code from proto changes; not hand-edited |
| New Go files (new packages, new handlers) | NO | New backend code has no existing UI integration to break |
| `docs/**` | NO | Documentation does not affect runtime behavior |
| `*_test.go`, `*.test.ts`, `*.test.tsx` | NO | Test-only changes do not affect runtime behavior |
| `.claude/**`, `.github/**` | NO | Tooling and CI config do not affect runtime behavior |
| `Makefile`, `*.md` | NO | Build and config files do not affect E2E behavior |
| `*.json` outside `frontend/` (e.g., `.claude/*.json`, root config) | NO | Non-frontend JSON config does not affect E2E behavior |
| `frontend/package.json`, `frontend/tsconfig*.json` | YES | Package deps and TypeScript config can affect runtime behavior |

**Tie-breaking rule**: If any changed file matches a YES row, E2E is relevant. Err on the side of waiting — a false positive costs 15 minutes; a false negative risks merging broken code.

Log the assessment reasoning so operators can audit the decision:

```bash
echo "E2E relevance assessment for PR #$PR_NUMBER:"
echo "Changed files:"
echo "$CHANGED_FILES"
echo ""
echo "Decision: E2E_RELEVANT=<yes|no>"
echo "Reason: <one-line explanation>"
```

#### 7b. Wait for CI Checks (Conditional)

**If E2E is relevant**, wait for all checks:

```bash
gh pr checks $PR_NUMBER --watch --fail-level all
```

**If E2E is NOT relevant**, wait only for `test` and `lint` checks, ignoring `e2e`:

```bash
CI_PASSED=false
API_ERRORS=0
while true; do
  CHECKS=$(gh pr checks $PR_NUMBER --json name,bucket 2>/dev/null)
  if [ $? -ne 0 ] || [ -z "$CHECKS" ]; then
    API_ERRORS=$((API_ERRORS + 1))
    echo "gh pr checks failed (attempt $API_ERRORS)"
    if [ "$API_ERRORS" -ge 5 ]; then
      echo "Too many consecutive API errors ($API_ERRORS) -- treating as CI failure"
      break
    fi
    sleep 30
    continue
  fi
  API_ERRORS=0

  TEST_BUCKET=$(echo "$CHECKS" | jq -r '.[] | select(.name == "test") | .bucket' 2>/dev/null)
  LINT_BUCKET=$(echo "$CHECKS" | jq -r '.[] | select(.name == "lint") | .bucket' 2>/dev/null)

  is_terminal() { [ "$1" = "pass" ] || [ "$1" = "fail" ] || [ "$1" = "cancel" ] || [ "$1" = "skipping" ]; }

  if is_terminal "$TEST_BUCKET" && is_terminal "$LINT_BUCKET"; then
    if [ "$TEST_BUCKET" = "pass" ] && [ "$LINT_BUCKET" = "pass" ]; then
      echo "test and lint passed -- proceeding without waiting for e2e"
      CI_PASSED=true
      break
    else
      echo "CI failed: test=$TEST_BUCKET lint=$LINT_BUCKET"
      break
    fi
  fi

  sleep 30
done
```

#### 7c. Fix CI Failures

For all-check mode, `gh pr checks --watch` exits non-zero on failure. For test/lint-only mode, check `CI_PASSED`. If CI failed in either mode, launch an Opus sub-agent to fix the failures:

```
Agent(
  description: "Fix CI for PR #<PR_NUMBER>",
  model: "opus",
  prompt: "You are working in <working-directory> on branch <branch>. PR #<PR_NUMBER> has failing CI checks. Run `gh pr checks <PR_NUMBER>` to see failures, then fix them. Run `make generate && make test` to verify locally before pushing. Commit fixes and push."
)
```

Re-check CI after the fix. If CI still fails after one fix attempt, add the `needs-human-review` label to the PR and to the Linear sub-ticket, and continue to the review step anyway (the review may catch the root cause).

**Note on skipped E2E**: If E2E was assessed as not relevant and later the `e2e` check fails independently, the CI Fix sub-agent should still attempt to fix it if encountered, but E2E failure alone does not block the review cycle when E2E was assessed as not relevant.

### 8. Review the PR (Round 1)

Launch a sub-agent to run the `/review-pr` skill. The review runs in a sub-agent to preserve the main agent's context window and to ensure the Codex review process is isolated.

```
Agent(
  description: "Review PR #<PR_NUMBER> round 1",
  prompt: "You are working in <working-directory> on the branch for PR #<PR_NUMBER>. Run the /review-pr skill with argument '<PR_NUMBER>'. Report the verdict (APPROVE or REQUEST_CHANGES), the count of critical/important/style findings, and the path to the review output file."
)
```

**Wait for the review sub-agent to complete.** Parse its response for:

- **Verdict**: APPROVE or REQUEST_CHANGES
- **Critical count**: Number of `[CRITICAL]` findings
- **Important count**: Number of `[IMPORTANT]` findings
- **Style count**: Number of `[STYLE]` findings

**If APPROVE (no findings):** Skip to step 10 (merge).

**If any findings exist (any severity):** Proceed to step 9 (fix all findings in-PR). Both `[CRITICAL]` and `[IMPORTANT]` findings are merge-blocking; only `[STYLE]` findings are fix-forward.

### 9. Fix-First Review Loop

The review follows a **fix-first** model. Round 1 findings are fixed directly in the PR — no follow-up tickets are created after round 1. Follow-up tickets are only created for findings that persist after round 2.

#### 9a. Fix ALL Findings (Round 1)

Read the review output to understand what needs fixing:

```bash
REVIEW_FILE="tmp/review-pr/pr-${PR_NUMBER}/round-1.md"
```

Launch an Opus sub-agent to fix **all** findings — critical, important, and style:

```
Agent(
  description: "Fix review findings PR #<PR_NUMBER> round 1",
  model: "opus",
  prompt: "You are working in <working-directory> on the branch for PR #<PR_NUMBER>.

The Codex code review (round 1) found issues that should be fixed in this PR. The review is at: <REVIEW_FILE>

Read the review file, then fix ALL findings — [CRITICAL], [IMPORTANT], and [STYLE]. For each fix:
1. Read the cited file and line
2. Understand the issue
3. Apply the concrete fix suggested (or a better one if you disagree -- leave a comment on the PR explaining why)
4. Run relevant tests to verify the fix

After all fixes:
- Run `make generate && make test` to ensure nothing is broken
- Commit: 'fix: address codex review round 1 findings'
- Push the fixes

Fix ALL findings, not just critical ones. The goal is to resolve everything in this PR."
)
```

**Wait for the fix sub-agent to complete.**

#### 9b. Re-Review (Round 2)

Launch another review sub-agent:

```
Agent(
  description: "Review PR #<PR_NUMBER> round 2",
  prompt: "You are working in <working-directory> on the branch for PR #<PR_NUMBER>. Run the /review-pr skill with argument '<PR_NUMBER>'. This is a re-review after fixes were applied. Report the verdict, finding counts, and output file path."
)
```

Parse the result:

- **If APPROVE (no findings):** Proceed to step 10 (merge).
- **If style-only findings remain (no CRITICAL or IMPORTANT):** Proceed to step 10 (merge with follow-up ticket).
- **If critical or important findings remain:** Proceed to step 9c (escalation).

#### 9c. Escalation After Round 2

If critical or important findings remain after round 2:

1. Post a summary comment on the PR listing the unresolved findings:

   ```bash
   gh pr comment $PR_NUMBER --body "$(cat <<'EOF'
   ## Unresolved Critical/Important Findings

   After 2 review rounds, the following critical or important findings remain unresolved:

   <list each unresolved critical/important finding with file, line, and description>

   This PR requires human review before merge.
   EOF
   )"
   ```

2. Add the `needs-human-review` label on the PR:

   ```bash
   gh pr edit $PR_NUMBER --add-label "needs-human-review"
   ```

3. Add the `needs-human-review` label on the Linear sub-ticket via `mcp__linear-server__save_issue` with `issue: "<SUB_IDENTIFIER>"` and `labels: ["needs-human-review", ...existing]`. Create the Linear label via `mcp__linear-server__create_issue_label` if it doesn't exist.

4. Post a comment on the Linear sub-ticket via `mcp__linear-server__save_comment` linking the PR and noting the escalation.

5. Do NOT merge the PR.

6. Continue to the next sub-ticket.

### 10. Merge the PR and Close the Linear Sub-Ticket

Before merging, handle any remaining style-only findings from round 2:

**If style-only findings remain after round 2** (no CRITICAL or IMPORTANT), create a follow-up **Linear sub-ticket attached to the parent plan**. Call `mcp__linear-server__save_issue` with:

- `team: "<TEAM_KEY>"`
- `parentId: "<PARENT_ID>"` — the plan's parent UUID so the follow-up is discoverable in step 12
- `title: "fix: address review findings from PR #${PR_NUMBER}"`
- `description`:

  ```markdown
  ## Context

  PR #${PR_NUMBER} (sub-ticket <SUB_IDENTIFIER> of plan <PARENT_IDENTIFIER>) was merged with style-only review findings that remain after round 2 fixes. Address these in a follow-up.

  ## Findings

  <paste remaining style findings from round 2 review output>

  ## Source

  Review output: `tmp/review-pr/pr-${PR_NUMBER}/round-2.md`

  Parent plan: <PARENT_IDENTIFIER>

  ## How to Implement

  Invoke the `/implement-sub-issue` skill with this ticket's Linear identifier.
  ```

Record the new sub-ticket's identifier as `FOLLOW_UP_IDENTIFIER`.

**Merge the PR:**

```bash
# Mark PR as ready (remove draft status)
gh pr ready $PR_NUMBER

# Merge with merge commit (not squash — preserves commit SHAs)
gh pr merge $PR_NUMBER --merge --delete-branch
```

Wait for the merge to complete. If it fails (e.g., merge conflict from another PR that landed), rebase and retry:

```bash
git fetch origin
git rebase origin/main
git push --force-with-lease
gh pr merge $PR_NUMBER --merge --delete-branch
```

After successful merge, run the **AC-gate** before transitioning the sub-ticket. Fetch the merged PR body and look for a `## Deferred Acceptance Criteria` heading (or case-insensitive variants: `Deferred ACs`, `Deferred Criteria`, `Deferred Items`). Collect the non-empty bullets under that heading (stop at the next `## ` heading or EOF); ignore empty bullets, template placeholders like `<...>`, and tokens like `TODO` / `N/A`.

```bash
PR_BODY=$(gh pr view $PR_NUMBER --json body --jq .body)
```

- **No deferred-AC bullets** → move the sub-ticket to `Done`. Call `mcp__linear-server__save_issue` with `issue: "<SUB_IDENTIFIER>"` and `state: "Done"` (or the `completed`-type state name for this team, via `mcp__linear-server__list_issue_statuses`). Record `SUB_RESULT=MERGED`.

- **At least one real deferred-AC bullet** → do NOT transition the sub-ticket to `Done`. Instead:

  1. Leave the sub-ticket in `In Progress`.
  2. Ensure the `needs-human-review` label exists for the team (create via `mcp__linear-server__create_issue_label` if missing), then call `mcp__linear-server__save_issue` with `issue: "<SUB_IDENTIFIER>"` and `labels: ["needs-human-review", ...existing labels]`.
  3. Post a comment on the sub-ticket via `mcp__linear-server__save_comment` summarizing the deferred items and linking the merged PR:

     ```
     ## Merged With Deferred Acceptance Criteria

     PR #<PR_NUMBER> was merged, but the PR body lists deferred ACs that this sub-ticket's state should not yet claim as complete. Leaving `In Progress` with `needs-human-review` so a human can triage whether to reopen scope here or spin out a follow-up sub-ticket of plan <PARENT_IDENTIFIER>.

     Deferred items from the PR:

     - <bullet 1>
     - <bullet 2>
     - ...

     PR: <PR_URL>
     ```

  4. Record `SUB_RESULT=MERGED_WITH_DEFERRED_ACS` (treat for reporting like `ESCALATED` — the parent plan must not treat the sub-ticket as cleanly complete in step 13).

### 11. Reset for Next Sub-Ticket

After merge, prepare the working directory for the next sub-ticket:

```bash
git checkout main
git pull origin main
```

Return to step 5 to process the next open sub-ticket.

### 12. Re-List Children for Follow-Up Tickets

After all original sub-tickets have been processed, re-call `mcp__linear-server__list_issues` with `parentId: "<PARENT_ID>"` to discover any follow-up tickets created during the review cycle (step 10).

Compare against the original sub-ticket list. Any **new** open sub-ticket (not in the original list) is a follow-up ticket.

For each follow-up ticket found, execute the same implement-review-merge cycle (steps 5–11). Track timing and results the same way as original sub-tickets.

Update the session name when processing follow-ups:

```
/rename <FOLLOW_UP_IDENTIFIER> <title> (follow-up, plan <PARENT_IDENTIFIER>)
```

If no follow-up tickets are found, proceed directly to step 13.

### 13. Completion — Post Summary with Wall Clock Timing

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
  - Review rounds: <count>
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
- `labels: [...existing minus "planning"]`

Otherwise (children remain in `started` due to deferred ACs or escalation), leave the parent in its current state and add the `needs-human-review` label to the parent as well so it surfaces in triage queues alongside the affected child(ren).

## Sub-Agent Isolation Model

Each sub-agent runs in the same working directory but is isolated by purpose:

| Sub-Agent | Model | Purpose | Reads | Writes |
|-----------|-------|---------|-------|--------|
| Implement | Opus | Code the sub-ticket via `/implement-sub-issue` | Ticket body, codebase | Branch, commits, PR, Linear comments/state |
| Review | Default | Run `/review-pr` (Codex) | PR diff, conventions | Review file, GitHub review |
| Fix | Opus | Fix review findings | Review file, codebase | Commits, push |
| CI Fix | Opus | Fix CI failures | CI output, codebase | Commits, push |

Using sub-agents preserves the orchestrator's context window. The orchestrator only tracks ticket state, PR numbers, and verdicts — not the full implementation or review details.

## Guardrails

### Merge Authority

| Situation | Action |
|-----------|--------|
| Clean review (APPROVE, no findings), no deferred ACs in PR body | Merge, move sub-ticket to Done |
| All findings fixed in round 1, clean re-review, no deferred ACs in PR body | Merge after clean re-review, move sub-ticket to Done |
| Style-only findings remain after round 2, no deferred ACs in PR body | Merge, create follow-up Linear sub-ticket attached to parent, move sub-ticket to Done |
| PR body lists deferred ACs (any scenario that would otherwise merge) | Merge the PR, but leave the sub-ticket in `In Progress` with `needs-human-review` and a deferred-items comment. Record `SUB_RESULT=MERGED_WITH_DEFERRED_ACS`. Parent plan stays open |
| Critical or important findings unresolved after round 2 | Do NOT merge, add `needs-human-review` label on PR and Linear sub-ticket |
| CI failures unresolved after 1 fix attempt | Add `needs-human-review` label on PR and Linear sub-ticket, continue to review |

### Safety

- **One sub-ticket at a time**: Sub-tickets are processed sequentially. No parallel PRs.
- **No force merges**: If a merge fails due to conflicts, rebase and retry once. If it fails again, escalate.
- **Review isolation**: The Codex review runs in a separate sub-agent process with no influence from the implementing agent. This ensures independent cross-model review.
- **Fix-first model**: Round 1 fix sub-agents address ALL findings (critical, important, and style) in the PR directly. Follow-up tickets are only created for style-only findings that remain after round 2. Critical and important findings that persist after round 2 trigger escalation.
- **AC-gate on sub-ticket `Done`**: Step 10 must parse the merged PR body for a `## Deferred Acceptance Criteria` (or equivalent) heading. Any non-empty bullet under that heading blocks the `Done` transition for that sub-ticket; instead, apply `needs-human-review`, keep the sub-ticket `In Progress`, and record `SUB_RESULT=MERGED_WITH_DEFERRED_ACS`. The parent plan must NOT move to `Done` in step 13 while any child is `MERGED_WITH_DEFERRED_ACS` — that counts as a non-completed child.
- **Follow-up attachment**: Follow-up tickets are created as Linear sub-tickets of the same parent so step 12 can discover and implement them.
- **Follow-up sweep**: After all original sub-tickets are processed, the plan re-lists children and implements any follow-up tickets that were added during review.
- **Escalation is permanent**: Once a PR gets `needs-human-review`, the agent does not re-attempt it. A human must intervene.
- **Wall clock timing**: Every sub-ticket and the overall plan track wall clock time. The closing summary on the parent ticket includes timing for each item.
- **Linear markdown**: Send real newlines in all Linear `body` / `description` values — never `\n` escape sequences.

### Commit Conventions

- Implementation commits follow CONTRIBUTING.md format and include `Refs: <SUB_IDENTIFIER>` in the trailer
- Fix commits: `fix: address codex review round <R> findings`
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

1. Fetch HOL-525, list its children: HOL-601, HOL-602, HOL-603
2. Implement HOL-601 → PR #50 → review (findings) → fix all in-PR → re-review (clean) → merge → move HOL-601 to Done
3. Implement HOL-602 → PR #51 → review → approve → merge → move HOL-602 to Done
4. Implement HOL-603 → PR #52 → review (findings) → fix in-PR → re-review (style-only remain) → merge + follow-up HOL-610 → move HOL-603 to Done
5. Re-list children of HOL-525, find HOL-610 → implement HOL-610 → PR #53 → review → merge → move HOL-610 to Done
6. Post summary with wall clock timing on HOL-525, move HOL-525 to Done, remove `planning` label

## Linear API Cheat Sheet

| Action | Tool | Key arguments |
|--------|------|---------------|
| Fetch a ticket | `mcp__linear-server__get_issue` | `id: "<IDENTIFIER>"` |
| List children of a ticket | `mcp__linear-server__list_issues` | `parentId: "<UUID>"` |
| Update state / labels / body | `mcp__linear-server__save_issue` | `issue`, `state` or `labels` or `description` |
| Create sub-ticket / follow-up | `mcp__linear-server__save_issue` | `team`, `parentId`, `title`, `description` |
| Post comment | `mcp__linear-server__save_comment` | `issue`, `body` |
| Look up workflow states | `mcp__linear-server__list_issue_statuses` | `team: "<TEAM_KEY>"` |
| List / create labels | `mcp__linear-server__list_issue_labels` / `mcp__linear-server__create_issue_label` | `team: "<TEAM_KEY>"` |
