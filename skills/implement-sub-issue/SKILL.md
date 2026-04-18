---
name: implement-sub-issue
description: Implement a single Linear ticket end-to-end. Use this skill when the user provides a Linear ticket (URL or identifier like HOL-525) and asks to implement, work on, fix, or resolve it. Triggers on phrases like "implement ticket", "work on this ticket", "fix this ticket", or when given a Linear ticket identifier alone. Handles the full workflow: fetch ticket from Linear, create branch, comment on ticket, implement, open a GitHub PR, run code review, fix findings, wait for CI, merge, and update the Linear ticket. For parent tickets with sub-tickets, use /implement-primary-issue instead.
version: 1.0.0
---

# Implement Ticket

Full workflow for implementing a single Linear ticket: fetch the ticket, create a feature branch, announce on the ticket (with agent slot), implement using repository conventions, open a GitHub PR that links back to the Linear ticket, run code review (before waiting for CI), fix findings, wait for CI checks, merge, and move the Linear ticket to `Done`.

For parent Linear tickets with sub-tickets, use the `/implement-primary-issue` skill instead — it iterates over sub-tickets, implements each one via this skill, runs code review/fix loops, and merges.

## Linear vs GitHub

Tickets live in Linear. Code still ships through GitHub PRs. This skill bridges the two:

- "Ticket" = Linear issue. Fetched via `mcp__linear-server__*` tools.
- "PR" = GitHub pull request. Opened via `gh`.
- The PR body must contain a Linear magic-word reference (e.g., `Fixes HOL-525`) so Linear auto-closes the ticket on merge. Because GitHub has no native mechanism to close a Linear ticket, we also explicitly update the ticket state via `save_issue` after merge as a belt-and-suspenders safeguard.
- Never send `\n` escape sequences in Linear bodies — use real newlines.

## Workflow

### 0. Start Wall Clock Timer

Record the start time immediately so total elapsed time can be reported at the end:

```bash
TICKET_START_TIME=$(date +%s)
```

### 1. Parse the Input and Fetch the Ticket

Accept `{{SKILL_INPUT}}` in either form:

- Identifier: `HOL-525`
- URL: `https://linear.app/<workspace>/issue/HOL-525/<slug>`

Extract the identifier (e.g., `HOL-525`) and call `mcp__linear-server__get_issue` with `id: "<IDENTIFIER>"`.

Record:

- `TICKET_ID` — Linear UUID
- `TICKET_IDENTIFIER` — e.g., `HOL-525`
- `TICKET_TITLE`
- `TICKET_URL`
- `TICKET_STATE` — current workflow state (e.g., `Todo`, `In Progress`)
- `TEAM_KEY` — e.g., `HOL`
- `PARENT_ID` / `PARENT_IDENTIFIER` — if present, note it; this sub-ticket might be part of a plan

**Sub-ticket check:** call `mcp__linear-server__list_issues` with `parentId: "<TICKET_ID>"`. If the ticket has its own sub-tickets, **stop and inform the user** to use `/implement-primary-issue` instead. Do not implement a parent ticket directly.

### 2. Determine the Agent Slot

Worktrees are named `holos-console-agent-<N>`. Extract the slot:

```bash
SLOT=$(basename "$(pwd)" | sed -n 's/.*agent-\([0-9]\+\).*/agent-\1/p')
```

If `$SLOT` is empty, fall back to `SLOT="agent-unknown-$(basename "$(pwd)")"`.

### 3. Name the Session

Rename the current session so the human operator can see what this agent is working on:

```
/rename <TICKET_IDENTIFIER> <ticket title>
```

For example: `/rename HOL-525 Send id_token instead of access_token`.

### 4. Fetch Origin and Create Branch

```bash
git fetch origin
git checkout main
git pull origin main
git checkout -b <branch-name>
```

**Branch naming**: `feat/<identifier-lowercased>-<slug>` where `<slug>` is the ticket title converted to lowercase with spaces replaced by hyphens, truncated to ~40 chars. Examples:

- `HOL-525` "Send id_token instead of access_token" → `feat/hol-525-send-id-token-instead-of-access-token`
- `HOL-42` "Add Playwright E2E test infrastructure" → `feat/hol-42-add-playwright-e2e-test-infrastructure`

Strip special characters from the slug (keep only alphanumeric and hyphens).

### 5. Comment on the Ticket

Post a comment announcing which agent is working on this ticket. Call `mcp__linear-server__save_comment` with:

- `issue: "<TICKET_IDENTIFIER>"`
- `body`:

  ```
  Working on this ticket.

  - Agent slot: <SLOT>
  - Branch: <branch-name>
  ```

Real newlines — no `\n` escape sequences.

Also move the ticket into the `In Progress` workflow state and add the `implementing` label. Ensure the `implementing` label exists for the team; if missing, create it via `mcp__linear-server__create_issue_label` with `name: "implementing"` and `team: "<TEAM_KEY>"`.

Call `mcp__linear-server__save_issue` with:

- `issue: "<TICKET_IDENTIFIER>"`
- `state: "In Progress"`
- `labels: ["implementing", ...existing labels]`

If the team does not have a state named exactly `In Progress`, call `mcp__linear-server__list_issue_statuses` with `team: "<TEAM_KEY>"` and pick the `started`-type state.

### 6. Read and Understand the Codebase

Before implementing:

1. Read `AGENTS.md` (or `CLAUDE.md`) for project conventions
2. Read `CONTRIBUTING.md` if it exists for commit message requirements
3. Explore the relevant code areas mentioned in the ticket
4. Understand existing patterns before writing new code

### 7. Implement Using RED GREEN Approach

Follow the RED GREEN pattern from the repository's conventions:

1. **RED** — Write failing tests first that define the expected behavior
2. **GREEN** — Write the minimum implementation to make the tests pass
3. Make regular commits as you work — each commit should be a logical unit

Commit message format (check CONTRIBUTING.md for the specific format):

```
<type>(<scope>): <short description>

<longer explanation if needed>

Refs: <TICKET_IDENTIFIER>
```

Include the Linear identifier in the trailer (or body) so commits are traceable back to the ticket. Linear will surface the commit under the ticket via the GitHub integration.

Run the relevant test commands to verify your implementation:

- `make test` — all tests
- `make test-go` — Go tests
- `make test-ui` — UI unit tests
- `make generate` — regenerate code if proto/schema files changed

**Always run `make generate` before committing** if any generated files might be affected.

#### When to run `make test-e2e` locally

Before opening the PR, assess whether your changes warrant a local E2E test run. Use the E2E relevance decision table in step 12a of this skill to decide — that table is the single source of truth for which file patterns are E2E-relevant.

**Local E2E is optional and best-effort.** Running `make test-e2e` requires `make certs` and a k3d cluster. If the environment does not support E2E, skip the local run and note it in the PR description. Example:

```
> Local E2E was not run (no k3d cluster available). Relying on CI E2E check.
```

### 8. Final Cleanup Phase

Before opening the PR, scan for:

- Dead code introduced or made stale by the implementation
- Obsolete comments or outdated references
- Unused imports
- Stale documentation

Commit cleanup separately with a message explaining what was removed and why.

### 9. Open the PR

The PR must link back to the Linear ticket so Linear auto-transitions the ticket to `Done` on merge. Linear recognizes magic words like `Fixes`, `Closes`, and `Resolves` followed by a ticket identifier.

```bash
gh pr create --title "<concise title under 70 chars>" --body "$(cat <<'EOF'
## Summary
- <bullet points describing changes>

Fixes <TICKET_IDENTIFIER>

## Test plan
- [ ] <specific things to verify>

## Deferred Acceptance Criteria
- [ ] <AC from the ticket that was NOT addressed in this PR and needs follow-up>

Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Use the Linear identifier (e.g., `HOL-525`) — **not** a GitHub issue number. If this ticket is a sub-ticket dispatched from a parent plan, close the sub-ticket identifier here, not the parent.

**Deferred Acceptance Criteria section (important)**: Include this section **only** if at least one acceptance criterion from the Linear ticket was not fully satisfied by this PR. If every AC is addressed, omit the entire `## Deferred Acceptance Criteria` heading. Presence of this section with any non-empty bullet blocks the automatic `Done` transition in step 13 (AC-gate); the ticket will stay `In Progress` with a `needs-human-review` label so a human can triage the gap. Do not add the heading "just in case" — an empty or placeholder-only deferred section still triggers the gate if the bullets parse as non-empty.

### 10. Review the PR with Codex (Round 1)

Run a codex-based code review immediately after opening the PR — do NOT wait for CI checks first. Code review runs in parallel with CI to minimize wall clock time.

This skill invokes codex directly rather than delegating to a separate review skill, so it works in any repository with this plugin installed without requiring additional plugins.

#### 10a. Preflight

Verify codex is available:

```bash
if [ -x scripts/check-codex ]; then
  eval "$(scripts/check-codex)"
else
  CODEX=$(command -v codex)
fi
"$CODEX" --version 2>&1 || true
```

If `$CODEX` is empty, install codex (`npm install -g @openai/codex`) and authenticate (`codex login`). If codex still cannot be located, **abort** and escalate via step 11c (`needs-human-review`).

#### 10b. Resolve PR Context and Round Number

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
BRANCH=$(git rev-parse --abbrev-ref HEAD)
PR_NUMBER=$(gh pr list --state open --head "$BRANCH" --json number --jq '.[0].number')
BASE_BRANCH=$(gh pr view "$PR_NUMBER" --json baseRefName -q .baseRefName)

mkdir -p "tmp/review-pr/pr-${PR_NUMBER}"
ROUND=$(ls "tmp/review-pr/pr-${PR_NUMBER}"/round-*.md 2>/dev/null | wc -l | tr -d ' ')
ROUND=$((ROUND + 1))
OUTPUT_FILE="tmp/review-pr/pr-${PR_NUMBER}/round-${ROUND}.md"
```

#### 10c. Build the Acceptance Criteria Block

Fetch the Linear ticket body via `mcp__linear-server__get_issue` with `id: "<TICKET_IDENTIFIER>"` so codex can judge whether the PR satisfies the acceptance criteria. Construct:

```
ACCEPTANCE_CRITERIA="=== Acceptance Criteria (from <TICKET_IDENTIFIER>) ===
<ticket body>

"
```

If the ticket body is empty, leave `ACCEPTANCE_CRITERIA` empty.

#### 10d. Run Codex Review

```bash
timeout 300 "$CODEX" exec review \
  --ephemeral \
  -o "$OUTPUT_FILE" \
  "Review this pull request. Run \`git diff ${BASE_BRANCH}...HEAD\` to see the changes.

You are reviewing PR #${PR_NUMBER}. Read AGENTS.md (or CLAUDE.md) in the repo for project conventions before reviewing.

${ACCEPTANCE_CRITERIA}=== Review Checklist ===

Report ONLY violations you actually find in the diff. Do not report passing checks.

--- Critical (must fix before merge) ---

1. Security: hardcoded credentials, secrets, command injection, XSS, SQL injection, insecure defaults, missing input validation at system boundaries.
2. Reliability: data loss paths, cascading failures, resource leaks, race conditions, nil/null dereferences, off-by-one errors.
3. Generated code not edited: files under gen/ and frontend/src/gen/ must not be hand-edited.

--- Important (should fix) ---

4. Acceptance criteria: does the PR satisfy the linked ticket's acceptance criteria? Any missed requirements?
5. Test coverage: new behavior should have tests. Prefer unit tests over E2E where possible.
6. RED GREEN: tests should define and exercise expected behavior — flag tautological or never-executed tests.
7. Codegen consistency: if proto/CUE schemas changed, generated code should have corresponding changes.
8. Error handling: errors propagated, not swallowed.

--- Style (optional) ---

9. UI conventions (semantic CSS tokens, not hardcoded colors where applicable).
10. Dead code, unused imports, stale comments referencing removed behavior.

--- NOT in scope ---

Do NOT comment on style preferences beyond the list above, comment formatting, naming opinions, or scope creep beyond the ticket's acceptance criteria. This code is unreleased — do not flag breaking changes, removed exports, or renamed APIs as issues.

=== Output Format ===

## Review Summary

**PR**: #${PR_NUMBER}
**Round**: ${ROUND}
**Scope**: Branch diff against ${BASE_BRANCH}
**Verdict**: APPROVE | REQUEST_CHANGES

## Findings

### [CRITICAL] <title>
- **File**: <path>:<line>
- **Category**: security | reliability | generated-code
- **Issue**: <what is wrong>
- **Fix**: <concrete suggestion>

### [IMPORTANT] <title>
- **File**: <path>:<line>
- **Category**: acceptance-criteria | tests | red-green | codegen | error-handling
- **Issue**: <what is wrong>
- **Fix**: <concrete suggestion>

### [STYLE] <title>
- **File**: <path>:<line>
- **Category**: ui-conventions | dead-code
- **Issue**: <what is wrong>
- **Fix**: <concrete suggestion>

If no issues are found, output only:

## Review Summary

**PR**: #${PR_NUMBER}
**Round**: ${ROUND}
**Scope**: Branch diff against ${BASE_BRANCH}
**Verdict**: APPROVE

No issues found. The changes follow project conventions.

IMPORTANT: Be specific and concise. Cite exact file paths and line numbers. Only report actual issues found in the diff."
```

The `--ephemeral` flag prevents codex from persisting session files. The `-o` flag writes only the final review message to the output file. `timeout 300` caps execution at 5 minutes.

#### 10e. Parse the Results

Read `$OUTPUT_FILE` with the Read tool and extract:

- **Verdict**: search for `**Verdict**: APPROVE` or `**Verdict**: REQUEST_CHANGES`
- **Critical count**: occurrences of `### [CRITICAL]`
- **Important count**: occurrences of `### [IMPORTANT]`
- **Style count**: occurrences of `### [STYLE]`

If the output file is empty or missing, codex failed. Check `codex login status` and network connectivity, then escalate via step 11c (`needs-human-review`).

#### 10f. Post the Review to GitHub

Post the review output to the PR as a persistent audit trail. Choose the event based on severity:

```bash
EVENT="APPROVE"
if grep -qE '^### \[(CRITICAL|IMPORTANT)\]' "$OUTPUT_FILE"; then
  EVENT="REQUEST_CHANGES"
elif grep -qE '^### \[STYLE\]' "$OUTPUT_FILE"; then
  EVENT="COMMENT"
fi

REVIEW_BODY=$(cat "$OUTPUT_FILE")
gh api "repos/${REPO}/pulls/${PR_NUMBER}/reviews" \
  --method POST \
  --field event="${EVENT}" \
  --input <(jq -Rn --arg body "$REVIEW_BODY" '{body: $body, event: env.EVENT}') \
  || gh pr comment "$PR_NUMBER" --body "## Codex Review Round ${ROUND}

$REVIEW_BODY"
```

If the structured review POST fails, fall back to a plain PR comment so the review output is still visible.

**If APPROVE (no findings):** Skip to step 12 (wait for CI).

**If any findings exist (any severity):** Proceed to step 11 (fix findings).

### 11. Fix-First Review Loop

The review follows a **fix-first** model. Round 1 findings are fixed directly in the PR — no follow-up tickets are created after round 1. Follow-up tickets are only created for findings that persist after round 2.

#### 11a. Fix ALL Findings (Round 1)

Read the review output:

```bash
REVIEW_FILE="tmp/review-pr/pr-${PR_NUMBER}/round-1.md"
```

Fix **all** findings — critical, important, and style. For each finding:

1. Read the cited file and line
2. Understand the issue
3. Apply the concrete fix suggested (or a better one if you disagree — leave a comment on the PR explaining why)
4. Run relevant tests to verify the fix

After all fixes:

- Run `make generate && make test` to ensure nothing is broken
- Commit: `fix: address codex review round 1 findings`
- Push the fixes

#### 11b. Re-Review (Round 2)

Repeat the codex review procedure from step 10 (`10a` through `10f`). The round number in step 10b auto-increments based on the number of existing `round-*.md` files in `tmp/review-pr/pr-${PR_NUMBER}/`, so the new output will be written to `round-2.md`.

Parse the result:

- **If APPROVE (no findings):** Proceed to step 12 (wait for CI).
- **If style-only findings remain (no CRITICAL or IMPORTANT):** Proceed to step 12 (wait for CI). Create a follow-up Linear ticket for the remaining style findings after merge (step 13).
- **If critical or important findings remain:** Proceed to step 11c (escalation).

#### 11c. Escalation After Round 2

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

3. Also add a `needs-human-review` label on the Linear ticket (create the label if missing via `mcp__linear-server__create_issue_label`):

   Call `mcp__linear-server__save_issue` with `issue: "<TICKET_IDENTIFIER>"` and `labels: ["needs-human-review", ...existing]`.

4. Do NOT merge the PR. Skip to step 14 (post summary comment) with result ESCALATED.

### 12. Wait for CI Checks and Fix Failures

After code review is complete (and findings are fixed), wait for CI checks to pass.

#### 12a. Assess E2E Relevance

Examine the PR's changed files to determine whether E2E tests are relevant:

```bash
CHANGED_FILES=$(gh pr diff $PR_NUMBER --name-only)
```

Apply the following decision table:

| Changed files match | E2E relevant? | Reason |
|---------------------|---------------|--------|
| `frontend/src/routes/**` | YES | UI routes affect user-facing behavior |
| `frontend/src/components/**` | YES | Shared UI components may affect rendered pages |
| `frontend/src/lib/**` | YES | Auth, transport, or query logic affects runtime |
| `console/oidc/**` | YES | OIDC provider affects the login flow |
| Existing files in `console/rpc/**` | YES | Existing RPC handler changes may affect UI |
| `console/console.go` | YES | Server setup or route registration affects E2E |
| `frontend/e2e/**` | YES | E2E tests themselves should run E2E |
| `frontend/package.json`, `frontend/tsconfig*.json` | YES | Package deps and TS config affect runtime |
| `proto/**` (new messages only) | NO | New proto definitions don't affect existing UI |
| `gen/**`, `frontend/src/gen/**` | NO | Generated code, not hand-edited |
| New Go files (new packages, new handlers) | NO | No existing UI integration to break |
| `docs/**`, `*_test.go`, `*.test.ts`, `*.test.tsx` | NO | Docs and test-only changes |
| `.claude/**`, `.github/**`, `Makefile`, `*.md` | NO | Tooling and config |

**Tie-breaking**: If any changed file matches a YES row, E2E is relevant.

#### 12b. Wait for CI Checks

**If E2E is relevant**, wait for all checks:

```bash
gh pr checks $PR_NUMBER --watch --fail-level all
```

**If E2E is NOT relevant**, poll until test and lint checks reach a terminal state:

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

  TEST_BUCKET=$(echo "$CHECKS" | jq -r '.[] | select(.name == "Unit Tests") | .bucket' 2>/dev/null)
  LINT_BUCKET=$(echo "$CHECKS" | jq -r '.[] | select(.name == "Lint") | .bucket' 2>/dev/null)

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

#### 12c. Fix CI Failures

If CI failed, diagnose and fix the failures:

1. Run `gh pr checks $PR_NUMBER` to see which checks failed
2. Read the CI logs to understand the failure
3. Fix the issue locally, run `make generate && make test` to verify
4. Commit and push the fix

Re-check CI after the fix. If CI still fails after one fix attempt, add the `needs-human-review` label (both on the PR and on the Linear ticket, per step 11c) and skip to step 14 with result ESCALATED.

### 13. Merge the PR and Close the Ticket

Before merging, handle any remaining style-only findings from round 2:

**If style-only findings remain after round 2** (no CRITICAL or IMPORTANT), create a follow-up **Linear ticket** (not a GitHub issue):

Call `mcp__linear-server__save_issue` with:

- `team: "<TEAM_KEY>"`
- `parentId: "<PARENT_ID>"` if the ticket being implemented has a parent (so the follow-up attaches to the same plan); otherwise omit `parentId` and reference the original ticket in the body
- `title: "fix: address review findings from PR #${PR_NUMBER}"`
- `description`:

  ```markdown
  ## Context

  PR #${PR_NUMBER} (ticket <TICKET_IDENTIFIER>) was merged with style-only review findings remaining after round 2 fixes.

  ## Findings

  <paste remaining style findings from round 2 review output>

  ## Source

  Review output: `tmp/review-pr/pr-${PR_NUMBER}/round-2.md`
  ```

Record the new follow-up ticket's identifier (e.g., `HOL-612`) as `FOLLOW_UP_IDENTIFIER`.

**Merge the PR:**

```bash
# Mark PR as ready (remove draft status)
gh pr ready $PR_NUMBER

# Merge with merge commit (preserves commit SHAs)
gh pr merge $PR_NUMBER --merge --delete-branch
```

If the merge fails due to conflicts, rebase and retry:

```bash
git fetch origin
git rebase origin/main
git push --force-with-lease
gh pr merge $PR_NUMBER --merge --delete-branch
```

#### 13a. AC-Gate: Block `Done` When the PR Lists Deferred ACs

Before transitioning the ticket to `Done`, fetch the merged PR body and check whether it declares any deferred acceptance criteria. A deferred AC is any acceptance criterion from the Linear ticket that this PR did not satisfy and that the implementing agent flagged in the PR body.

```bash
PR_BODY=$(gh pr view $PR_NUMBER --json body --jq .body)
```

Parse `PR_BODY` for a heading that matches (case-insensitive): `## Deferred Acceptance Criteria`, `## Deferred ACs`, `## Deferred Criteria`, or `## Deferred Items`. Collect every non-empty bullet under that heading (stop at the next `## ` heading or EOF). Strip whitespace; ignore bullets whose text is empty, a placeholder like `<...>`, `TODO`, or `N/A`.

- **Zero real bullets under the heading (or no heading at all)** → no deferred ACs; proceed to the `Done` transition below.
- **One or more real bullets** → the PR shipped with an unresolved AC gap. Do **NOT** move the ticket to `Done`. Instead:

  1. Keep the ticket in `In Progress` (do not call `save_issue` with `state: "Done"`).
  2. Ensure a `needs-human-review` label exists for the team; create it via `mcp__linear-server__create_issue_label` if missing. Then call `mcp__linear-server__save_issue` with `issue: "<TICKET_IDENTIFIER>"` and `labels: ["needs-human-review", ...existing labels]`.
  3. Post a comment on the ticket via `mcp__linear-server__save_comment` with `issue: "<TICKET_IDENTIFIER>"` and body (real newlines):

     ```
     ## Merged With Deferred Acceptance Criteria

     PR #<PR_NUMBER> was merged, but the PR body lists deferred acceptance criteria that this ticket's state should not yet claim as complete. Leaving the ticket `In Progress` with `needs-human-review` so a human can triage whether to reopen scope here or spin out a follow-up ticket.

     Deferred items from the PR:

     - <bullet 1>
     - <bullet 2>
     - ...

     PR: <PR_URL>
     ```

  4. Record the ticket result as `MERGED_WITH_DEFERRED_ACS` (treat it like an escalation for reporting in step 14 — do not mark it MERGED). Proceed to step 14.

Only if the AC-gate passes (no deferred ACs) do the following:

Move the Linear ticket to `Done` (belt-and-suspenders — the `Fixes` magic word should do this too, but race conditions happen). Call `mcp__linear-server__save_issue` with:

- `issue: "<TICKET_IDENTIFIER>"`
- `state: "Done"`

If the team does not have a state named exactly `Done`, use `mcp__linear-server__list_issue_statuses` with `team: "<TEAM_KEY>"` and pick the `completed`-type state.

### 14. Post Summary Comment on the Ticket

Calculate elapsed time and post a summary comment on the Linear ticket. Call `mcp__linear-server__save_comment` with:

- `issue: "<TICKET_IDENTIFIER>"`
- `body`:

  ```
  ## Implementation Complete

  - PR: #<PR_NUMBER> (<PR_URL>)
  - Branch: <branch-name>
  - Result: <MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED>
  - Review rounds: <count>
  - Wall clock time: <minutes>m <seconds>s
  - Agent slot: <SLOT>
  [- Follow-up ticket: <FOLLOW_UP_IDENTIFIER> (style-only review findings)]
  [- Deferred ACs: <N> (ticket left In Progress with needs-human-review)]
  ```

Compute wall clock time:

```bash
TICKET_END_TIME=$(date +%s)
ELAPSED=$((TICKET_END_TIME - TICKET_START_TIME))
MINUTES=$((ELAPSED / 60))
SECONDS=$((ELAPSED % 60))
```

## Key Conventions

- **No backwards compatibility**: This code is not yet released; breaking changes are fine
- **RED GREEN**: Write tests before implementation
- **Regular commits**: Commit logical units as you go, not one giant commit at the end
- **make generate**: Always run before committing if proto or generated files are involved
- **E2E decision-making**: Assess whether `make test-e2e` is warranted using the E2E relevance heuristic (step 12a). Run it locally when relevant and the environment supports it; otherwise note the skip in the PR description
- **Cleanup phase**: Every implementation ends with a cleanup commit
- **Wall clock timing**: Record start time at step 0, post elapsed time in the summary comment at step 14
- **Review before CI**: Run code review immediately after opening the PR, before waiting for CI checks. This minimizes wall clock time by running review in parallel with CI
- **Fix-first model**: Fix ALL findings (critical, important, style) in-PR after round 1. Only create follow-up tickets for style-only findings that persist after round 2
- **Merge authority**: Merge after review is clean and CI passes. Escalate with `needs-human-review` (on both the PR and the Linear ticket) if critical/important findings persist after round 2 or CI fails after one fix attempt
- **Single tickets only**: If the ticket has sub-tickets of its own, stop and direct the user to `/implement-primary-issue`
- **Close the right ticket**: PRs must include `Fixes <TICKET_IDENTIFIER>` for the specific Linear ticket being implemented (the sub-ticket identifier, not the parent, when dispatched from a plan)
- **AC-gate on `Done`**: Never move a Linear ticket to `Done` when the merged PR body lists deferred acceptance criteria under a `## Deferred Acceptance Criteria` (or equivalent) heading. Leave the ticket `In Progress`, add `needs-human-review`, and post a comment linking the deferred items. Ticket state must match reality — a PR that shipped with an unresolved AC gap has not actually completed the ticket
- **Linear markdown**: Send real newlines in all Linear `body` / `description` values — never `\n` escape sequences

## Merge Authority

| Situation | Action |
|-----------|--------|
| Clean review (APPROVE, no findings), CI green, no deferred ACs in PR body | Merge, move ticket to Done |
| All findings fixed, clean re-review, CI green, no deferred ACs in PR body | Merge after clean re-review, move ticket to Done |
| Style-only findings remain after round 2, CI green, no deferred ACs in PR body | Merge, create follow-up Linear ticket, move ticket to Done |
| PR body lists deferred ACs (any scenario that would otherwise merge) | Merge the PR, but leave the Linear ticket in `In Progress` with `needs-human-review` and a comment listing the deferred items. Do NOT transition to Done |
| Critical or important findings unresolved after round 2 | Do NOT merge, add `needs-human-review` label on PR and ticket |
| CI failures unresolved after 1 fix attempt | Do NOT merge, add `needs-human-review` label on PR and ticket |

## Linear API Cheat Sheet

| Action | Tool | Key arguments |
|--------|------|---------------|
| Fetch ticket | `mcp__linear-server__get_issue` | `id: "<IDENTIFIER>"` |
| List children of a ticket | `mcp__linear-server__list_issues` | `parentId: "<TICKET_ID>"` |
| Update state / labels / body | `mcp__linear-server__save_issue` | `issue`, `state` or `labels` or `description` |
| Create follow-up ticket | `mcp__linear-server__save_issue` | `team`, `parentId`, `title`, `description` |
| Post comment | `mcp__linear-server__save_comment` | `issue`, `body` |
| Look up workflow states | `mcp__linear-server__list_issue_statuses` | `team: "<TEAM_KEY>"` |
| Create label | `mcp__linear-server__create_issue_label` | `name`, `team` |
