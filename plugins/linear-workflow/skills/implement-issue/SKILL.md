---
name: implement-issue
description: v3.1.0 — Implement a Linear issue end-to-end, either as one leaf issue or as a parent orchestrating children. All implementation work — orchestrator and sub-issue workers alike — runs in the harness the user invoked; --model adjusts only the model within that harness, and only code review crosses to the opposite harness. Cross-runtime review posts findings to the PR; reviewer-output failures stop the merge and produce redacted diagnostics plus a best-effort related Linear issue and document. Every Linear and PR comment the skill posts ends with an agent-attribution footer naming the harness, model, and reasoning effort (for example `claude fable-5 high`) so colleagues know they are reading agent output, not the human account owner. Use --reviewer only to override reviewer selection. Triggers when the user provides a Linear issue URL or identifier (for example PLA-287) and asks to implement, work on, fix, resolve, or execute its plan.
version: 3.1.0
# Guardrail: whenever version changes, update the leading vX.Y.Z prefix in description in the same PR.
---

# Implement Issue

Implement a Linear issue end-to-end. This skill self-detects whether the issue is a leaf issue (no children) or a parent issue (has children) and adapts its behavior:

- **Leaf mode**: Branch, implement, open PR, run adversarial cross-runtime code review (the latest Claude Opus reviews Codex implementations; the Codex frontier model reviews Claude implementations; `--reviewer` explicitly overrides), post the review back to the PR as a comment, and allow up to 2 fix rounds. If an invoked reviewer produces no usable output, capture diagnostics, attempt the related Linear issue and document, and stop for human review without merging; otherwise wait for CI, merge, and mark Done.
- **Parent mode**: Orchestrate implementation of all child issues — each worker runs as a sub-agent of the same harness the orchestrator runs in, inheriting the session model unless the `--model` argument overrides it — then track results, sweep for follow-ups, and post a summary.

**The harness never changes for implementation.** The orchestrator always runs in the top-level harness the user invoked, and every sub-issue worker is a sub-agent of that same harness. If the user starts in Codex, all implementation runs in Codex; if the user starts in Claude Code, all implementation runs in Claude Code. The only cross-harness dispatch this skill ever performs is adversarial code review.

Implement the Linear issue **{{SKILL_INPUT}}**.

## Arguments and Model Selection

`{{SKILL_INPUT}}` contains the issue reference and optional flags:

```
<issue> [--model <name>] [--reviewer <codex|claude|fable|opus|sonnet|haiku>]
```

Parse and record:

- The issue reference (identifier or URL) — strip any flags before parsing it in step 1.
- `MODEL_OVERRIDE` — the value of `--model <name>` (also accepted as `model=<name>`) if present; otherwise unset. The name must be one the **current harness** understands for its own sub-agents (e.g., `fable`/`opus`/`sonnet`/`haiku` in Claude Code; a Codex model slug in Codex). `--model` selects a model **within** the invoking harness — it never selects a harness.
- `REVIEWER_OVERRIDE` — the value of `--reviewer <name>` (also accepted as `reviewer=<name>`) if present; otherwise unset. Controls **only** the code reviewer selection in L8 — it never affects worker or orchestrator dispatch. `codex` selects the Codex frontier reviewer; `claude` or `opus` selects the latest Claude Opus through the `opus` alias; `fable`/`sonnet`/`haiku` explicitly select that Claude model instead.

Every point in this skill that dispatches **implementation work** — sub-issue workers (P6), nested orchestrators (P8), and retry dispatches — resolves the model with this priority. Code review is deliberately independent: L8 uses `REVIEWER_OVERRIDE` when present and otherwise selects the opposite model family from the detected primary runtime. `MODEL_OVERRIDE` never selects the reviewer.

1. **`MODEL_OVERRIDE`** — the `--model` argument always wins.
2. **Session default** — no override: the worker runs on the invoking session's model. In Claude Code, spawn the sub-agent **without a `model` parameter** so it inherits the session model (e.g., Fable or Opus in remote mode). In Codex, pass the session's own model explicitly when known (a fresh `codex exec` cannot see it otherwise), and only fall back to omitting `--model` — the Codex configured default — when it is not. This is the normal path — never hardcode a fallback model.

Issue labels never influence dispatch. Do not read, match, or honor any label (`codex`, `fable`, `opus`, `sonnet`, `haiku`, or otherwise) as a routing signal — implementation routing comes only from `MODEL_OVERRIDE` and the session default, and the harness is always the one the user invoked.

Workers are always dispatched through the current harness's native sub-agent mechanism: in Claude Code, the `Agent()` tool; in Codex, a `codex exec` subprocess (still the Codex harness). In the worker templates below, `model: "<RESOLVED_MODEL>"` means: pass `MODEL_OVERRIDE` when set, and **omit the model parameter entirely** when resolution fell through to the session default.

When this skill re-invokes itself for a sub-issue or nested parent, propagate `MODEL_OVERRIDE` and `REVIEWER_OVERRIDE` in the invocation when set: `/linear-workflow:implement-issue <SUB_IDENTIFIER> --model <name> --reviewer <name>` (include each flag only when its override is set).

## Linear Conventions

- **Issue** = Linear issue. Fetched via `mcp__linear-server__*` tools.
- **PR** = GitHub pull request. Opened via `gh`.
- The PR body must contain `Fixes <IDENTIFIER>` so Linear auto-closes the issue on merge.
- Always send real newlines in Linear `body` / `description` values — never `\n` escape sequences.
- Every comment this skill posts — on Linear issues and on GitHub PRs — must end with the agent-attribution footer defined in the "Agent Attribution" section, even where a comment template below does not repeat it inline.

## Worktree Conventions

This skill runs inside a git worktree created by an agent harness — a Claude Code remote session, a local Claude Code worktree, Cyrus, or any similar tool that checks out a per-session branch in its own worktree. The skill makes no assumptions specific to any one harness. The conventions that follow from worktree use:

- **Never check out `main` locally.** A branch can only be checked out in one worktree at a time, and `main` belongs to the repo's primary worktree. Refresh state with `git fetch origin` and reference `origin/main` directly.
- **Create every feature branch from `origin/main`**, regardless of which branch the harness checked out in this worktree. This is the skill's own guarantee of a correct base branch — it does not rely on the harness's base-branch routing.
- **Session branch naming varies by harness.** Examples: `claude/<slug>` (Claude Code remote sessions), `cyrus/<identifier>-<slug>`, or `<user>/<identifier>-<slug>` (Linear's `gitBranchName`). Never assume a specific prefix — match a branch to an issue by checking whether the lowercased branch name contains the lowercased issue identifier.
- **Push with `git push origin HEAD`** (no `-u`) so upstream tracking stays on `origin/main`.

## Codex Frontier Model Resolution

Before this skill invokes `codex exec` for **code review**, resolve `CODEX_FRONTIER_MODEL` once from the current Codex model catalog. (Implementation workers in a Codex harness do not use this resolution — they inherit the session default or `MODEL_OVERRIDE` per "Arguments and Model Selection".) Select the visible model whose description identifies it as the latest frontier model, preferring the lowest numeric priority:

```bash
CODEX_FRONTIER_MODEL=$(codex debug models | jq -er '
  [.models[]
   | select(.visibility == "list")
   | select(((.description // "") | ascii_downcase) | contains("latest frontier"))]
  | sort_by(.priority)
  | (.[0].slug // empty)
')
```

Require a non-empty result and pass `--model "$CODEX_FRONTIER_MODEL"` to every review `codex exec` call. Resolve from the catalog at runtime instead of hardcoding a versioned model slug or inheriting a possibly stale configured default. If the frontier model cannot be resolved, treat the Codex CLI review route as unavailable.

**Always run `codex exec` in the foreground and define completion by an observed exit status.** A host execution tool may yield a live connector session before the foreground process exits: a result with an exit code is complete, while a result with a session handle (for example, `session_id`) and no exit code is still running. In the latter case, wait on that exact handle with the connector-native stdin/wait operation until it reports an exit code; do not restart the command. If an outer orchestration call itself yields a `cell_id`, resume that cell with its own wait operation and let it continue waiting on the shell session — never restart either layer. Waiting this way preserves the same foreground execution and is not backgrounding, process-table polling, or a second attempt. Never background the process (no `run_in_background`, `&`, or `disown`) and never poll `ps`/`pgrep`/`jobs`/`/proc` to detect completion: process-table greps match the agent's own shell or the `grep` itself and deadlock the session until the harness force-recovers it. Bound long calls with `timeout`, and allow enough cumulative connector wait time to observe that inner timeout's exit status. The L8 Codex CLI reviewer (Method 1) applies this in full detail.

## Primary Runtime Detection

At the start of the skill, before dispatching any implementation or review worker, detect and record the immutable `PRIMARY_RUNTIME` for this invocation:

1. Ask the native host identity first. If it identifies itself as Codex, set `PRIMARY_RUNTIME=codex`. If it identifies itself as Claude Code, set `PRIMARY_RUNTIME=claude`. A known native identity is authoritative; do not inspect environment markers.
2. Only when native host identity is unavailable, inspect environment markers:
   - `CODEX_THREAD_ID` alone → `PRIMARY_RUNTIME=codex`.
   - `CLAUDE_CODE_ENTRYPOINT` or `CLAUDECODE` without `CODEX_THREAD_ID` → `PRIMARY_RUNTIME=claude`.
   - Conflicting or absent markers → `PRIMARY_RUNTIME=unknown`.
3. If it is unknown, use the explicit unknown-runtime resolution in L8.

The primary runtime is the agent that performs the implementation steps in this invocation, not a CLI binary it later launches. **Never detect the runtime with `command -v codex` or `command -v claude`**: both CLIs can be installed in either host. Detect once and do not recompute inside a review subprocess, because child processes can inherit the primary host's environment markers. In particular, a native Claude host launched beneath Codex remains `claude` even if it inherited `CODEX_THREAD_ID`.

Each leaf worker detects its own runtime. Because P6 always dispatches workers as sub-agents of the orchestrator's own harness, every worker's detection yields the same runtime as the orchestrator's — which is exactly what guarantees that L8's cross-runtime review pairing is adversarial for the whole tree.

## Agent Attribution

Colleagues reading Linear issues and GitHub PRs must be able to tell at a glance that a comment came from an AI agent operating on behalf of a human, not from the human account owner. Every comment this skill posts carries an attribution footer.

Immediately after Primary Runtime Detection, resolve these once and reuse them for the whole invocation:

- `AGENT_HARNESS` — the harness running this skill: `claude` when `PRIMARY_RUNTIME=claude`, `codex` when `PRIMARY_RUNTIME=codex`. When the runtime is unknown, use the native host's self-reported name if it provides one; otherwise `unknown-harness`.
- `AGENT_MODEL` — the model slug this session is actually running (for example `fable-5`, `opus-4-6`, `solstice-alpha`). Take it from the harness's native self-identity: Claude Code sessions know their model ID; a Codex session knows its configured model slug. Use `unknown-model` when it cannot be determined — never guess a plausible slug.
- `AGENT_EFFORT` — the reasoning-effort setting (for example `low`, `medium`, `high`, `xhigh`) when the harness exposes one for this session; omit the token entirely when unknown.

Compose the signature by joining the parts with single spaces:

```
AGENT_SIGNATURE = <AGENT_HARNESS> <AGENT_MODEL>[ <AGENT_EFFORT>]
```

Examples: `claude fable-5 high`, `codex solstice-alpha high`, `claude sonnet-4-5`.

**Footer requirement.** Every comment body this skill sends — Linear comments via `mcp__linear-server__save_comment`, PR comments via `gh pr comment`, and PR close messages via `gh pr close --comment` — must end with a blank line, a `---` separator line, and this line:

```
🤖 <AGENT_SIGNATURE> — automated comment posted by an AI agent on behalf of this repository's operator. Replies here reach an agent, not the human directly.
```

This applies to **every** comment template in this skill, including templates that do not repeat the footer inline. Issue descriptions, issue titles, PR bodies, and Linear documents are exempt — only comments carry the footer.

Sub-agent workers dispatched in Parent Mode run this skill themselves and resolve their own signature: the harness matches the orchestrator's, but `AGENT_MODEL` may differ when `--model` is set, and each worker's footer must state the model that worker actually runs.

Review-round PR comments identify two models, and both must be present: the `Code Review — Round <N>` heading names the reviewer that produced the findings (its resolved slug, for example `codex solstice-alpha` or `claude opus`), and the footer names the agent that posted the comment.

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

---
🤖 <AGENT_SIGNATURE> — automated comment posted by an AI agent on behalf of this repository's operator. Replies here reach an agent, not the human directly.
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
5. **`PRIMARY_RUNTIME=unknown` and no project reviewer exists** → post a `Code Review Cannot Proceed` comment, add `needs-human-review` to the PR and Linear issue, and skip to L16 with result `ESCALATED`. Do not guess and risk same-runtime self-review. Do not run the reviewer-output debug-capture flow below: no reviewer was invoked, so there is no attempt evidence to capture.

`MODEL_OVERRIDE` controls implementation dispatch only. It must not influence L8. Because implementation always runs in the harness the user invoked, `PRIMARY_RUNTIME` reliably identifies the implementation family, and selecting the opposite family here is what guarantees adversarial review: a Codex-invoked implementation is reviewed by the latest Claude Opus, and a Claude-invoked implementation is reviewed by the Codex frontier model.

**Posting the review back to the PR (all reviewer paths).** Every review round must end with the reviewer's findings posted as a comment on the PR. For the Codex CLI, Codex MCP, project-configured, and Claude CLI paths, after parsing the output post it:

```bash
gh pr comment $PR_NUMBER --body "$(cat <<EOF
## Code Review — Round <N> (reviewer: <resolved reviewer slug, e.g. codex $CODEX_FRONTIER_MODEL | claude $CLAUDE_REVIEW_MODEL | project-configured>)

<the reviewer's full findings and verdict, verbatim>

---
🤖 <AGENT_SIGNATURE> — automated comment posted by an AI agent on behalf of this repository's operator. Replies here reach an agent, not the human directly.
EOF
)"
```

The heading names the reviewer model; the footer names the posting agent per the "Agent Attribution" section. Both are required on every review-round comment.

If `REVIEWER_OVERRIDE` deliberately selects the same model family as `PRIMARY_RUNTIME`, add this line below the PR comment heading for every reviewer path: `Same-family review explicitly requested via --reviewer; cross-runtime pairing was overridden.`

**Escalation debug capture (all reviewer invocation paths).** Run this flow exactly once when a selected reviewer path produces no usable output: the Claude probe fails, both permitted attempts for a Claude or project-configured review round are inconclusive, or the Codex CLI attempts and Codex MCP method both fail. Do not run it for selection rule 5 above because that path never invoked or probed a reviewer, so it has no attempt evidence to capture. The caller supplies `FAILURE_POINT`, `FAILURE_SUMMARY`, `REVIEWER_PATH`, `REVIEWER_MODEL`, the path-specific `Code Review Cannot Proceed` PR-comment body, and all completed-attempt evidence. Record `ESCALATION_ISSUE_IDENTIFIER`, `ESCALATION_DOCUMENT_REFERENCE`, and `ESCALATION_LINEAR_ERRORS` for L16.

1. **Assemble the debug bundle before deleting temporary artifacts.** Build Markdown with these sections, modeled after a diagnostic incident report:

   - `# Reviewer failure debug capture`
   - `## Summary`: failure point and summary, `ISSUE_IDENTIFIER`, PR number, repository, branch, primary runtime, reviewer path and exact model, skill name `implement-issue`, skill version from this file's front matter, and the [canonical SKILL.md URL](https://github.com/holos-run/skills/blob/main/plugins/linear-workflow/skills/implement-issue/SKILL.md).
   - `## Environment`: start and completion timestamps, execution connector or host when known, and relevant non-secret tool versions.
   - `## Escalation evidence`: why the output was unusable, the PR-comment posting status, and any method-selection or MCP qualification failures.
   - `## Attempt-by-attempt statuses`: one subsection per probe, CLI attempt, or MCP attempt. Record the exact command or remote invocation run; connector session ID (or `none`) and connector exit code; start and completion timestamps; whether the shell reached each status-recording line; `PROBE_STATUS` / `PROBE_PARSE`, `GH_DIFF_STATUS` / `CLAUDE_STATUS` / `EXTRACT_STATUS`, or `CODEX_STATUS` as applicable; stderr tails; result-envelope excerpts; and post-completion byte counts for the diff and every output, stderr, or envelope artifact. Use `not applicable` or `not produced` for fields a method does not have instead of omitting them.
   - `## Conclusion and confidence`: the most likely failure boundary, confidence level, and what remains unknown.

   Apply the same publication-safety rules required by the escalation PR comment below to the entire bundle: truncate every stderr tail to its last 20 lines, cap each result-envelope excerpt at approximately 2000 characters, and redact API keys, bearer tokens, `sk-` / `ghp_`-style secrets, URLs with embedded credentials, and other credential-shaped strings. Preserve exact commands only after applying those redactions. The full PR diff is evidence only by byte count; do not embed its contents in the Linear document.

2. **Post the existing path-specific PR escalation comment.** Render the caller's `Code Review Cannot Proceed` body with the compact, redacted attempt evidence that is available as `ESCALATION_COMMENT`, append the agent-attribution footer per the "Agent Attribution" section, then post it with the shared bounded invocation:

   ```bash
   timeout --kill-after=10 60 gh pr comment "$PR_NUMBER" --body "$ESCALATION_COMMENT"
   COMMENT_STATUS=$?
   ```

   If it fails, append `COMMENT_STATUS` to `ESCALATION_LINEAR_ERRORS` and continue.

3. **Attempt to create the related escalation issue.** Ensure `needs-human-review` exists for `TEAM_KEY`, then call `mcp__linear-server__save_issue` with:

   - `team: "<TEAM_KEY>"`
   - `title: "fix(implement-issue): code reviewer returned no output on PR #<PR_NUMBER> (<REPO>)"`
   - `labels: ["needs-human-review"]`
   - `relatedTo: ["<ISSUE_IDENTIFIER>"]`
   - `description` containing one short paragraph with the failure summary, reviewer path and model, PR/repository reference, skill name and front-matter version, the canonical SKILL.md URL above, a pointer to the attached document by its exact title (`Reviewer failure debug capture — PR #<PR_NUMBER> <REPO> (<ISSUE_IDENTIFIER>)`), and this exact ask: `Modify the implement-issue skill so this failure mode is prevented or diagnosable.`

   Capture the created issue's identifier as `ESCALATION_ISSUE_IDENTIFIER`. The issue description must remain planning context, not a copy of the debug bundle.

4. **Attempt to attach the bundle as a Linear document.** Only if step 3 returned an identifier, call `mcp__linear-server__save_document` with:

   - `title: "Reviewer failure debug capture — PR #<PR_NUMBER> <REPO> (<ISSUE_IDENTIFIER>)"`
   - `issue: "<ESCALATION_ISSUE_IDENTIFIER>"`
   - `content: <the redacted debug bundle Markdown>`

   Capture the document URL, slug, or ID as `ESCALATION_DOCUMENT_REFERENCE`; when the returned reference is linkable, make one best-effort `mcp__linear-server__save_issue` call to update the escalation issue description's document pointer with that link. A failed description update is recorded in `ESCALATION_LINEAR_ERRORS` and never halts the flow. Then post a comment on the original issue identified by `ISSUE_IDENTIFIER`, linking `ESCALATION_ISSUE_IDENTIFIER` and the attached document. If document creation fails, still post a comment linking the escalation issue and state that document attachment failed.

5. **Degrade gracefully.** Treat every Linear create, document, description-update, or comment call in steps 3–4 as a single best-effort attempt. If issue creation fails, do not attempt document creation; append the failure to `ESCALATION_LINEAR_ERRORS` and continue. If document creation, the description update, or the linking comment fails, append that failure and continue. Never retry indefinitely or let this flow prevent the existing PR-comment and label escalation. Apply `needs-human-review` to the PR and to `ISSUE_IDENTIFIER` (the original Linear issue), remove temporary reviewer artifacts, and skip to L16 with result `ESCALATED`; L16 reports any created escalation issue and every best-effort failure.

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

If found, each round in L9–L11 runs Codex as **one foreground, bounded Bash call** whose review is captured to a file. Four rules make this reliable — they exist because execution connectors can yield before process exit, while process-table completion checks can match the agent's own shell (or the `grep`/`pgrep` itself) and deadlock the session until the harness force-recovers it:

1. **Never background it.** Run `codex exec` as a single foreground Bash call. Do not set `run_in_background`, do not append `&`, do not `disown`.
2. **Never poll for completion.** Do not run `ps`, `pgrep`, `jobs`, `wait`, or read `/proc` to find Codex. `--output-last-message` writes the review to the output file atomically when Codex exits; there is nothing to poll.
3. **Bound it.** Wrap the call in `timeout 600`. Configure the host execution call and its cumulative connector waits to allow the inner timeout to finish and report its exit status. A completed round whose inner timeout expires is inconclusive (below); a connector yield before that point is merely still running.
4. **Wait through yields on the same session.** Completion means the host connector has reported an exit code, not that its initial call returned control. If the initial result contains a session handle such as `session_id` but no exit code, call the connector-native stdin/wait operation with empty input on that exact handle until an exit code is present. Never restart the Bash command. If the outer tool orchestration yields a `cell_id`, resume that same cell with its wait operation; it must continue the existing connector wait. These waits preserve the one foreground call and are neither process-table polling nor additional attempts.

Harness adapters should implement that fourth rule with their native field and operation names. For example:

```javascript
let run = await tools.exec_command({ cmd: reviewCommand, yield_time_ms: 30000 })
const sessionId = run.session_id
while (run.exit_code == null) {
  if (!sessionId) {
    throw new Error("execution yielded without an exit code or session handle")
  }
  run = await tools.write_stdin({
    session_id: sessionId,
    chars: "",
    yield_time_ms: 60000,
  })
}
// Parse artifacts only after exit_code is non-null.
```

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

Before interpreting the result, apply the completion gate: the host execution session must have an observed exit status. A connector `session_id` with no exit code, a status variable or status artifact not yet written, or an empty redirect-target file while the process runs means **still running**, never inconclusive. Continue waiting on the same session. Only a completed session with an observed exit status may be interpreted below or consume a retry attempt.

Interpret the completed result deterministically — never re-run merely to "check if it finished":

- **`CODEX_STATUS` is 0 and `$REVIEW_OUT` is non-empty:** that file is the round's review. Parse it for the verdict and per-severity findings and post it to the PR.
- **`CODEX_STATUS` is nonzero or `$REVIEW_OUT` is empty:** the round is inconclusive. Re-run this exact command **once**. If the rerun is also inconclusive, fall through to Method 2.

**Method 2 — Codex MCP server.** If the CLI invocation is inconclusive, check whether a Codex MCP server is connected to this session: use tool discovery with query `+codex` and look for a Codex conversation tool. Use it only if the latest frontier slug was resolved from the Codex catalog and the tool accepts an explicit `model` argument; invoke it with `model: "$CODEX_FRONTIER_MODEL"` and the review prompt, then parse the returned conversation text exactly like CLI output. A Codex tool whose model cannot be set to the resolved frontier slug does not satisfy this reviewer route.

Do not use `@codex review` as a fallback for this route because the GitHub integration does not accept the dynamically resolved frontier slug.

**If both methods fail:** do not fall back to Claude. Surface an explicit error and escalate now (do not proceed to L9–L11). Run the shared **Escalation debug capture** flow exactly once with `REVIEWER_PATH` identifying the Codex CLI and MCP qualification path, `REVIEWER_MODEL="$CODEX_FRONTIER_MODEL"` (or `unresolved`), and evidence from both completed CLI attempts plus the MCP availability/qualification result. Use the following path-specific PR-comment body in step 2, replacing each placeholder with compact, redacted evidence from the completed attempts:

```markdown
## Code Review Cannot Proceed

This implementation is routed to the latest Codex frontier reviewer, but no qualifying invocation method is available: the current frontier slug could not be resolved from `codex debug models`, `codex exec --model "$CODEX_FRONTIER_MODEL"` was inconclusive, or no connected Codex MCP tool could use the resolved frontier slug. The reviewer cannot silently downgrade to Claude because that may be the implementation model. Marking for human review.

Evidence:
- Failure point: <frontier resolution | CLI attempt 1 | CLI attempt 2 | MCP qualification or invocation>
- Statuses: <CODEX_STATUS values and MCP availability / explicit-model qualification / invocation status>
- stderr tails (last 20 lines each, redacted): <captured tails, or "empty">
- Result excerpt (truncated, redacted): <CLI output and MCP result excerpts, or "unparseable">
```

The shared flow attempts the related escalation issue and debug document, applies the PR and Linear labels, and skips directly to L16 with result `ESCALATED` even if its Linear creates fail.

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

Apply the same completion gate and final-line verdict requirement used by the CLI reviewers. If the configured command completes with empty review text or without a conclusive final verdict, record its exact expanded command, connector session and exit evidence, timestamps, stdout/stderr byte counts, and redacted stderr tail, then rerun that exact command once. If the rerun is also inconclusive, run the shared **Escalation debug capture** flow exactly once with `REVIEWER_PATH=project-configured`, the configured model if determinable (otherwise `unknown`), and both attempts' evidence. Use this path-specific PR-comment body in step 2, replacing each placeholder with compact, redacted evidence:

```markdown
## Code Review Cannot Proceed

The project-configured reviewer completed twice without producing usable review output or a conclusive final verdict. Substituting another reviewer could violate the configured cross-runtime review policy. Marking for human review.

Evidence:
- Failure point: <project-configured review round N, attempts 1 and 2>
- Command: <exact expanded command, redacted>
- Statuses: <connector session / exit status and completion-gate evidence for each attempt>
- Output sizes: <stdout and stderr byte counts for each attempt>
- stderr tails (last 20 lines each, redacted): <captured tails, or "empty">
- Result excerpts (truncated, redacted): <review output excerpts, or "empty">
```

Do not substitute another reviewer. The shared flow applies the labels and skips to L16 with result `ESCALATED`, even if creating the related issue or document fails.

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

The probe succeeds only when `PROBE_STATUS` and `PROBE_PARSE` are both 0. If `claude` or `jq` is missing, or the probe fails, the Claude reviewer is unavailable. Run the shared **Escalation debug capture** flow exactly once with `FAILURE_POINT=probe`, the probe's exact command, connector-session evidence, `PROBE_STATUS` / `PROBE_PARSE`, byte counts, stderr tails, and envelope evidence. Use the Claude `Code Review Cannot Proceed` body below as the path-specific PR-comment body. The shared flow applies `needs-human-review` to the PR and the original issue identified by `ISSUE_IDENTIFIER`, then skips to L16 with result `ESCALATED`, even if creating the related issue or document fails. Do not invoke Codex or inherit the current session model, and do not spend review rounds discovering a statically broken CLI.

If the probe succeeds, each round in L9–L11 runs Claude Code as **one foreground, bounded call**, under the same four rules as the Codex CLI reviewer: never background it, never poll the process table for completion, bound every network-dependent command with `timeout --kill-after=<grace> <seconds>`, and wait through connector yields on the exact same session until the connector reports an exit code. A foreground tool session may yield control before process exit; that yield is still the first attempt and waiting on it with the connector-native stdin/wait operation is not backgrounding, polling, or another attempt. If an outer orchestration cell yields, resume its `cell_id` with the outer wait operation so it continues waiting on the existing shell session. The inner `timeout --kill-after=10 60` diff fetch and `timeout --kill-after=15 490` Claude call remain authoritative: cumulative connector waits must allow enough time to observe their exit status, including kill grace, rather than imposing a shorter host deadline. A host that permits a longer Claude bound may raise it only when the combined inner worst case remains strictly below the host deadline.

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

**Apply the completion gate before inspecting any status variable or artifact.** The host connector must report an exit code for the foreground Bash call. If it instead reports a `session_id` with no exit code, continue waiting on that exact session with empty connector-native stdin/wait calls. If the outer orchestration yields a `cell_id`, resume that cell with its own wait operation. Until an exit code is observed, unset `GH_DIFF_STATUS` / `CLAUDE_STATUS` / `EXTRACT_STATUS`, missing status artifacts, and empty `review.json` / `review.err` redirect targets are expected signs that the process is still running — never evidence of an inconclusive attempt. Do not read or parse the artifacts, restart the command, or charge a retry while the session is live.

After the completion gate passes, interpret the result deterministically — never re-run merely to "check if it finished". A round is **conclusive** only when ALL of the following hold; a partial `gh` diff, a truncated JSON stream rescued by `jq`, or review text with no verdict must never pass as a completed review:

- `GH_DIFF_STATUS` is 0 and `$REVIEW_DIR/pr.diff` is non-empty (a diff failure is a `gh` failure — `CLAUDE_STATUS` reads `not-run` so the evidence names the failing component, never Claude);
- `CLAUDE_STATUS` is 0;
- `EXTRACT_STATUS` is 0 and `$REVIEW_DIR/review.txt` is non-empty;
- `VERDICT` is non-empty. The verdict comes only from the review's **final nonblank line**, which the prompt demands be a dedicated `Verdict:` line (the extraction tolerates surrounding Markdown emphasis). Prose that merely mentions a verdict token, an echoed rubric, a refusal with an earlier standalone verdict, or a token embedded in another word (`DISAPPROVE`) can never pass.

**Conclusive:** `$REVIEW_DIR/review.txt` is the round's review with verdict `$VERDICT`. Parse the per-severity findings and post it to the PR. **Anything else after observed connector completion — the round is inconclusive.** Before retrying, record these fields separately so a connector yield can never be mistaken for an empty Claude result:

- connector session ID (or `none` when the call never yielded) and connector exit code;
- attempt start and completion timestamps;
- whether the shell reached the `GH_DIFF_STATUS`, `CLAUDE_STATUS`, and `EXTRACT_STATUS` recording lines;
- `GH_DIFF_STATUS`, `CLAUDE_STATUS` (124 means `timeout` expired and `SIGTERM` ended the call; 137 means the call ignored `SIGTERM` and `--kill-after` sent `SIGKILL`), and `EXTRACT_STATUS`;
- byte counts for `pr.diff`, `gh-diff.err`, `review.json`, `review.err`, `review.txt`, and `review-jq.err`, measured only after connector completion;
- the tails of `review.err`, `gh-diff.err`, and `review-jq.err`, plus the envelope's `subtype`/`is_error` from `review.json` if it parsed.

Then rerun the exact foreground command once. If the rerun is also inconclusive, do not invoke Codex or inherit the current session model. Run the shared **Escalation debug capture** flow exactly once with both attempts' complete evidence and the Claude `Code Review Cannot Proceed` body below as the path-specific PR-comment body. The shared flow applies the labels and skips to L16 with result `ESCALATED`, even if creating the related issue or document fails.

**Escalation comment body (probe failure and inconclusive rounds alike).** The body must carry the captured evidence — never only a prose conclusion like "completed without producing review output", which leaves the failure undiagnosable. Before posting, truncate each stderr tail to its last 20 lines, cap the result-envelope excerpt at ~2000 characters, and redact anything credential-shaped (API keys, bearer tokens, `sk-`/`ghp_`-style strings, URLs with embedded credentials) — CLI diagnostics can leak local paths and account details, and a PR comment is public within the repo:

```markdown
## Code Review Cannot Proceed

This Codex implementation requires review by the latest Claude Opus model (`--model $CLAUDE_REVIEW_MODEL`). The reviewer cannot silently downgrade to Codex because that would be same-family self-review. Marking for human review.

Evidence:
- Failure point: <probe | review round N attempt M>
- Statuses: <GH_DIFF_STATUS / CLAUDE_STATUS / EXTRACT_STATUS, or PROBE_STATUS / PROBE_PARSE; note 124 = timed out at <bound>s, 137 = SIGKILL after ignoring SIGTERM>
- stderr tails (last 20 lines each, redacted): <from the captured stderr files, or "empty">
- Result envelope (truncated, redacted): <subtype / is_error / result excerpt, or "unparseable">
```

The shared escalation flow posts this body with its bounded invocation in step 2. If `COMMENT_STATUS` is nonzero, do not stall: record it in `ESCALATION_LINEAR_ERRORS`, still attempt the related Linear issue and document, still remove `$REVIEW_DIR`, still apply the `needs-human-review` labels, and still finish with result `ESCALATED`.

For automatic Codex-runtime review, the PR comment header must identify `opus (latest Claude Opus)`, not `claude session model`. This makes accidental regression to the behavior in the linked failure visible without pinning a model version.

### L9. Round 1: Review and Fix

Run the reviewer selected in L8. Parse the output for:

- **Verdict**: APPROVE or REQUEST_CHANGES
- **Finding counts** by severity: CRITICAL, IMPORTANT, STYLE

Post the review back to the PR as a comment per the "Posting the review back to the PR" requirement in L8. This applies to every round — L9, L10, and L11.

If a reviewer path exhausts its permitted attempts without usable output in any round, follow L8's shared **Escalation debug capture** flow and stop without merging; do not treat absent output as approval or continue to CI.

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
- Reviewer escalation: <ESCALATION_ISSUE_IDENTIFIER> (if reviewer-output escalation created one)
- Escalation attachment: <ESCALATION_DOCUMENT_REFERENCE> (if created)
- Escalation errors: <ESCALATION_LINEAR_ERRORS> (if any best-effort step failed)
```

---

## Parent Mode

Orchestrator for a parent issue with children. The orchestrator runs in the top-level harness the user invoked and dispatches each child issue to a worker sub-agent of that **same harness** (inheriting the session model unless `--model` overrides it), per the "Arguments and Model Selection" resolution rules. It never hands implementation to a different harness.

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

Process sub-issues **sequentially** (each phase depends on the previous one). Each open child is dispatched to a worker sub-agent of the orchestrator's own harness — the model comes from `MODEL_OVERRIDE` or the session default, never from issue labels. The dispatched worker runs the full leaf lifecycle for that one sub-issue and returns a short result summary.

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

**Dispatch per sub-issue** — always a sub-agent of the orchestrator's own harness, with the model from the "Arguments and Model Selection" resolution (`MODEL_OVERRIDE` if set, else the session default). Issue labels play no part. Never dispatch implementation to a different harness than the one the user invoked.

In every worker template below (both harnesses, initial dispatch and retries), the skill invocation inside the prompt must propagate the overrides per "Arguments and Model Selection": append ` --model <MODEL_OVERRIDE>` when `MODEL_OVERRIDE` is set and ` --reviewer <REVIEWER_OVERRIDE>` when `REVIEWER_OVERRIDE` is set. CLI-level flags on `codex exec` do not reach the child skill's argument parsing — only flags written into the `/linear-workflow:implement-issue` invocation text do.

If `PRIMARY_RUNTIME=claude`, spawn a Claude Code sub-agent:

```
Agent(
  description: "Implement <SUB_IDENTIFIER>",
  model: "<RESOLVED_MODEL — omit entirely for the session default>",
  prompt: "Invoke /linear-workflow:implement-issue <SUB_IDENTIFIER><propagated flags> to implement
  this sub-issue end-to-end. The skill handles branching, implementation, code review, CI, merge,
  and issue transitions. Run to completion. Return a short summary: result (MERGED |
  MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED), PR number, and any follow-up issue identifier."
)
```

If `PRIMARY_RUNTIME=codex`, spawn the worker as a `codex exec` subprocess — still the Codex harness. Resolve `WORKER_MODEL` as `MODEL_OVERRIDE` when set; otherwise, if the orchestrator knows the model its own session is running, use that; otherwise leave `WORKER_MODEL` unset and omit `--model` so the worker uses the Codex CLI's configured default (a fresh `codex exec` reads configuration — it cannot see a model selected interactively in the invoking session, so propagate the session model explicitly whenever it is known). Run it under the same foreground/completion-gate rules as every `codex exec` call in this skill:

```bash
set --
[ -n "${WORKER_MODEL:-}" ] && set -- --model "$WORKER_MODEL"
codex exec "$@" --dangerously-bypass-approvals-and-sandbox \
  "Invoke /linear-workflow:implement-issue <SUB_IDENTIFIER><propagated flags> to implement this
sub-issue end-to-end. The skill handles branching, implementation, code review, CI, merge, and
issue transitions. Run to completion. Return a short summary: result (MERGED |
MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED), PR number, and any follow-up issue identifier."
```

(The `set --` argument-list form keeps the optional `--model` flag as separate words in any POSIX shell — `${VAR:+...}` inline expansion is not portable to zsh, which would pass `--model <name>` as a single argument.)

If `PRIMARY_RUNTIME=unknown`, dispatch through whatever native sub-agent mechanism the current harness provides — never launch a different harness's CLI to run implementation.

Wait for the dispatched worker to complete before starting the next one.

#### Handling Usage Limits

Usage limits apply when a worker's output indicates capacity is exhausted — for example output containing phrases like "usage limit reached", "rate limit exceeded", "quota exceeded", or "you have reached your usage limit" — without returning a valid implementation result (`MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED`). Do not count usage-limit events as stuck-worker retry attempts.

**When any worker hits a usage limit (either harness, any model):**

1. Do not increment the retry counter.
2. Parse the earliest "retry after" time from all usage-limit messages (e.g., "available again at HH:MM UTC", "retry in N minutes", "resets at HH:MM"). Convert to seconds until that time.
3. If no retry time is parseable, default to 15 minutes (`900` seconds).
4. Wait (`sleep <seconds_until_retry>`) — do not prompt the user.
5. Re-dispatch the sub-issue in the **same harness** with its original model resolution (`MODEL_OVERRIDE` if set, else session default). Never switch harnesses to dodge a usage limit — that would move implementation out of the harness the user invoked.

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
4. Dispatch a replacement worker in the same harness with the same model resolution (`MODEL_OVERRIDE` if set, else session default), using the same sub-issue identifier plus a warning. Do not describe implementation steps — just provide context and let the skill decide what to do.

   If `PRIMARY_RUNTIME=claude` (omit `model` for the session default, as always):
   ```
   Agent(
     description: "Implement <SUB_IDENTIFIER> (retry <N>)",
     model: "<RESOLVED_MODEL — omit entirely for the session default>",
     prompt: "Invoke /linear-workflow:implement-issue <SUB_IDENTIFIER><propagated flags>.

     Warning: A previous attempt did not complete. Point: <e.g. 'wrote files but did not commit'>. Reason: <e.g. 'worker returned without a result summary'>.

     Invoke the skill and let it run to completion. Return a short result summary: result (MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED), PR number, and any follow-up issue identifier."
   )
   ```

   If `PRIMARY_RUNTIME=codex` (resolve `WORKER_MODEL` exactly as in the initial P6 dispatch, using the same portable `set --` form for the optional flag):
   ```bash
   set --
   [ -n "${WORKER_MODEL:-}" ] && set -- --model "$WORKER_MODEL"
   codex exec "$@" --dangerously-bypass-approvals-and-sandbox \
     "Invoke /linear-workflow:implement-issue <SUB_IDENTIFIER><propagated flags>.

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

Process follow-ups with the **full P6 pre-dispatch sequence** — including the *Pre-Dispatch: Detect and Discard Partial Work* step — then dispatch one worker per follow-up using the same-harness dispatch rules from P6.

### P8. Nested Parent Issues

If a sub-issue or follow-up is itself a parent (has its own children), the dispatched worker invoked in P6 will simply re-enter this skill in Parent Mode and orchestrate its own children with the same rules — including the P6 pre-dispatch partial-work cleanup for each grand-child. Because P6 workers are sub-agents of the same harness, every nested orchestrator also runs in the harness the user originally invoked. Nested orchestration composes naturally — no special handling is needed for depth.

Nested parent orchestrators follow the same resolution: `MODEL_OVERRIDE` (propagated via `--model`) first, otherwise inherit the session-configured model.

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
| Selected reviewer path produces no usable output after its permitted attempts | Do NOT merge; capture diagnostics, attempt a related escalation issue and document, and apply `needs-human-review` |

## Prerequisites

- **Linear MCP**: `mcp__linear-server__*` tools configured for normal issue transitions and, on selected-reviewer output failure, best-effort related issue creation plus issue-backed document creation
- **GitHub CLI**: `gh` authenticated with repo access
- **Git**: Clean working directory
- **Same-harness implementation**: The orchestrator and every implementation worker run in the harness the user invoked. Only code review crosses harnesses. `--reviewer` is the only reviewer-routing override. `--model` affects implementation only, and only the model — never the harness.
- **Codex access**: Required for review when the user invoked the skill in Claude Code: the `codex` CLI and `jq` must resolve the latest frontier slug from `codex debug models`; `codex exec --model "$CODEX_FRONTIER_MODEL"` is preferred, and a model-selectable Codex MCP server may be used after an inconclusive CLI review. When the user invoked the skill in Codex, `codex exec` is also how Parent Mode spawns its same-harness workers.
- **Claude access**: The `claude` CLI must be on `PATH` and authenticated when a Codex implementation is automatically paired with the latest Claude Opus. L8 proves this end-to-end with a bounded `--print --output-format json` probe before the first round; `jq` is required to parse the result envelope. Reviewer-output failure escalates to human review with a redacted debug bundle plus best-effort creation of a related Linear issue and attached document; it never falls back to the implementation model family.

## Linear API Cheat Sheet

| Action | Tool | Key arguments |
|--------|------|---------------|
| Fetch issue (incl. blocking relations) | `mcp__linear-server__get_issue` | `id` |
| List children | `mcp__linear-server__list_issues` | `parentId` |
| Update issue | `mcp__linear-server__save_issue` | `issue`, `state` / `labels` / `description` |
| Create issue | `mcp__linear-server__save_issue` | `team`, `title`, `description`, optional `relatedTo` |
| Create sub-issue | `mcp__linear-server__save_issue` | `team`, `parentId`, `title`, `description` |
| Create issue document | `mcp__linear-server__save_document` | `issue`, `title`, `content` |
| Post comment | `mcp__linear-server__save_comment` | `issue`, `body` |
| List comments | `mcp__linear-server__list_comments` | `issueId` |
| List statuses | `mcp__linear-server__list_issue_statuses` | `team` |
| List / create labels | `mcp__linear-server__list_issue_labels` / `mcp__linear-server__create_issue_label` | `team` |
