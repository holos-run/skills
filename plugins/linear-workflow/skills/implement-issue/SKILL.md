---
name: implement-issue
description: Implement a Linear issue end-to-end. Handles both single issues (branch, code, PR, review, CI, merge) and parent issues with sub-issues (sub-agent orchestration over children). Workers inherit the session-configured model by default; override with a --model argument or issue routing labels. Code review uses the opposite model family from the primary implementation runtime (the latest Claude Opus reviews Codex; the Codex frontier model reviews Claude) and posts findings back to the PR; override the reviewer with --reviewer. Use this skill when the user provides a Linear issue (URL or identifier like PLA-287) and asks to implement, work on, fix, or resolve it. Triggers on phrases like "implement issue", "work on this issue", "fix this issue", "implement linear plan", "execute linear plan", or when given a Linear issue identifier.
version: 2.15.0
---

# Implement Issue

Implement a Linear issue end-to-end. This skill self-detects whether the issue is a leaf issue (no children) or a parent issue (has children) and adapts its behavior:

- **Leaf mode**: Branch, implement, open PR, run adversarial cross-runtime code review (the latest Claude Opus reviews Codex implementations; the Codex frontier model reviews Claude implementations; `--reviewer` explicitly overrides), post the review back to the PR as a comment, up to 2 fix rounds, wait for CI, merge, and mark Done.
- **Parent mode**: Orchestrate implementation of all child issues — each worker inherits the session model by default, with `--model` argument or routing labels (`codex`, `fable`, `opus`, `sonnet`, `haiku`) overriding — then track results, sweep for follow-ups, and post a summary.

Implement the Linear issue **{{SKILL_INPUT}}**.

## Arguments and Model Selection

`{{SKILL_INPUT}}` contains the issue reference and optional flags:

```
<issue> [--model <fable|opus|sonnet|haiku|codex>] [--reviewer <codex|claude|fable|opus|sonnet|haiku>]
```

Parse and record:

- The issue reference (identifier or URL) — strip any flags before parsing it in step 1.
- `MODEL_OVERRIDE` — the value of `--model <name>` (also accepted as `model=<name>`) if present; otherwise unset.
- `REVIEWER_OVERRIDE` — the value of `--reviewer <name>` (also accepted as `reviewer=<name>`) if present; otherwise unset. Controls **only** the code reviewer selection in L8 — it never affects worker or orchestrator dispatch. `codex` selects the Codex frontier reviewer; `claude` or `opus` selects the latest Claude Opus through the `opus` alias; `fable`/`sonnet`/`haiku` explicitly select that Claude model instead.

Every point in this skill that dispatches **implementation work** — sub-issue workers (P6), nested orchestrators (P8), and retry dispatches — resolves the model with this priority. Code review is deliberately independent: L8 uses `REVIEWER_OVERRIDE` when present and otherwise selects the opposite model family from the detected primary runtime. `MODEL_OVERRIDE` and routing labels never select the reviewer.

1. **`MODEL_OVERRIDE`** — the `--model` argument always wins, over routing labels and project config alike.
2. **Issue routing label** — lowercase the issue's label names and match `codex`, `fable`, `opus`, `sonnet`, or `haiku`. If multiple are present, prefer `codex` > `fable` > `opus` > `sonnet` > `haiku`.
3. **Session default** — no override and no routing label: spawn the Claude sub-agent **without a `model` parameter** so it inherits the model configured for the Claude Code session (e.g., Fable or Opus in remote mode). This is the normal path — never hardcode a fallback model.

A resolution of `codex` dispatches via the Codex CLI; any other resolved name is passed verbatim as the `Agent()` `model` parameter. In the `Agent()` templates below, `model: "<RESOLVED_MODEL>"` means: pass the resolved name when priority 1 or 2 produced one, and **omit the `model` parameter entirely** when resolution fell through to the session default.

When this skill re-invokes itself for a sub-issue or nested parent, propagate `MODEL_OVERRIDE` and `REVIEWER_OVERRIDE` in the invocation when set: `/linear-workflow:implement-issue <SUB_IDENTIFIER> --model <name> --reviewer <name>` (include each flag only when its override is set). Routing labels need no propagation — each worker reads its own issue's labels.

## Linear Conventions

- **Issue** = Linear issue. Fetched via `mcp__linear-server__*` tools.
- **PR** = GitHub pull request. Opened via `gh`.
- The PR body must contain `Fixes <IDENTIFIER>` so Linear auto-closes the issue on merge.
- Always send real newlines in Linear `body` / `description` values — never `\n` escape sequences.

## Worktree Conventions

This skill runs inside a git worktree created by an agent harness — a Claude Code remote session, a local Claude Code worktree, Cyrus, or any similar tool that checks out a per-session branch in its own worktree. The skill makes no assumptions specific to any one harness. The conventions that follow from worktree use:

- **Never check out `main` locally.** A branch can only be checked out in one worktree at a time, and `main` belongs to the repo's primary worktree. Refresh state with `git fetch origin` and reference `origin/main` directly.
- **Create every feature branch from `origin/main`**, regardless of which branch the harness checked out in this worktree. This is the skill's own guarantee of a correct base branch — it does not rely on the harness's base-branch routing.
- **Session branch naming varies by harness.** Examples: `claude/<slug>` (Claude Code remote sessions), `cyrus/<identifier>-<slug>`, or `<user>/<identifier>-<slug>` (Linear's `gitBranchName`). Never assume a specific prefix — match a branch to an issue by checking whether the lowercased branch name contains the lowercased issue identifier.
- **Push with `git push origin HEAD`** (no `-u`) so upstream tracking stays on `origin/main`.

## Codex Frontier Model Resolution

Before this skill invokes `codex exec` for review or implementation, resolve `CODEX_FRONTIER_MODEL` once from the current Codex model catalog. Select the visible model whose description identifies it as the latest frontier model, preferring the lowest numeric priority:

```bash
CODEX_FRONTIER_MODEL=$(codex debug models | jq -er '
  [.models[]
   | select(.visibility == "list")
   | select(((.description // "") | ascii_downcase) | contains("latest frontier"))]
  | sort_by(.priority)
  | (.[0].slug // empty)
')
```

Require a non-empty result and pass `--model "$CODEX_FRONTIER_MODEL"` to every `codex exec` call. Resolve from the catalog at runtime instead of hardcoding a versioned model slug or inheriting a possibly stale configured default. If the frontier model cannot be resolved, treat the Codex CLI route as unavailable.

**Always run `codex exec` in the foreground and let the Bash call return — it returns exactly when Codex exits.** Never background it (no `run_in_background`, `&`, or `disown`) and never poll `ps`/`pgrep`/`jobs`/`/proc` to detect completion: process-table greps match the agent's own shell or the `grep` itself and deadlock the session until the harness force-recovers it. Bound long calls with `timeout`. The L8 Codex CLI reviewer (Method 1) applies this in full detail.

## Primary Runtime Detection

At the start of the skill, before dispatching any implementation or review worker, detect and record the immutable `PRIMARY_RUNTIME` for this invocation:

1. Ask the native host identity first. If it identifies itself as Codex, set `PRIMARY_RUNTIME=codex`. If it identifies itself as Claude Code, set `PRIMARY_RUNTIME=claude`. A known native identity is authoritative; do not inspect environment markers.
2. Only when native host identity is unavailable, inspect environment markers:
   - `CODEX_THREAD_ID` alone → `PRIMARY_RUNTIME=codex`.
   - `CLAUDE_CODE_ENTRYPOINT` or `CLAUDECODE` without `CODEX_THREAD_ID` → `PRIMARY_RUNTIME=claude`.
   - Conflicting or absent markers → `PRIMARY_RUNTIME=unknown`.
3. If it is unknown, use the explicit unknown-runtime resolution in L8.

The primary runtime is the agent that performs the implementation steps in this invocation, not a CLI binary it later launches. **Never detect the runtime with `command -v codex` or `command -v claude`**: both CLIs can be installed in either host. Detect once and do not recompute inside a review subprocess, because child processes can inherit the primary host's environment markers. In particular, a native Claude host launched beneath Codex remains `claude` even if it inherited `CODEX_THREAD_ID`.

Each leaf worker detects its own runtime. Parent orchestrators do not force their `PRIMARY_RUNTIME` onto child invocations; the worker selected in P6 becomes that child's primary implementation runtime.

## Claude Review Model Mapping

When a Codex implementation is reviewed by Claude, always invoke Claude Code with `--model opus`. Claude Code defines `opus` as the alias for the latest available Claude Opus model. Do not pin a version-specific Opus model, inherit the parent session model, or use a lower-cost fallback. Run the Claude CLI in the foreground, bound it with `timeout`, and capture its final text before continuing. Invoke it with `--print --output-format json` so success is machine-checkable from the result envelope, capture stderr to a file, and prove the CLI end-to-end with a cheap probe before the first review round. The L8 Claude reviewer applies this in full detail.

---

## Mode Detection

### 1. Parse Input and Fetch Issue

Parse `{{SKILL_INPUT}}` to extract the Linear issue identifier (after stripping the flags described in "Arguments and Model Selection"). Accept either form:

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

### 2. Check for Blocking Issues

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

**If no active blockers:** continue immediately to step 3.

### 3. Check for Children

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
git checkout -b feat/<identifier-lowercased>-<slug> origin/main
```

Branch naming: `feat/<identifier-lowercased>-<slug>` where slug is the issue title in lowercase, spaces replaced by hyphens, special characters stripped, truncated to ~40 chars.

**Never check out `main` locally.** Per Worktree Conventions, this skill runs in a Git worktree, and a branch can only be checked out in one worktree at a time — `main` belongs to the repo's primary worktree, not this one. Branch directly from `origin/main` instead, no matter which session branch the harness checked out here. The `git checkout -b <name> origin/main` form creates the branch from `origin/main`'s tip and sets the new branch's upstream tracking to `origin/main`.

Always push with `git push origin HEAD` (no `-u`) so the upstream tracking stays on `origin/main`. The PR is opened against `main` regardless.

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

**Deferred Acceptance Criteria section**: Include only if at least one AC from the issue was not satisfied. If every AC is addressed, omit the entire heading. Presence of this section with non-empty bullets blocks the Done transition in step L15.

### L8. Resolve the Review Command

Run adversarial code review on the PR. Up to 2 fix rounds, then a final gate check.

Resolve which reviewer L9 will invoke. **L8 does not run the review** — it only selects the path and verifies prerequisites. L9 actually executes it.

Detect variables (used by all paths below). All shell snippets in this section are meant to be expanded by the runner's shell — `$PR_NUMBER`, `$BRANCH`, `$REPO` are real shell variables, not placeholders:

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
BRANCH=$(git rev-parse --abbrev-ref HEAD)
PR_NUMBER=$(gh pr list --state open --head "$BRANCH" --json number --jq '.[0].number')
```

**Reviewer selection (in priority order):**

1. **`REVIEWER_OVERRIDE` is set** → honor it. `codex` selects the Codex frontier reviewer. `claude` or `opus` selects the latest Claude Opus through `--model opus`. `fable`/`sonnet`/`haiku` selects that explicit Claude model.
2. **`PRIMARY_RUNTIME=codex`** → select the latest Claude Opus through `--model opus`.
3. **`PRIMARY_RUNTIME=claude`** → select the latest Codex frontier reviewer resolved from the current Codex model catalog.
4. **`PRIMARY_RUNTIME=unknown` and project config contains a reviewer command** → use the fenced shell command from the project's `CLAUDE.md` or `AGENTS.md` `## Code Review` section.
5. **`PRIMARY_RUNTIME=unknown` and no project reviewer exists** → post a `Code Review Cannot Proceed` comment, add `needs-human-review` to the PR and Linear issue, and skip to L16 with result `ESCALATED`. Do not guess and risk same-runtime self-review.

`MODEL_OVERRIDE` and issue routing labels control implementation dispatch only. They must not influence L8. This separation is what guarantees adversarial review when, for example, a `codex` label sends implementation to Codex: the resulting Codex leaf invocation detects itself and selects the latest Claude Opus for review.

**Posting the review back to the PR (all reviewer paths).** Every review round must end with the reviewer's findings posted as a comment on the PR. For the Codex CLI, Codex MCP, project-configured, and Claude CLI paths, after parsing the output post it:

```bash
gh pr comment $PR_NUMBER --body "$(cat <<EOF
## Code Review — Round <N> (<reviewer: codex | claude | model name | project-configured>)

<the reviewer's full findings and verdict, verbatim>
EOF
)"
```

If `REVIEWER_OVERRIDE` deliberately selects the same model family as `PRIMARY_RUNTIME`, add this line below the PR comment heading for every reviewer path: `Same-family review explicitly requested via --reviewer; cross-runtime pairing was overridden.`

**Codex frontier reviewer (`PRIMARY_RUNTIME=claude` by default, or selected via `--reviewer codex`):**

The shared review prompt for CLI, MCP, and project-configured invocation methods is:

```
You are an adversarial code reviewer. Review the diff of PR #$PR_NUMBER in $REPO. For CLI invocations, the complete diff follows this prompt on stdin; review that supplied diff directly and do not re-fetch it. For remote reviewer methods that do not receive stdin, fetch the diff with `gh pr diff $PR_NUMBER`.

Examine every changed file. Report findings using these severity levels:
- [CRITICAL] — security vulnerabilities, data loss, crashes, correctness bugs
- [IMPORTANT] — error handling gaps, race conditions, missing validation, test gaps
- [STYLE] — naming, formatting, dead code, minor improvements

End your review with a final line containing exactly `Verdict: APPROVE` (no critical or important findings) or `Verdict: REQUEST_CHANGES` (any critical or important findings exist). A review without this final verdict line is treated as inconclusive.

List each finding with file path, line number, severity, and description.
```

Probe the following methods **in order** and use the first one that can run the Codex frontier model. Record the chosen method once in L8 and reuse it for every round in L9–L11. Never fall back to Claude when this path fails: that would turn a Claude implementation into same-family self-review.

**Method 1 — Codex CLI (preferred).** Check CLI and catalog availability, then resolve and record `CODEX_FRONTIER_MODEL` once for every review round:

```bash
command -v codex >/dev/null
command -v jq >/dev/null
CODEX_FRONTIER_MODEL=$(codex debug models | jq -er '
  [.models[]
   | select(.visibility == "list")
   | select(((.description // "") | ascii_downcase) | contains("latest frontier"))]
  | sort_by(.priority)
  | (.[0].slug // empty)
')
test -n "$CODEX_FRONTIER_MODEL"
```

If found, each round in L9–L11 runs Codex as **one foreground, bounded Bash call** whose review is captured to a file. Three rules make this reliable — they exist because the dominant failure mode is backgrounding Codex and then trying to detect completion by grepping the process table for its PID, which matches the agent's own shell (or the `grep`/`pgrep` itself) and deadlocks the session until the harness force-recovers it:

1. **Never background it.** Run `codex exec` as a single foreground Bash call and let the call return — the return *is* the completion signal. Do not set `run_in_background`, do not append `&`, do not `disown`.
2. **Never poll for completion.** Do not run `ps`, `pgrep`, `jobs`, `wait`, or read `/proc` to find Codex. `--output-last-message` writes the review to the output file atomically when Codex exits; there is nothing to poll.
3. **Bound it.** Wrap the call in `timeout 600` and set the Bash tool's own timeout to its `600000` ms maximum. A round that needs longer than 10 minutes is treated as inconclusive (below), never waited on indefinitely.

The command L9 runs each round (feed the diff on stdin so Codex spends no turns re-fetching it):

```bash
REVIEW_OUT=$(mktemp)
gh pr diff "$PR_NUMBER" > "$REVIEW_OUT.diff"
if [ ! -s "$REVIEW_OUT.diff" ]; then
  CODEX_STATUS=1
else
  timeout 600 codex exec \
    --model "$CODEX_FRONTIER_MODEL" \
    --dangerously-bypass-approvals-and-sandbox \
    --skip-git-repo-check \
    --output-last-message "$REVIEW_OUT" \
    "<the review prompt above, with $PR_NUMBER and $REPO expanded; the diff is also supplied on stdin>" \
    < "$REVIEW_OUT.diff"
  CODEX_STATUS=$?
fi
```

Interpret the result deterministically — never re-run merely to "check if it finished":

- **`CODEX_STATUS` is 0 and `$REVIEW_OUT` is non-empty:** that file is the round's review. Parse it for the verdict and per-severity findings and post it to the PR.
- **`CODEX_STATUS` is nonzero or `$REVIEW_OUT` is empty:** the round is inconclusive. Re-run this exact command **once**. If the rerun is also inconclusive, fall through to Method 2.

**Method 2 — Codex MCP server.** If the CLI invocation is inconclusive, check whether a Codex MCP server is connected to this session: use tool discovery with query `+codex` and look for a Codex conversation tool. Use it only if the latest frontier slug was resolved from the Codex catalog and the tool accepts an explicit `model` argument; invoke it with `model: "$CODEX_FRONTIER_MODEL"` and the review prompt, then parse the returned conversation text exactly like CLI output. A Codex tool whose model cannot be set to the resolved frontier slug does not satisfy this reviewer route.

Do not use `@codex review` as a fallback for this route because the GitHub integration does not accept the dynamically resolved frontier slug.

**If both methods fail:** do not fall back to Claude. Surface an explicit error and escalate now (do not proceed to L9–L11):

```bash
gh pr comment $PR_NUMBER --body "$(cat <<'EOF'
## Code Review Cannot Proceed

This implementation is routed to the latest Codex frontier reviewer, but no qualifying invocation method is available: the current frontier slug could not be resolved from `codex debug models`, `codex exec --model "$CODEX_FRONTIER_MODEL"` was inconclusive, or no connected Codex MCP tool could use the resolved frontier slug. The reviewer cannot silently downgrade to Claude because that may be the implementation model. Marking for human review.
EOF
)"
gh pr edit $PR_NUMBER --add-label "needs-human-review"
```

Then call `mcp__linear-server__save_issue` with `issue: "<ISSUE_IDENTIFIER>"` and `labels: ["needs-human-review", ...existing]`. Skip directly to step L16 with result `ESCALATED`.

**Project-configured reviewer (`PRIMARY_RUNTIME=unknown`, no `--reviewer`, and project config provides a command):**

The project's `CLAUDE.md` or `AGENTS.md` may contain a section headed `## Code Review` with a fenced code block. The command is a template with these variables:

- `$PR_NUMBER` — the PR number
- `$BRANCH` — the current branch name
- `$REPO` — the repository in `owner/repo` format

Example from a project's CLAUDE.md:

<pre>
## Code Review

```bash
CODEX_FRONTIER_MODEL=$(codex debug models | jq -er '
  [.models[]
   | select(.visibility == "list")
   | select(((.description // "") | ascii_downcase) | contains("latest frontier"))]
  | sort_by(.priority)
  | (.[0].slug // empty)
')
test -n "$CODEX_FRONTIER_MODEL"
codex exec --model "$CODEX_FRONTIER_MODEL" --dangerously-bypass-approvals-and-sandbox \
  "Review PR #$PR_NUMBER on branch $BRANCH in $REPO. \
   Report findings as [CRITICAL], [IMPORTANT], or [STYLE]. \
   Respond with APPROVE or REQUEST_CHANGES."
```
</pre>

If found, that command is what L9 will run, with variables resolved from the shell.

**Claude reviewer (`PRIMARY_RUNTIME=codex` by default, or selected via `--reviewer claude|opus|fable|sonnet|haiku`):**

Resolve `CLAUDE_REVIEW_MODEL`: `claude` and `opus` both resolve to the moving `opus` alias; the other explicit reviewer values retain their names. For the automatic Codex-runtime route, it is always `opus`, ensuring each run uses the latest available Claude Opus model.

**Probe the CLI end-to-end before the first round.** `command -v claude` alone is not sufficient: the binary can be present yet unable to serve the review model — not authenticated, no access to the resolved model, or no network from the invoking sandbox. In every one of those cases `claude --print` exits with empty stdout and the explanation only on stderr, which is exactly the undiagnosable "completed without producing review output" failure this probe exists to prevent. Run:

```bash
command -v claude >/dev/null
command -v jq >/dev/null
REVIEW_DIR=$(mktemp -d)
if [ -z "$REVIEW_DIR" ] || [ ! -d "$REVIEW_DIR" ]; then
  echo "mktemp -d failed: cannot stage reviewer artifacts" >&2
  exit 1
fi
timeout --kill-after=15 120 claude --print \
  --model "$CLAUDE_REVIEW_MODEL" \
  --output-format json \
  "Reply with exactly the word OK" \
  </dev/null >"$REVIEW_DIR/probe.json" 2>"$REVIEW_DIR/probe.err"
PROBE_STATUS=$?
jq -e '(.type == "result") and (.is_error == false)
       and ((.result | type) == "string") and ((.result | length) > 0)' \
  "$REVIEW_DIR/probe.json" >/dev/null 2>"$REVIEW_DIR/probe-jq.err"
PROBE_PARSE=$?
```

`mktemp -d` creates `REVIEW_DIR` with mode 700; every reviewer artifact (diff, envelopes, stderr) lives inside it — never in ad-hoc sibling paths — and the whole directory is removed with `rm -rf "$REVIEW_DIR"` when review concludes or escalates, because it holds the full PR diff and CLI diagnostics. The `if` guard exits before any child path exists: when `mktemp` fails or returns empty, the shell call ends there — treat that nonzero exit as a probe failure and escalate. Never construct child paths from an unverified `REVIEW_DIR`, because an empty prefix turns them into root-level paths like `/pr.diff`. Run the probe with the host shell tool's deadline strictly above its 135 s worst case (120 s inner bound + 15 s kill grace) — e.g., 180000 ms for Claude Code's Bash tool, whose default deadline is shorter. The jq filter requires `.is_error == false` literally (a missing field must fail, so `| not` is not acceptable) and a non-empty string `result`.

The probe succeeds only when `PROBE_STATUS` and `PROBE_PARSE` are both 0. If `claude` or `jq` is missing, or the probe fails, the Claude reviewer is unavailable: post a `Code Review Cannot Proceed` comment (template below) that names the required model **and includes the probe evidence** — `PROBE_STATUS` and `PROBE_PARSE` plus the tails of `probe.err` and `probe-jq.err`, or the JSON envelope's `subtype`/`result` when the error surfaced there — then add `needs-human-review` to the PR and Linear issue and skip to L16 with result `ESCALATED`. Do not invoke Codex or inherit the current session model, and do not spend review rounds discovering a statically broken CLI.

If the probe succeeds, each round in L9–L11 runs Claude Code as **one foreground, bounded call**, under the same three rules as the Codex CLI reviewer: never background it, never poll for completion, and bound every network-dependent command with `timeout --kill-after=<grace> <seconds>` so a process that ignores `SIGTERM` still dies from `SIGKILL`. The host shell tool's own deadline must be **strictly longer** than the attempt's combined worst case — the bounded diff fetch plus the bounded Claude call plus their kill graces (extraction is local and negligible) — so every `timeout` exit status is observed before any host-side kill. With Claude Code's 600000 ms Bash tool maximum that means `timeout --kill-after=10 60` for the diff fetch and `timeout --kill-after=15 490` for the Claude call (≤ 575 s combined worst case); a Codex host that allows longer foreground calls may raise the Claude bound, keeping the combined worst case strictly below the host deadline.

Claude Code `--print` accepts a prompt argument and appends piped stdin to that prompt; supply the diff on stdin. Use `--output-format json`, not `text`: the JSON envelope makes success machine-checkable (`type`, `is_error`, non-empty `result`) where an empty text stream is ambiguous, and capturing stderr separately means every failure leaves diagnosable evidence. Each attempt starts from a clean slate so a failed attempt can never surface a previous attempt's files:

```bash
rm -f "$REVIEW_DIR"/pr.diff "$REVIEW_DIR"/gh-diff.err \
      "$REVIEW_DIR"/review.json "$REVIEW_DIR"/review.err \
      "$REVIEW_DIR"/review.txt "$REVIEW_DIR"/review-jq.err
timeout --kill-after=10 60 gh pr diff "$PR_NUMBER" \
  > "$REVIEW_DIR/pr.diff" 2> "$REVIEW_DIR/gh-diff.err"
GH_DIFF_STATUS=$?
if [ "$GH_DIFF_STATUS" -ne 0 ] || [ ! -s "$REVIEW_DIR/pr.diff" ]; then
  CLAUDE_STATUS=not-run
else
  timeout --kill-after=15 490 claude --print \
    --model "$CLAUDE_REVIEW_MODEL" \
    --output-format json \
    "<the review prompt above, with $PR_NUMBER and $REPO expanded; the complete diff is supplied on stdin>" \
    < "$REVIEW_DIR/pr.diff" > "$REVIEW_DIR/review.json" 2> "$REVIEW_DIR/review.err"
  CLAUDE_STATUS=$?
fi
jq -er 'select((.type == "result") and (.is_error == false))
        | .result | select(type == "string")' \
  "$REVIEW_DIR/review.json" > "$REVIEW_DIR/review.txt" 2> "$REVIEW_DIR/review-jq.err"
EXTRACT_STATUS=$?
VERDICT=$(awk 'NF {line=$0} END {print line}' "$REVIEW_DIR/review.txt" \
  | grep -E '^[[:space:]*_]*Verdict:[[:space:]*_]*(APPROVE|REQUEST_CHANGES)[[:space:]*_]*$' \
  | grep -oE 'APPROVE|REQUEST_CHANGES')
```

Interpret the result deterministically — never re-run merely to "check if it finished". A round is **conclusive** only when ALL of the following hold; a partial `gh` diff, a truncated JSON stream rescued by `jq`, or review text with no verdict must never pass as a completed review:

- `GH_DIFF_STATUS` is 0 and `$REVIEW_DIR/pr.diff` is non-empty (a diff failure is a `gh` failure — `CLAUDE_STATUS` reads `not-run` so the evidence names the failing component, never Claude);
- `CLAUDE_STATUS` is 0;
- `EXTRACT_STATUS` is 0 and `$REVIEW_DIR/review.txt` is non-empty;
- `VERDICT` is non-empty. The verdict comes only from the review's **final nonblank line**, which the prompt demands be a dedicated `Verdict:` line (the extraction tolerates surrounding Markdown emphasis). Prose that merely mentions a verdict token, an echoed rubric, a refusal with an earlier standalone verdict, or a token embedded in another word (`DISAPPROVE`) can never pass.

**Conclusive:** `$REVIEW_DIR/review.txt` is the round's review with verdict `$VERDICT`. Parse the per-severity findings and post it to the PR. **Anything else — the round is inconclusive.** Before retrying, record the attempt's evidence: `GH_DIFF_STATUS`, `CLAUDE_STATUS` (124 means `timeout` expired and `SIGTERM` ended the call; 137 means the call ignored `SIGTERM` and `--kill-after` sent `SIGKILL`), `EXTRACT_STATUS`, the tails of `review.err`, `gh-diff.err`, and `review-jq.err`, and the envelope's `subtype`/`is_error` from `review.json` if it parsed. Then rerun the exact foreground command once. If the rerun is also inconclusive, do not invoke Codex or inherit the session model. Post the `Code Review Cannot Proceed` comment below, add `needs-human-review` to the PR and Linear issue, and skip to L16 with result `ESCALATED`.

**Escalation comment (probe failure and inconclusive rounds alike).** The comment must carry the captured evidence — never only a prose conclusion like "completed without producing review output", which leaves the failure undiagnosable. Before posting, truncate each stderr tail to its last 20 lines, cap the result-envelope excerpt at ~2000 characters, and redact anything credential-shaped (API keys, bearer tokens, `sk-`/`ghp_`-style strings, URLs with embedded credentials) — CLI diagnostics can leak local paths and account details, and a PR comment is public within the repo:

```bash
timeout --kill-after=10 60 gh pr comment $PR_NUMBER --body "$(cat <<EOF
## Code Review Cannot Proceed

This Codex implementation requires review by the latest Claude Opus model (\`--model $CLAUDE_REVIEW_MODEL\`). The reviewer cannot silently downgrade to Codex because that would be same-family self-review. Marking for human review.

Evidence:
- Failure point: <probe | review round N attempt M>
- Statuses: <GH_DIFF_STATUS / CLAUDE_STATUS / EXTRACT_STATUS, or PROBE_STATUS / PROBE_PARSE; note 124 = timed out at <bound>s, 137 = SIGKILL after ignoring SIGTERM>
- stderr tails (last 20 lines each, redacted): <from the captured stderr files, or "empty">
- Result envelope (truncated, redacted): <subtype / is_error / result excerpt, or "unparseable">
EOF
)"
COMMENT_STATUS=$?
```

The escalation posting is itself network-dependent, so it carries the same `timeout --kill-after` bound as every other remote command. If `COMMENT_STATUS` is nonzero, do not stall: note the failed posting in the L16 summary, still remove `$REVIEW_DIR`, still apply the `needs-human-review` labels, and still finish with result `ESCALATED`.

For automatic Codex-runtime review, the PR comment header must identify `opus (latest Claude Opus)`, not `claude session model`. This makes accidental regression to the behavior in the linked failure visible without pinning a model version.

### L9. Round 1: Review and Fix

Run the reviewer selected in L8. Parse the output for:

- **Verdict**: APPROVE or REQUEST_CHANGES
- **Finding counts** by severity: CRITICAL, IMPORTANT, STYLE

Post the review back to the PR as a comment per the "Posting the review back to the PR" requirement in L8. This applies to every round — L9, L10, and L11.

**If APPROVE (no findings):** Skip to step L12.

**If any findings:** Fix ALL findings — critical, important, and style. For each:

1. Read the cited file and line
2. Understand the issue
3. Apply the fix
4. Run tests to verify

After all fixes:

- Run the project's test suite to ensure nothing is broken
- Commit: `fix: address code review round 1 findings`
- Push

### L10. Round 2: Re-Review and Fix

Run the review command again. Parse the output.

- **If APPROVE:** Proceed to step L12.
- **If style-only findings remain (no CRITICAL or IMPORTANT):** Proceed to step L12. Create a follow-up issue for style findings after merge (step L14).
- **If CRITICAL or IMPORTANT findings remain:** Fix all findings, commit, push. Proceed to L11.

### L11. Final Review (Gate Check)

Run the review command one final time. Parse the output.

- **If APPROVE or style-only:** Proceed to step L12.
- **If CRITICAL or IMPORTANT findings still remain:** Escalate.

**Escalation:**

1. Post a summary comment on the PR listing unresolved findings:

   ```bash
   gh pr comment $PR_NUMBER --body "$(cat <<'EOF'
## Unresolved Critical/Important Findings

After 3 review rounds, the following findings remain unresolved:

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

4. Do NOT merge. Skip to step L16 with result ESCALATED.

### L12. Wait for CI and Fix Failures

After code review is complete, wait for CI checks.

```bash
gh pr checks $PR_NUMBER --watch --fail-level all
```

If CI fails:

1. Read the CI logs to understand the failure
2. Fix the issue locally, run the project's test suite
3. Commit and push

If CI still fails after one fix attempt, escalate (add `needs-human-review` per step L11) and skip to step L16 with result ESCALATED.

### L13. Merge

Before merging, handle remaining style-only findings from round 2 by creating a follow-up issue (step L14).

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

### L14. Follow-Up Issue for Style Findings

If style-only findings remain after round 2, create a follow-up Linear issue. Call `mcp__linear-server__save_issue`:

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

### L15. AC-Gate and Close Issue

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

### L16. Post Summary

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

Orchestrator for a parent issue with children. Routes each child issue to a Claude Code sub-agent (inheriting the session model unless overridden) or the Codex CLI, per the "Arguments and Model Selection" resolution rules.

### P1. Verify Primary Issue Branch and Rebase

Before doing any orchestration work, ensure the session is on the **primary issue's branch** — never the parent issue's branch — and that the branch is rebased on the latest `origin/main`. The harness may have started this worktree on a generic session branch (see Worktree Conventions); in that case this step moves the session onto the primary issue's branch. Run this step at the top of **every** orchestrator invocation, including re-invocations and resumptions.

Define `IDENT_LC` as `ISSUE_IDENTIFIER` lowercased (e.g., `HOL-1061` → `hol-1061`). All branch-name comparisons in this step are **case-insensitive** — match the lowercased branch name against `IDENT_LC`.

1. Determine the expected branch for the primary issue:
   - Prefer `gitBranchName` from the `mcp__linear-server__get_issue` response (e.g., `jeff/hol-1061-…`).
   - If absent, look for an existing local or remote branch whose lowercased name contains `IDENT_LC` (e.g., a `claude/hol-1061-…` or `cyrus/hol-1061-…` session worktree branch — see Worktree Conventions). Any such branch is acceptable as long as it is unique to this primary issue.

2. Classify the **current** branch and act accordingly. Lowercase the current branch name, then pick the first matching case:

   - **Primary issue branch** — the lowercased name contains `IDENT_LC`. The session is already where it should be; continue to step 3.

   - **Generic session branch** — the lowercased name contains **no Linear issue identifier at all** (no `<team>-<number>` token matching `[a-z]{2,}-[0-9]+`). This is the normal case for Claude Code remote sessions and other harnesses that create the worktree on a generic branch (e.g., `claude/<slug>` or an auto-generated worktree branch) rather than an issue branch. **Do not abort.** Switch to the primary issue's branch:
     - If the expected branch from step 1 exists locally, check it out; if it exists only on the remote, check it out with tracking (`git checkout -b <branch> origin/<branch>`). If the checkout fails because that branch is checked out in another worktree, create a uniquely-suffixed variant from `origin/main` instead (e.g., `<expected-branch>-2`).
     - Otherwise create it fresh from `origin/main`:
       ```bash
       git fetch origin
       git checkout -b <expected-branch> origin/main
       ```
       Use the expected branch name from step 1 when one was determined (e.g., Linear's `gitBranchName` such as `jeff/hol-1061-…`); if step 1 produced none, use `feat/<ident-lc>-<slug>` following the L2 naming convention.
     - Leave the generic session branch untouched — do not delete it or commit to it. Then continue to step 3.

   - **Foreign issue branch** — the lowercased name contains a Linear issue identifier that is **not** `IDENT_LC` (for example, the parent issue's identifier). Abort:
     - Ensure the `needs-human-review` label exists for the team (call `mcp__linear-server__list_issue_labels`; create it via `mcp__linear-server__create_issue_label` if missing).
     - Post a comment on the issue:
       ```
       Refusing to orchestrate from branch `<current-branch>` — it belongs to a different issue than the primary issue <ISSUE_IDENTIFIER>. Expected the lowercased branch name to contain `<IDENT_LC>` or no issue identifier at all. Switch worktrees and try again.
       ```
     - Add `needs-human-review` to the issue and stop the skill.

3. Fetch the latest `origin/main`, then check for **parent-branch contamination** on the plan branch before rebasing. The orchestrator never commits to the plan branch (all real work goes to sub-issue feature branches), so any commits between `origin/main` and HEAD that reference foreign issue identifiers indicate the harness created this worktree's branch from the wrong base — typically a parent issue's branch rather than `main`. A plain rebase would replay those foreign commits onto `origin/main`, polluting the plan branch:

   ```bash
   git fetch origin
   FOREIGN_COMMITS=$(git log --format='%H %s%n%b' origin/main..HEAD \
     | grep -oE '[A-Z]{2,}-[0-9]+' \
     | grep -v "^${ISSUE_IDENTIFIER}$" \
     | sort -u)
   ```

   **If `$FOREIGN_COMMITS` is non-empty:** hard-reset the plan branch to `origin/main` and post a recovery comment. The reset is safe because the orchestrator never commits to the plan branch — there is no work to preserve:

   ```bash
   git reset --hard origin/main
   ```

   Post a comment on the primary issue via `mcp__linear-server__save_comment`:

   ```
   Detected parent-branch contamination on the plan branch (commits referencing: <list of foreign identifiers>).
   Reset the plan branch to origin/main. Sub-issue workers will branch from origin/main as designed.
   ```

   Then continue to P2 — the plan branch now matches `origin/main` exactly.

   **If `$FOREIGN_COMMITS` is empty:** fall through to the existing rebase path:

   ```bash
   git rebase origin/main
   ```

   If the rebase succeeds with dropped commits (commits that already landed on `origin/main` via merged PRs), that is the desired behavior — those phases are already complete and must not be re-implemented.

   If the rebase fails due to conflicts:
   - **Before** running `git rebase --abort`, capture the conflicting paths while the rebase is still in progress:
     ```bash
     CONFLICTS=$(git diff --name-only --diff-filter=U)
     ```
   - Then run `git rebase --abort`.
   - Ensure the `needs-human-review` label exists for the team (create via `mcp__linear-server__create_issue_label` if missing).
   - Post a comment on the issue listing the captured conflicting paths from `$CONFLICTS`.
   - Add `needs-human-review` to the issue and stop the skill.

### P2. Start Wall Clock Timer

```bash
PLAN_START_TIME=$(date +%s)
```

### P3. List Children

Call `mcp__linear-server__list_issues` with `parentId: "<ISSUE_ID>"`, sorted by creation time ascending.

For each child, record:

- `identifier` (e.g., `PLA-301`)
- `id` (UUID)
- `title`
- `statusType` (`triage`, `backlog`, `unstarted`, `started`, `completed`, `canceled`)
- `labels` (names, lowercased for routing)

Skip any child whose status is `completed` or `canceled`.

If no open children remain, post a comment that all work is complete and stop.

### P4. Review Existing Progress

After listing children in P3, inspect the diff between `origin/main` and `HEAD` to detect work already implemented in this branch from a prior orchestrator session. The orchestrator **must not** re-implement what already landed on `main` or what already exists as commits on the primary issue branch.

```bash
git log --oneline origin/main..HEAD
git diff --stat origin/main..HEAD
```

Classify each open sub-issue from P3 as **already-implemented** when either of the following holds:

- Its Linear status is `completed` or `canceled` — the rebase has already incorporated any merged work from `origin/main`, so nothing remains to do. (P3 already filters these out, but list them in the resume comment for visibility.)
- A commit message on the primary issue branch (`origin/main..HEAD`) references the sub-issue's identifier — for example, `feat(...): … Refs: APP-301` or any commit subject/body that includes the identifier in the form `<TEAM>-<NUMBER>`. This indicates a prior orchestrator session committed work for that sub-issue without merging.

Build a `SKIP_SUB_ISSUES` set containing every already-implemented sub-issue identifier. The dispatch step (P6) **must consult `SKIP_SUB_ISSUES` and skip any sub-issue in it** — do not dispatch a worker for those identifiers and do not delete their existing branch state.

Post a brief comment on the parent issue summarizing the orchestration state:

```
## Orchestration state

- Branch: <current-branch>
- Commits ahead of origin/main: <N>
- Already-implemented sub-issues (skipped): <list of identifiers, or "none">
- Remaining sub-issues to dispatch: <list of identifiers>
```

### P5. Transition Labels

Replace `planning` with `implementing` on the parent issue.

Ensure the `implementing` label exists. Then call `mcp__linear-server__save_issue`:

- `issue: "<ISSUE_IDENTIFIER>"`
- `labels: ["implementing", ...existing labels minus "planning"]`

### P6. Dispatch Sub-Issues

Process sub-issues **sequentially** (each phase depends on the previous one). Before dispatching each open child, the orchestrator (this session) must inspect that sub-issue's labels and choose the implementation runner from those labels. The dispatched worker runs the full leaf lifecycle for that one sub-issue and returns a short result summary.

Capture the orchestrator's primary issue branch once at the start of P6 — pre-dispatch cleanup may need to switch back to it if a previous run crashed while a sub-issue branch was checked out:

```bash
PRIMARY_BRANCH=$(git rev-parse --abbrev-ref HEAD)
```

**Honor the `SKIP_SUB_ISSUES` set built in P4**: if a sub-issue's identifier appears in `SKIP_SUB_ISSUES`, skip it entirely — do not run pre-dispatch cleanup, do not dispatch a worker, and do not delete its branch or PR. Move on to the next sub-issue.

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
3. If `$SUB_BRANCH` is the current branch (a previous orchestrator session crashed while it was checked out), switch back to the orchestrator's primary issue branch first — `git branch -D` will refuse to delete a checked-out branch. The orchestrator already knows its primary branch from P1; capture it once at the start of P6 as `PRIMARY_BRANCH` and reuse here:
   ```bash
   if [ "$(git rev-parse --abbrev-ref HEAD)" = "$SUB_BRANCH" ]; then
     git checkout "$PRIMARY_BRANCH"
   fi
   ```
4. Delete the local branch:
   ```bash
   git branch -D "$SUB_BRANCH" 2>/dev/null || true
   ```
5. Refresh `origin/main` without checking out `main` locally. The orchestrator stays on its own primary issue branch (e.g., `jeff/hol-XXXX-…`, `claude/hol-XXXX-…`, or `cyrus/hol-XXXX-…`) — that branch is already rebased on `origin/main` per P1, so a plain fetch is sufficient. **Do not run `git checkout main`**: `main` is checked out in another worktree and the checkout will fail.
   ```bash
   git fetch origin
   ```
6. Reset the sub-issue to its unstarted state and remove the `implementing` label:
   Call `mcp__linear-server__save_issue` with `issue: "<SUB_IDENTIFIER>"`, `state: "<unstarted-type state>"`, removing `implementing` from labels.
7. Post a comment on the sub-issue via `mcp__linear-server__save_comment`:
   ```
   Discarding partial work from a previous attempt. Starting a clean implementation.
   ```

**Dispatch selection per sub-issue** — apply the "Arguments and Model Selection" resolution:

- If `MODEL_OVERRIDE` is set, it wins for every child: `codex` routes to the **Codex CLI**; any other name routes to a Claude sub-agent with that `model`.
- Else lowercase the child's label names and match routing labels: `codex` → Codex CLI; `fable`/`opus`/`sonnet`/`haiku` → Claude sub-agent with that `model`. Conflicts prefer `codex` > `fable` > `opus` > `sonnet` > `haiku`.
- Else (no override, no routing label) spawn a Claude sub-agent with **no `model` parameter** so the worker inherits the session-configured model.

If routing selects Claude, spawn a sub-agent. Append ` --model <MODEL_OVERRIDE>` to the skill invocation only when `MODEL_OVERRIDE` is set:

```
Agent(
  description: "Implement <SUB_IDENTIFIER>",
  model: "<RESOLVED_MODEL — omit entirely for the session default>",
  prompt: "Invoke /linear-workflow:implement-issue <SUB_IDENTIFIER> to implement this sub-issue
  end-to-end. The skill handles branching, implementation, code review, CI, merge, and issue
  transitions. Run to completion. Return a short summary: result (MERGED |
  MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED), PR number, and any follow-up issue identifier."
)
```

If routing selects Codex, run the Codex CLI directly:

```bash
CODEX_FRONTIER_MODEL=$(codex debug models | jq -er '
  [.models[]
   | select(.visibility == "list")
   | select(((.description // "") | ascii_downcase) | contains("latest frontier"))]
  | sort_by(.priority)
  | (.[0].slug // empty)
')
test -n "$CODEX_FRONTIER_MODEL"
codex exec --model "$CODEX_FRONTIER_MODEL" --dangerously-bypass-approvals-and-sandbox \
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
2. Switch the runner for this sub-issue to Claude and dispatch a sub-agent with **no `model` parameter** (inheriting the session model), using the same sub-issue identifier and prompt.

**If a Claude runner hits a usage limit (any model — session default, override, or label-routed):**

1. Do not increment the retry counter.
2. Parse the earliest "retry after" time from all usage-limit messages (e.g., "available again at HH:MM UTC", "retry in N minutes", "resets at HH:MM"). Convert to seconds until that time.
3. If no retry time is parseable, default to 15 minutes (`900` seconds).
4. Wait (`sleep <seconds_until_retry>`) — do not prompt the user.
5. Re-dispatch the sub-issue using its original model resolution (`MODEL_OVERRIDE` if set, else routing label, else session default).

Never prompt the user for guidance on usage limits — resolve autonomously.

#### Orchestrator Constraint: Never Take Over Implementation Work

**The orchestrator must never implement sub-issue work directly.** It does not write code, create commits, create branches, or perform any leaf-mode steps itself — not even to "help" a stuck worker finish. If a worker fails or gets stuck, the orchestrator's only permitted response is to clean up and dispatch a replacement worker using the same routing rules.

#### Detecting a Stuck Worker

A worker has **failed to complete** if it returns without a valid result summary (MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED) — for example, it got stuck mid-implementation and did not finish.

Use a retry loop with up to **3 total attempts** per sub-issue:

1. If the worker's returned output does not contain a valid result summary, it got stuck.
2. Note where it got stuck and why (e.g. "wrote files but did not commit", "opened PR but did not wait for CI").
3. Refresh `origin/main`. The orchestrator stays on its own primary issue branch — **never check out `main` locally**, which would conflict with the repo's primary worktree.
   ```bash
   git fetch origin
   ```
4. Re-resolve the sub-issue's model (re-check labels; `MODEL_OVERRIDE` still wins), then dispatch a replacement worker with the same sub-issue identifier plus a warning. Do not describe implementation steps — just provide context and let the skill decide what to do.

   If the route is Claude (omit `model` for the session default, as always):
   ```
   Agent(
     description: "Implement <SUB_IDENTIFIER> (retry <N>)",
     model: "<RESOLVED_MODEL — omit entirely for the session default>",
     prompt: "Invoke /linear-workflow:implement-issue <SUB_IDENTIFIER>.

     Warning: A previous attempt did not complete. Point: <e.g. 'wrote files but did not commit'>. Reason: <e.g. 'worker returned without a result summary'>.

     Invoke the skill and let it run to completion. Return a short result summary: result (MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED), PR number, and any follow-up issue identifier."
   )
   ```

   If the route is Codex:
   ```bash
   CODEX_FRONTIER_MODEL=$(codex debug models | jq -er '
     [.models[]
      | select(.visibility == "list")
      | select(((.description // "") | ascii_downcase) | contains("latest frontier"))]
     | sort_by(.priority)
     | (.[0].slug // empty)
   ')
   test -n "$CODEX_FRONTIER_MODEL"
   codex exec --model "$CODEX_FRONTIER_MODEL" --dangerously-bypass-approvals-and-sandbox \
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

After each sub-issue, refresh `origin/main`. The orchestrator stays on its own primary issue branch between dispatches — **never check out `main` locally**. The next sub-issue's worker will branch directly from `origin/main` in its L2 step, so no local `main` is ever required.

```bash
git fetch origin
```

### P7. Sweep for Follow-Up Issues

After all original children are processed, re-list children via `mcp__linear-server__list_issues` with `parentId: "<ISSUE_ID>"`.

Compare against the original list. Any new open child is a follow-up created during review.

Process follow-ups with the **full P6 pre-dispatch sequence** — including the *Pre-Dispatch: Detect and Discard Partial Work* step — then dispatch one worker per follow-up using the same label-routing rules.

### P8. Nested Parent Issues

If a sub-issue or follow-up is itself a parent (has its own children), the dispatched worker invoked in P6 will simply re-enter this skill in Parent Mode and orchestrate its own children with the same model-resolution rules — including the P6 pre-dispatch partial-work cleanup for each grand-child. Nested orchestration composes naturally — no special handling is needed for depth beyond re-resolving the model at each parent.

Nested parent orchestrators follow the same resolution: `MODEL_OVERRIDE` (propagated via `--model`) first, then the nested parent issue's own routing label (`codex` → Codex CLI; `fable`/`opus`/`sonnet`/`haiku` → that Claude model), otherwise inherit the session-configured model.

### P9. Post Summary

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

### P10. Close Parent

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
- **Cross-runtime review**: Claude-hosted implementation requires a Codex frontier path; Codex-hosted implementation requires the Claude Code CLI with access to the latest Claude Opus through the `opus` alias. `--reviewer` is the only reviewer-routing override. `--model` and issue routing labels affect implementation only.
- **Codex access**: The `codex` CLI and `jq` must resolve the latest frontier slug from `codex debug models`; `codex exec --model "$CODEX_FRONTIER_MODEL"` is preferred, and a model-selectable Codex MCP server may be used after an inconclusive CLI review. The Codex CLI is the only Codex method usable for implementation dispatch in Parent Mode.
- **Claude access**: The `claude` CLI must be on `PATH` and authenticated when a Codex implementation is automatically paired with the latest Claude Opus. L8 proves this end-to-end with a bounded `--print --output-format json` probe before the first round; `jq` is required to parse the result envelope. Reviewer failure escalates to human review with the captured exit status and stderr; it never falls back to the implementation model family.

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
