---
name: implement-issue
description: v1.0.0 — Implement a plan item tracked as a markdown file in the repository's plans folder, either as one leaf item or as a primary plan orchestrating its phases. No external tracker — status, labels, and activity are recorded in the plan files and every change is committed and pushed to main as it happens. All implementation work — orchestrator and phase workers alike — runs in the harness the user invoked; --model adjusts only the model within that harness, and only code review crosses to the opposite harness. Cross-runtime review posts findings to the PR; reviewer-output failures stop the merge and produce redacted diagnostics plus a best-effort escalation plan item. Every PR comment and plan-file activity entry the skill writes ends with an agent-attribution footer naming the harness, model, and reasoning effort (for example `claude fable-5 high`) so colleagues know they are reading agent output, not the human account owner. Use --reviewer only to override reviewer selection. Triggers when the user provides a plan item identifier (for example 007 or 007-02) or a path to a plan file and asks to implement, work on, fix, resolve, or execute its plan.
version: 1.0.0
# Guardrail: whenever version changes, update the leading vX.Y.Z prefix in description in the same PR.
---

# Implement Issue

Implement a plan item end-to-end. A plan item is a markdown file with YAML front matter in the repository's plans folder (see Plan File Conventions). This skill self-detects whether the item is a leaf (no phase files beneath it) or a parent (has phase files) and adapts its behavior:

- **Leaf mode**: Branch, implement, open PR, run adversarial cross-runtime code review (the latest Claude Opus reviews Codex implementations; the Codex frontier model reviews Claude implementations; `--reviewer` explicitly overrides), post the review back to the PR as a comment, and allow up to 2 fix rounds. If an invoked reviewer produces no usable output, capture diagnostics, write a best-effort escalation plan item, and stop for human review without merging; otherwise wait for CI, merge, and mark the item done.
- **Parent mode**: Orchestrate implementation of all phase items — each worker runs as a sub-agent of the same harness the orchestrator runs in, inheriting the session model unless the `--model` argument overrides it — then track results, sweep for follow-ups, and record a summary.

**The harness never changes for implementation.** The orchestrator always runs in the top-level harness the user invoked, and every phase worker is a sub-agent of that same harness. If the user starts in Codex, all implementation runs in Codex; if the user starts in Claude Code, all implementation runs in Claude Code. The only cross-harness dispatch this skill ever performs is adversarial code review.

Implement the plan item **{{SKILL_INPUT}}**.

## Arguments and Model Selection

`{{SKILL_INPUT}}` contains the item reference and optional flags:

```
<item> [--model <name>] [--reviewer <codex|claude|fable|opus|sonnet|haiku>]
```

Parse and record:

- The item reference (identifier like `007` / `007-02`, or a path to a plan file) — strip any flags before resolving it in step 1.
- `MODEL_OVERRIDE` — the value of `--model <name>` (also accepted as `model=<name>`) if present; otherwise unset. The name must be one the **current harness** understands for its own sub-agents (e.g., `fable`/`opus`/`sonnet`/`haiku` in Claude Code; a Codex model slug in Codex). `--model` selects a model **within** the invoking harness — it never selects a harness.
- `REVIEWER_OVERRIDE` — the value of `--reviewer <name>` (also accepted as `reviewer=<name>`) if present; otherwise unset. Controls **only** the code reviewer selection in L8 — it never affects worker or orchestrator dispatch. `codex` selects the Codex frontier reviewer; `claude` or `opus` selects the latest Claude Opus through the `opus` alias; `fable`/`sonnet`/`haiku` explicitly select that Claude model instead.

Every point in this skill that dispatches **implementation work** — phase workers (P6), nested orchestrators (P8), and retry dispatches — resolves the model with this priority. Code review is deliberately independent: L8 uses `REVIEWER_OVERRIDE` when present and otherwise selects the opposite model family from the detected primary runtime. `MODEL_OVERRIDE` never selects the reviewer.

1. **`MODEL_OVERRIDE`** — the `--model` argument always wins.
2. **Session default** — no override: the worker runs on the invoking session's model. In Claude Code, spawn the sub-agent **without a `model` parameter** so it inherits the session model. In Codex, pass the session's own model explicitly when known (a fresh `codex exec` cannot see it otherwise), and only fall back to omitting `--model` — the Codex configured default — when it is not. This is the normal path — never hardcode a fallback model.

Plan-file labels never influence dispatch. Do not read, match, or honor any label as a routing signal — implementation routing comes only from `MODEL_OVERRIDE` and the session default, and the harness is always the one the user invoked.

Workers are always dispatched through the current harness's native sub-agent mechanism: in Claude Code, the `Agent()` tool; in Codex, a `codex exec` subprocess (still the Codex harness). In the worker templates below, `model: "<RESOLVED_MODEL>"` means: pass `MODEL_OVERRIDE` when set, and **omit the model parameter entirely** when resolution fell through to the session default.

When this skill re-invokes itself for a phase or nested parent, propagate `MODEL_OVERRIDE` and `REVIEWER_OVERRIDE` in the invocation when set: `/simple-workflow:implement-issue <SUB_ID> --model <name> --reviewer <name>` (include each flag only when its override is set).

## Plan File Conventions

This skill operates on markdown files in the repository. It uses no tracker MCP tools; GitHub is used only for PRs, CI, and merges via `gh`.

- **Plans folder** (`PLANS_DIR`) — resolved in step 1: the directory named by a `Plans directory: <path>` / `Plans folder: <path>` / `plans_dir: <path>` line or a `## Plans` section in the repository's `CLAUDE.md` or `AGENTS.md`; otherwise top-level `plans/`.
- **Item** = one markdown file in `PLANS_DIR` with YAML front matter. Equivalent to an issue in a tracker.
- **Primary plan** = `<PLANS_DIR>/<NNN>-<slug>.md`. **Phase** = `<PLANS_DIR>/<NNN>-<slug>/<PP>-<phase-slug>.md`. A child's directory is always the parent's file path minus `.md`, so **children of item X live in `<X path minus .md>/`**.
- **Identifier** = `NNN` for a primary plan, `NNN-PP` for a phase, `NNN-PP-QQ` for a nested phase (`^[0-9]{3,}(-[0-9]{2})*$`).
- **TODO list** = `<PLANS_DIR>/TODO.md`, one line per item, updated in the same status commit as the item.
- **Activity** = the `## Activity` section at the end of each item; entries there replace tracker comments.
- **PR** = GitHub pull request, opened via `gh`. The PR body must contain `Plan: <ITEM_ID> (<ITEM_PATH>)` so readers can find the item; nothing auto-closes, so this skill marks the item done itself after merge.
- **`main`** means the repository's default branch. Substitute `master` or another name when the repository uses one.
- **Source of truth for status is `origin/main`.** Before reading any item's status, run `git fetch origin` and read the file with `git show origin/main:<path>`. The current worktree may be on a feature branch that predates recent status commits.
- Every PR comment and every Activity entry this skill writes must end with the agent-attribution footer defined in the "Agent Attribution" section, even where a template below does not repeat it inline.

### Front matter schema

```yaml
---
id: "007-02"
title: "feat(auth): add login schema"
status: todo          # todo | in-progress | done | canceled
labels: []            # planning | implementing | needs-human-review | escalation
parent: "007"         # omitted on primary plans
blocked_by: []        # identifiers that must be done or canceled first
related: []           # identifiers of related items
branch: ""            # feature branch once implementation starts
pr: ""                # PR number once opened
created: 2026-09-07
updated: 2026-09-07
---
```

Quote identifiers so YAML never parses `007` as `7`. Update `updated` whenever any other field changes. Status types map to tracker semantics as: `todo` = unstarted, `in-progress` = started, `done` = completed, `canceled` = canceled.

### TODO.md format

Nested task list under `## Open` with primary plans as top-level bullets and phases indented beneath; finished plans move to `## Done`. Line format:

```
- [ ] **<id>** <title> — `<status>`[ `<label>`...] — [<relative path>](<relative path>)
```

Tick the box when status is `done` or `canceled`. Show labels as additional backticked tokens after the status. When a primary plan and all of its phases are done or canceled, move its whole block to `## Done`. Create the file with a header, `## Open`, and `## Done` when missing. If the list disagrees with a file's front matter, the front matter wins — fix the list.

## Recording Status Updates

Plan files live on `main`. Every change this skill makes to `PLANS_DIR` lands on `main` through a **status commit**: a commit built on the current `origin/main` tip in a temporary detached worktree, pushed straight to `main`. Feature branches never touch `PLANS_DIR` — code PRs contain only code, and status is visible on `main` the moment it changes. This never checks out `main` in the current worktree.

Procedure (`RECORD_STATUS`), run for each logical status change:

```bash
git fetch origin
STATUS_WT=$(mktemp -d)
if [ -z "$STATUS_WT" ] || [ ! -d "$STATUS_WT" ]; then
  echo "mktemp -d failed: cannot stage a status commit" >&2
  exit 1
fi
git worktree add --detach "$STATUS_WT" origin/main
# Edit files under "$STATUS_WT/$PLANS_DIR" with the file tools — update front
# matter, append Activity entries, create follow-up or escalation items, update
# TODO.md. Never edit anything outside PLANS_DIR here.
git -C "$STATUS_WT" add -- "$PLANS_DIR"
git -C "$STATUS_WT" commit -m "chore(plans): <summary of the status change>

Refs: <identifiers touched>"
git -C "$STATUS_WT" push origin HEAD:main
PUSH_STATUS=$?
```

Then:

1. **Push succeeded** — remove the worktree and continue:
   ```bash
   git worktree remove --force "$STATUS_WT"
   git worktree prune
   git fetch origin
   ```
2. **Push rejected as non-fast-forward** (someone pushed to `main` meanwhile) — rebase and retry, up to 3 attempts total:
   ```bash
   git -C "$STATUS_WT" fetch origin
   git -C "$STATUS_WT" rebase origin/main
   git -C "$STATUS_WT" push origin HEAD:main
   ```
3. **Push rejected by branch protection** (the remote refuses direct pushes to `main`) — fall back to a pull request:
   ```bash
   STATUS_BRANCH="plans/status-$(date +%Y%m%d%H%M%S)"
   git -C "$STATUS_WT" push origin "HEAD:refs/heads/$STATUS_BRANCH"
   gh pr create --head "$STATUS_BRANCH" --base main \
     --title "chore(plans): <summary>" \
     --body "Automated plan status update from simple-workflow."
   gh pr merge --merge --delete-branch "$STATUS_BRANCH"
   ```
   If the merge is refused (required reviews or checks), leave the PR open, note its URL in the final report, and continue — the status is recorded on the branch and pending on `main`.
4. **Anything else fails** — remove the worktree, report the error, and stop. Never work around it by committing plan files to the current feature branch.

Honor the repository's commit signing configuration when committing (for example `git commit -S` where signed commits are required). Never pass `--no-gpg-sign`.

If the session's current branch is `main` itself (the skill was invoked from the primary checkout rather than a worktree), fast-forward it after every successful status commit:

```bash
[ "$(git rev-parse --abbrev-ref HEAD)" = "main" ] && git pull --ff-only origin main
```

**Activity entry format** (append to the end of the `## Activity` section; create the section if missing):

```markdown
### <ISO-8601 UTC timestamp> — <short heading>

<body>

🤖 <AGENT_SIGNATURE> — automated entry written by an AI agent on behalf of this repository's operator.
```

Wherever this skill says "post an Activity entry" or "set status/labels" on an item, it means: run `RECORD_STATUS` with those edits plus the matching `TODO.md` line update. Group edits that belong to one logical transition into one status commit.

## Worktree Conventions

This skill runs inside whatever checkout the harness provides — a Claude Code remote session worktree, a local worktree, or the repository's primary checkout. The conventions that follow:

- **Never switch to `main` to do work.** A branch can only be checked out in one worktree at a time, and `main` usually belongs to the repo's primary worktree. Refresh state with `git fetch origin` and reference `origin/main` directly. Status commits reach `main` through the detached worktree in `RECORD_STATUS`, never through a local checkout of `main`.
- **Create every feature branch from `origin/main`**, regardless of which branch the harness checked out here. This is the skill's own guarantee of a correct base branch.
- **Match a branch to an item by identifier**: a branch belongs to item `<id>` when its lowercased name contains `<id>-` after a slash (for example `feat/007-02-login-handler`).
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

Colleagues reading plan files and GitHub PRs must be able to tell at a glance that a comment or activity entry came from an AI agent operating on behalf of a human, not from the human account owner. Every comment and Activity entry this skill writes carries an attribution footer.

Immediately after Primary Runtime Detection, resolve these once and reuse them for the whole invocation:

- `AGENT_HARNESS` — the harness running this skill: `claude` when `PRIMARY_RUNTIME=claude`, `codex` when `PRIMARY_RUNTIME=codex`. When the runtime is unknown, use the native host's self-reported name if it provides one; otherwise `unknown-harness`.
- `AGENT_MODEL` — the model slug this session is actually running (for example `fable-5`, `opus-4-6`, `solstice-alpha`). Take it from the harness's native self-identity: Claude Code sessions know their model ID; a Codex session knows its configured model slug. Use `unknown-model` when it cannot be determined — never guess a plausible slug.
- `AGENT_EFFORT` — the reasoning-effort setting (for example `low`, `medium`, `high`, `xhigh`) when the harness exposes one for this session; omit the token entirely when unknown.

Compose the signature by joining the parts with single spaces:

```
AGENT_SIGNATURE = <AGENT_HARNESS> <AGENT_MODEL>[ <AGENT_EFFORT>]
```

Examples: `claude fable-5 high`, `codex solstice-alpha high`, `claude sonnet-4-5`.

**Footer requirement.** Every PR comment body this skill sends — via `gh pr comment` and `gh pr close --comment` — must end with a blank line, a `---` separator line, and this line:

```
🤖 <AGENT_SIGNATURE> — automated comment posted by an AI agent on behalf of this repository's operator. Replies here reach an agent, not the human directly.
```

Every Activity entry written to a plan file must end with a blank line and this line:

```
🤖 <AGENT_SIGNATURE> — automated entry written by an AI agent on behalf of this repository's operator.
```

This applies to **every** comment and Activity template in this skill, including templates that do not repeat the footer inline. Item bodies, titles, PR bodies, and escalation debug bundles are exempt — only comments and Activity entries carry the footer.

Sub-agent workers dispatched in Parent Mode run this skill themselves and resolve their own signature: the harness matches the orchestrator's, but `AGENT_MODEL` may differ when `--model` is set, and each worker's footer must state the model that worker actually runs.

Review-round PR comments identify two models, and both must be present: the `Code Review — Round <N>` heading names the reviewer that produced the findings (its resolved slug, for example `codex solstice-alpha` or `claude opus`), and the footer names the agent that posted the comment.

## Claude Review Model Mapping

When a Codex implementation is reviewed by Claude, always invoke Claude Code with `--model opus`. Claude Code defines `opus` as the alias for the latest available Claude Opus model. Do not pin a version-specific Opus model, inherit the parent session model, or use a lower-cost fallback. Run the Claude CLI in the foreground, bound it with `timeout`, and capture its final text before continuing. Invoke it with `--print --output-format json` so success is machine-checkable from the result envelope, capture stderr to a file, and prove the CLI end-to-end with a cheap probe before the first review round. The L8 Claude reviewer applies this in full detail.

---

## Mode Detection

### 1. Resolve the Plans Folder and the Item

Resolve `PLANS_DIR` per Plan File Conventions (repository-indicated directory, else top-level `plans/`). If the directory does not exist, tell the user there are no plans to implement and stop.

Parse `{{SKILL_INPUT}}` to extract the item reference (after stripping the flags described in "Arguments and Model Selection"). Accept either form:

- Identifier: `007` or `007-02`
- Path: `plans/007-add-login/02-login-handler.md` (relative to the repository root)

Run `git fetch origin`, then locate the file: for an identifier, the file under `PLANS_DIR` whose front matter `id` matches (its path also encodes it: `<NNN>-*.md` or `<NNN>-*/<PP>-*.md`). Read it from `origin/main` with `git show origin/main:<path>`. If it does not exist on `origin/main`, tell the user and stop.

Record:

- `ITEM_PATH` — repository-relative path of the item file
- `ITEM_ID` — front matter `id`, e.g., `007-02`
- `ITEM_TITLE`
- `ITEM_DIR` — `ITEM_PATH` minus `.md`; the directory where this item's children live
- `EXISTING_LABELS`
- `PARENT_ID` — front matter `parent`, if present (this item may be a phase of a plan); `PARENT_PATH` is then the file whose `id` equals `PARENT_ID`

### 2. Check for Blocking Items

Before entering any implementation mode, check whether the item is blocked by other open items.

Read the item's `blocked_by` list. For each identifier, locate its file and read its front matter from `origin/main`. Record:

- `BLOCKER_ID`
- `BLOCKER_TITLE`
- `BLOCKER_STATUS` — `todo`, `in-progress`, `done`, or `canceled`

Filter to **active blockers**: those whose status is NOT `done` and NOT `canceled`. A `blocked_by` identifier with no matching file is reported as a broken reference and treated as active until a human fixes it.

**If active blockers exist:**

1. Post an Activity entry on the item headed `Blocked`:

   ```
   This item is blocked by the following open items and cannot be implemented yet:

   - <BLOCKER_ID>: <BLOCKER_TITLE> (status: <status>)
   [... one line per active blocker]

   Waiting for all blockers to reach done or canceled before proceeding.
   ```

2. **Poll until unblocked.** Repeat the following loop until all active blockers are resolved:

   a. Wait 60 seconds (`sleep 60`).
   b. `git fetch origin`, then for each remaining active blocker re-read its status from `origin/main`.
   c. Remove any blocker whose status is now `done` or `canceled`.
   d. If the active list is empty, break out of the loop.

3. After the loop exits (all blockers resolved), post an Activity entry headed `Unblocked`:

   ```
   All blocking items are now done or canceled. Proceeding with implementation.
   ```

**If no active blockers:** continue immediately to step 3.

### 3. Check for Children

List `<ITEM_DIR>/*.md` on `origin/main` (`git ls-tree --name-only origin/main "<ITEM_DIR>/"`).

- **Child files exist** → enter **Parent Mode** (jump to the Parent Mode section below)
- **No child files** → enter **Leaf Mode** (continue to the next section)

---

## Leaf Mode

Full lifecycle for implementing a single item with no children.

### L1. Start Wall Clock Timer

```bash
ISSUE_START_TIME=$(date +%s)
```

### L2. Create Branch

```bash
git fetch origin
git checkout -b feat/<ITEM_ID>-<slug> origin/main
```

Branch naming: `feat/<ITEM_ID>-<slug>` where slug is the item title in lowercase, spaces replaced by hyphens, special characters stripped, truncated to ~40 chars.

**Never switch to `main` locally.** Per Worktree Conventions, branch directly from `origin/main` instead, no matter which branch the harness checked out here. The `git checkout -b <name> origin/main` form creates the branch from `origin/main`'s tip and sets its upstream tracking to `origin/main`.

Always push with `git push origin HEAD` (no `-u`) so the upstream tracking stays on `origin/main`. The PR is opened against `main` regardless.

### L3. Announce on the Item

Run `RECORD_STATUS` with these edits to `ITEM_PATH` (commit summary `chore(plans): start <ITEM_ID>`):

- `status: in-progress`
- `labels`: add `implementing` to `EXISTING_LABELS`
- `branch: feat/<ITEM_ID>-<slug>`
- `updated`: today
- Append an Activity entry headed `Working on this item`:

```
Working on this item.

- Branch: feat/<ITEM_ID>-<slug>
```

- Update the TODO line to show `in-progress` `implementing`

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

Commit messages should follow the project's format (from CONTRIBUTING.md or CLAUDE.md). Include the item identifier in the trailer:

```
<type>(<scope>): <short description>

Refs: <ITEM_ID>
```

**Never modify files under `PLANS_DIR` on the feature branch.** Status changes go through `RECORD_STATUS` only.

### L6. Final Cleanup

Before opening the PR, scan for:

- Dead code introduced or made stale by the implementation
- Obsolete comments or outdated references
- Unused imports

Commit cleanup separately.

### L7. Open the PR

```bash
git push origin HEAD
gh pr create --title "<concise title under 70 chars>" --body "$(cat <<'EOF'
## Summary
- <bullet points describing changes>

Plan: <ITEM_ID> (<ITEM_PATH>)

## Test plan
- [ ] <specific things to verify>

## Deferred Acceptance Criteria
- [ ] <AC from the item NOT addressed in this PR>

EOF
)"
```

Use `ITEM_ID` — the identifier of the item currently being implemented. **Never reference the parent plan as the PR's item.** If this is a phase, include only `Plan: <ITEM_ID> (...)`, never `Plan: <PARENT_ID>`.

**Deferred Acceptance Criteria section**: Include only if at least one AC from the item was not satisfied. If every AC is addressed, omit the entire heading. Presence of this section with non-empty bullets blocks the done transition in step L15.

After the PR exists, run `RECORD_STATUS` (commit summary `chore(plans): <ITEM_ID> opened PR #<N>`) setting `pr: "<PR_NUMBER>"` and `updated`, and appending an Activity entry headed `PR opened` with body `- PR: #<PR_NUMBER> <PR_URL>`.

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
5. **`PRIMARY_RUNTIME=unknown` and no project reviewer exists** → post a `Code Review Cannot Proceed` comment, add `needs-human-review` to the PR and the item, and skip to L16 with result `ESCALATED`. Do not guess and risk same-runtime self-review. Do not run the reviewer-output debug-capture flow below: no reviewer was invoked, so there is no attempt evidence to capture.

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

**Escalation debug capture (all reviewer invocation paths).** Run this flow exactly once when a selected reviewer path produces no usable output: the Claude probe fails, both permitted attempts for a Claude or project-configured review round are inconclusive, the Codex CLI attempts and Codex MCP method both fail, or the shared diff-acquisition procedure exhausts its fetch retries (`FAILURE_POINT=diff-fetch` — a `gh` failure that never charges a reviewer attempt, so the evidence must name `gh`, not the reviewer). Do not run it for selection rule 5 above because that path never invoked or probed a reviewer, so it has no attempt evidence to capture. The caller supplies `FAILURE_POINT`, `FAILURE_SUMMARY`, `REVIEWER_PATH`, `REVIEWER_MODEL`, the path-specific `Code Review Cannot Proceed` PR-comment body, and all completed-attempt evidence. Record `ESCALATION_ITEM_ID`, `ESCALATION_ITEM_PATH`, and `ESCALATION_ERRORS` for L16.

1. **Assemble the debug bundle before deleting temporary artifacts.** Build Markdown with these sections, modeled after a diagnostic incident report:

   - `# Reviewer failure debug capture`
   - `## Summary`: failure point and summary, `ITEM_ID`, PR number, repository, branch, primary runtime, reviewer path and exact model, skill name `implement-issue`, skill version from this file's front matter, and the [canonical SKILL.md URL](https://github.com/holos-run/skills/blob/main/plugins/simple-workflow/skills/implement-issue/SKILL.md).
   - `## Environment`: start and completion timestamps, execution connector or host when known, and relevant non-secret tool versions.
   - `## Escalation evidence`: why the output was unusable, the PR-comment posting status, and any method-selection or MCP qualification failures.
   - `## Attempt-by-attempt statuses`: one subsection per probe, CLI attempt, or MCP attempt. Record the exact command or remote invocation run; connector session ID (or `none`) and connector exit code; start and completion timestamps; whether the shell reached each status-recording line; `PROBE_STATUS` / `PROBE_PARSE`, `GH_DIFF_STATUS` / `CLAUDE_STATUS` / `EXTRACT_STATUS`, or `CODEX_STATUS` as applicable; stderr tails; result-envelope excerpts; and post-completion byte counts for the diff and every output, stderr, or envelope artifact. Use `not applicable` or `not produced` for fields a method does not have instead of omitting them.
   - `## Conclusion and confidence`: the most likely failure boundary, confidence level, and what remains unknown.

   Apply the same publication-safety rules required by the escalation PR comment below to the entire bundle: truncate every stderr tail to its last 20 lines, cap each result-envelope excerpt at approximately 2000 characters, and redact API keys, bearer tokens, `sk-` / `ghp_`-style secrets, URLs with embedded credentials, and other credential-shaped strings. Preserve exact commands only after applying those redactions. The full PR diff is evidence only by byte count; do not embed its contents in the plan file.

2. **Post the existing path-specific PR escalation comment.** Render the caller's `Code Review Cannot Proceed` body with the compact, redacted attempt evidence that is available as `ESCALATION_COMMENT`, append the agent-attribution footer per the "Agent Attribution" section, then post it with the shared bounded invocation:

   ```bash
   timeout --kill-after=10 60 gh pr comment "$PR_NUMBER" --body "$ESCALATION_COMMENT"
   COMMENT_STATUS=$?
   ```

   If it fails, append `COMMENT_STATUS` to `ESCALATION_ERRORS` and continue.

3. **Write the escalation plan item and mark the original item.** In one `RECORD_STATUS` (commit summary `chore(plans): escalate <ITEM_ID> reviewer failure on PR #<PR_NUMBER>`):

   - Allocate the next top-level identifier `ESCALATION_ITEM_ID` (highest `[0-9]{3,}-` prefix in `PLANS_DIR` plus one, zero-padded) and create `<PLANS_DIR>/<ESCALATION_ITEM_ID>-reviewer-failure-pr-<PR_NUMBER>.md` with front matter `title: "fix(implement-issue): code reviewer returned no output on PR #<PR_NUMBER> (<REPO>)"`, `status: todo`, `labels: ["needs-human-review", "escalation"]`, `related: ["<ITEM_ID>"]`, and a body containing: one short paragraph with the failure summary, reviewer path and model, PR/repository reference, skill name and front-matter version, the canonical SKILL.md URL above, and this exact ask: `Modify the implement-issue skill so this failure mode is prevented or diagnosable.` Then a `## Debug capture` section containing the redacted debug bundle from step 1.
   - Add the escalation item to `TODO.md`.
   - On `ITEM_PATH`: add `needs-human-review` to labels, add `ESCALATION_ITEM_ID` to `related`, and append an Activity entry headed `Reviewer escalation` linking the escalation item by path.

   Record `ESCALATION_ITEM_PATH`. If the status commit fails, append the failure to `ESCALATION_ERRORS` and continue.

4. **Degrade gracefully.** Treat every step above as a single best-effort attempt. Never retry indefinitely or let this flow prevent the PR-comment and label escalation. Apply `needs-human-review` to the PR (`gh pr edit $PR_NUMBER --add-label "needs-human-review"`), remove temporary reviewer artifacts, and skip to L16 with result `ESCALATED`; L16 reports any created escalation item and every best-effort failure.

**PR diff acquisition (all CLI reviewer paths).** Fetching the PR diff is a prerequisite of a review round, not part of the reviewer invocation. Its failures are `gh`/network failures and must never consume a reviewer attempt — the reviewer was never launched. Every review round acquires its diff with this procedure:

1. Resolve the PR head before the round: `HEAD_SHA=$(timeout --kill-after=10 30 gh pr view "$PR_NUMBER" --json headRefOid -q .headRefOid)`.
2. **Reuse a verified diff when safe.** If a diff fetched earlier in this skill invocation exists, is non-empty, and was recorded against this same `HEAD_SHA`, copy or reuse that file instead of refetching — the diff of an unchanged head is immutable, and reuse removes a network dependency from reviewer retries. If `HEAD_SHA` cannot be resolved, do not reuse; fetch fresh.
3. Otherwise fetch with `timeout --kill-after=10 60 gh pr diff "$PR_NUMBER"`, capturing stdout to the attempt's diff file and stderr to a sibling file. Transient transport failures (GraphQL `EOF`, connection reset, HTTP 5xx) are retried **here**, inside this step: up to 3 fetch attempts total, sleeping 15 seconds between attempts.
4. A fetch attempt succeeds when its exit status is 0 and the diff file is non-empty. On success, record `HEAD_SHA` alongside the file so later rounds and retries can reuse it under rule 2.
5. If all 3 fetch attempts fail, do not invoke the reviewer and do not charge a reviewer attempt. Run the shared **Escalation debug capture** flow once with `FAILURE_POINT=diff-fetch` and evidence naming `gh` as the failing component — per-attempt exit statuses, redacted stderr tails, and byte counts — never the reviewer.

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
# Acquire "$REVIEW_OUT.diff" per the shared "PR diff acquisition" procedure above
# (head-SHA reuse, bounded fetch, up to 3 transport retries). Acquisition failure
# escalates there with FAILURE_POINT=diff-fetch and never reaches — or charges —
# this reviewer invocation, so the diff here is always present and non-empty.
timeout 600 codex exec \
  --model "$CODEX_FRONTIER_MODEL" \
  --dangerously-bypass-approvals-and-sandbox \
  --skip-git-repo-check \
  --output-last-message "$REVIEW_OUT" \
  "<the review prompt above, with $PR_NUMBER and $REPO expanded; the diff is also supplied on stdin>" \
  < "$REVIEW_OUT.diff"
CODEX_STATUS=$?
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

The shared flow writes the escalation plan item, applies the PR label and item label, and skips directly to L16 with result `ESCALATED` even if the status commit fails.

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

Do not substitute another reviewer. The shared flow applies the labels and skips to L16 with result `ESCALATED`, even if writing the escalation item fails.

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

`mktemp -d` creates `REVIEW_DIR` with mode 700; every reviewer artifact (diff, envelopes, stderr) lives inside it — never in ad-hoc sibling paths — and the whole directory is removed with `rm -rf "$REVIEW_DIR"` when review concludes or escalates, because it holds the full PR diff and CLI diagnostics. That final removal is best-effort: some host command policies reject `rm` invocations outright, and a rejected cleanup must never stall or fail the review — leave the mode-700 directory in place and note it. No mid-review `rm` is ever needed: each review attempt stages its artifacts in a fresh per-attempt subdirectory (below), so a failed attempt can never surface a previous attempt's files and no attempt requires an `rm`-based reset. The `if` guard exits before any child path exists: when `mktemp` fails or returns empty, the shell call ends there — treat that nonzero exit as a probe failure and escalate. Never construct child paths from an unverified `REVIEW_DIR`, because an empty prefix turns them into root-level paths like `/pr.diff`. Run the probe with the host shell tool's deadline strictly above its 135 s worst case (120 s inner bound + 15 s kill grace) — e.g., 180000 ms for Claude Code's Bash tool, whose default deadline is shorter. The jq filter requires `.is_error == false` literally (a missing field must fail, so `| not` is not acceptable) and a non-empty string `result`.

The probe succeeds only when `PROBE_STATUS` and `PROBE_PARSE` are both 0. If `claude` or `jq` is missing, or the probe fails, the Claude reviewer is unavailable. Run the shared **Escalation debug capture** flow exactly once with `FAILURE_POINT=probe`, the probe's exact command, connector-session evidence, `PROBE_STATUS` / `PROBE_PARSE`, byte counts, stderr tails, and envelope evidence. Use the Claude `Code Review Cannot Proceed` body below as the path-specific PR-comment body. The shared flow applies `needs-human-review` to the PR and the item, then skips to L16 with result `ESCALATED`, even if writing the escalation item fails. Do not invoke Codex or inherit the current session model, and do not spend review rounds discovering a statically broken CLI.

If the probe succeeds, each round in L9–L11 runs Claude Code as **one foreground, bounded call**, under the same four rules as the Codex CLI reviewer: never background it, never poll the process table for completion, bound every network-dependent command with `timeout --kill-after=<grace> <seconds>`, and wait through connector yields on the exact same session until the connector reports an exit code. A foreground tool session may yield control before process exit; that yield is still the first attempt and waiting on it with the connector-native stdin/wait operation is not backgrounding, polling, or another attempt. If an outer orchestration cell yields, resume its `cell_id` with the outer wait operation so it continues waiting on the existing shell session. The inner `timeout --kill-after=10 60` diff fetch and the `timeout --kill-after=15 "$CLAUDE_REVIEW_TIMEOUT"` Claude call (default 490 seconds) remain authoritative: cumulative connector waits must allow enough time to observe their exit status, including kill grace, rather than imposing a shorter host deadline. The Claude bound is configurable via `CLAUDE_REVIEW_TIMEOUT`; a host that permits a longer bound may raise it only when the combined inner worst case — raised bound plus kill grace — remains strictly below the host deadline.

Claude Code `--print` accepts a prompt argument and appends piped stdin to that prompt; supply the diff on stdin. Use `--output-format json`, not `text`: the JSON envelope makes success machine-checkable (`type`, `is_error`, non-empty `result`) where an empty text stream is ambiguous, and capturing stderr separately means every failure leaves diagnosable evidence.

**Diff-only review.** The complete diff arrives on stdin, so the reviewer needs no repository exploration — unbounded agentic exploration of the repository is the leading suspect when a review consumes its entire time budget without producing a result envelope. Before the first round, check `claude --help` for a tool-restriction capability: when the installed CLI supports `--tools`, add `--tools ""` to the review command to disable tools entirely; otherwise, when it supports a disallow list, disallow all write and execution tools and permit at most read-only repository access; when neither is supported, run unrestricted. Record which restriction was applied so escalation evidence can report it.

Each attempt stages its artifacts in a fresh per-attempt directory — never an `rm`-based reset of shared paths, which some host command policies reject:

```bash
ATTEMPT_DIR=$(mktemp -d "$REVIEW_DIR/attempt.XXXXXX")
if [ -z "$ATTEMPT_DIR" ] || [ ! -d "$ATTEMPT_DIR" ]; then
  echo "mktemp -d failed: cannot stage reviewer attempt artifacts" >&2
  exit 1
fi
# Acquire "$ATTEMPT_DIR/pr.diff" per the shared "PR diff acquisition" procedure
# (head-SHA reuse, bounded fetch, up to 3 transport retries; record GH_DIFF_STATUS
# and stderr in "$ATTEMPT_DIR/gh-diff.err"). Acquisition failure escalates there
# with FAILURE_POINT=diff-fetch and never charges a reviewer attempt, so reaching
# the claude invocation means the diff is present and non-empty.
CLAUDE_REVIEW_TIMEOUT=${CLAUDE_REVIEW_TIMEOUT:-490}
timeout --kill-after=15 "$CLAUDE_REVIEW_TIMEOUT" claude --print \
  --model "$CLAUDE_REVIEW_MODEL" \
  --output-format json \
  <tool-restriction flags per "Diff-only review" above, when supported> \
  "<the review prompt above, with $PR_NUMBER and $REPO expanded; the complete diff is supplied on stdin>" \
  < "$ATTEMPT_DIR/pr.diff" > "$ATTEMPT_DIR/review.json" 2> "$ATTEMPT_DIR/review.err"
CLAUDE_STATUS=$?
jq -er 'select((.type == "result") and (.is_error == false))
        | .result | select(type == "string")' \
  "$ATTEMPT_DIR/review.json" > "$ATTEMPT_DIR/review.txt" 2> "$ATTEMPT_DIR/review-jq.err"
EXTRACT_STATUS=$?
VERDICT=$(awk 'NF {line=$0} END {print line}' "$ATTEMPT_DIR/review.txt" \
  | grep -E '^[[:space:]*_]*Verdict:[[:space:]*_]*(APPROVE|REQUEST_CHANGES)[[:space:]*_]*$' \
  | grep -oE 'APPROVE|REQUEST_CHANGES')
printf 'GH_DIFF_STATUS=%s\nCLAUDE_STATUS=%s\nEXTRACT_STATUS=%s\nVERDICT=%s\n' \
  "$GH_DIFF_STATUS" "$CLAUDE_STATUS" "$EXTRACT_STATUS" "$VERDICT" \
  > "$ATTEMPT_DIR/status.env"
```

The trailing `printf` persists every phase's status independently in `$ATTEMPT_DIR/status.env`. This matters because a wrapper that ends by printing or recording its captured statuses exits 0 even when inner phases failed — the wrapper's (or connector's) exit code is **never** evidence that the review succeeded. Diagnosis reads the persisted per-phase statuses, artifact byte counts, and `VERDICT`, not the wrapper exit alone.

**Apply the completion gate before inspecting any status variable or artifact.** The host connector must report an exit code for the foreground Bash call. If it instead reports a `session_id` with no exit code, continue waiting on that exact session with empty connector-native stdin/wait calls. If the outer orchestration yields a `cell_id`, resume that cell with its own wait operation. Until an exit code is observed, unset `GH_DIFF_STATUS` / `CLAUDE_STATUS` / `EXTRACT_STATUS`, missing status artifacts, and empty `review.json` / `review.err` redirect targets are expected signs that the process is still running — never evidence of an inconclusive attempt. Do not read or parse the artifacts, restart the command, or charge a retry while the session is live.

After the completion gate passes, interpret the result deterministically — never re-run merely to "check if it finished". A round is **conclusive** only when ALL of the following hold; a partial `gh` diff, a truncated JSON stream rescued by `jq`, or review text with no verdict must never pass as a completed review:

- `GH_DIFF_STATUS` is 0 and `$ATTEMPT_DIR/pr.diff` is non-empty (guaranteed by the shared diff-acquisition procedure — a diff failure is a `gh` failure handled and escalated there, so it never reaches this interpretation and never charges a reviewer attempt);
- `CLAUDE_STATUS` is 0;
- `EXTRACT_STATUS` is 0 and `$ATTEMPT_DIR/review.txt` is non-empty;
- `VERDICT` is non-empty. The verdict comes only from the review's **final nonblank line**, which the prompt demands be a dedicated `Verdict:` line (the extraction tolerates surrounding Markdown emphasis). Prose that merely mentions a verdict token, an echoed rubric, a refusal with an earlier standalone verdict, or a token embedded in another word (`DISAPPROVE`) can never pass.

**Conclusive:** `$ATTEMPT_DIR/review.txt` is the round's review with verdict `$VERDICT`. Parse the per-severity findings and post it to the PR. **Anything else after observed connector completion — the round is inconclusive.** Before retrying, record these fields separately so a connector yield can never be mistaken for an empty Claude result:

- connector session ID (or `none` when the call never yielded) and connector exit code;
- attempt start and completion timestamps;
- whether the shell reached the `GH_DIFF_STATUS`, `CLAUDE_STATUS`, and `EXTRACT_STATUS` recording lines;
- `GH_DIFF_STATUS`, `CLAUDE_STATUS` (124 means `timeout` expired and `SIGTERM` ended the call; 137 means the call ignored `SIGTERM` and `--kill-after` sent `SIGKILL`), and `EXTRACT_STATUS`;
- byte counts for `pr.diff`, `gh-diff.err`, `review.json`, `review.err`, `review.txt`, and `review-jq.err`, measured only after connector completion;
- the tails of `review.err`, `gh-diff.err`, and `review-jq.err`, plus the envelope's `subtype`/`is_error` from `review.json` if it parsed.

**Only attempts that actually invoked `claude` count toward the two permitted reviewer attempts.** Diff-acquisition failures are retried and, if persistent, escalated inside the shared procedure — a failed prerequisite must never consume the sole reviewer retry when Claude was never launched.

Then rerun the foreground command once in a fresh attempt directory. When the failed attempt timed out (`CLAUDE_STATUS` 124 or 137), harden the rerun for observability and success odds:

- Reuse the already-verified diff under the shared procedure's head-SHA rule instead of depending on another network fetch.
- Add CLI diagnostics when the installed version supports them — for example `--debug-file "$ATTEMPT_DIR/claude-debug.log"`, or a stream-JSON output mode with partial messages — so a second timeout leaves evidence instead of a 0-byte envelope; redact any diagnostic excerpt before it appears in a PR comment or plan file, and keep the strict final-verdict gate unchanged.
- Optionally raise `CLAUDE_REVIEW_TIMEOUT` (for example, double it), but only when the host execution deadline still strictly covers the raised bound plus kill grace.

If the rerun is also inconclusive, do not invoke Codex or inherit the current session model. Run the shared **Escalation debug capture** flow exactly once with both attempts' complete evidence and the Claude `Code Review Cannot Proceed` body below as the path-specific PR-comment body. The shared flow applies the labels and skips to L16 with result `ESCALATED`, even if writing the escalation item fails.

**Escalation comment body (probe failure and inconclusive rounds alike).** The body must carry the captured evidence — never only a prose conclusion like "completed without producing review output", which leaves the failure undiagnosable. Before posting, truncate each stderr tail to its last 20 lines, cap the result-envelope excerpt at ~2000 characters, and redact anything credential-shaped (API keys, bearer tokens, `sk-`/`ghp_`-style strings, URLs with embedded credentials) — CLI diagnostics can leak local paths and account details, and a PR comment is public within the repo:

```markdown
## Code Review Cannot Proceed

This Codex implementation requires review by the latest Claude Opus model (`--model $CLAUDE_REVIEW_MODEL`). The reviewer cannot silently downgrade to Codex because that would be same-family self-review. Marking for human review.

Evidence:
- Failure point: <probe | diff-fetch | review round N attempt M>
- Diff acquisition: <reused verified diff for unchanged head SHA | per-fetch-attempt statuses, redacted>
- Tool restriction: <flags applied per "Diff-only review", or "none supported">
- Statuses: <GH_DIFF_STATUS / CLAUDE_STATUS / EXTRACT_STATUS, or PROBE_STATUS / PROBE_PARSE; note 124 = timed out at <bound>s, 137 = SIGKILL after ignoring SIGTERM>
- stderr tails (last 20 lines each, redacted): <from the captured stderr files, or "empty">
- Result envelope (truncated, redacted): <subtype / is_error / result excerpt, or "unparseable">
```

The shared escalation flow posts this body with its bounded invocation in step 2. If `COMMENT_STATUS` is nonzero, do not stall: record it in `ESCALATION_ERRORS`, still attempt the escalation plan item, still remove `$REVIEW_DIR`, still apply the `needs-human-review` labels, and still finish with result `ESCALATED`.

For automatic Codex-runtime review, the PR comment header must identify `opus (latest Claude Opus)`, not `claude session model`. This makes accidental regression to same-family review visible without pinning a model version.

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
- **If style-only findings remain (no CRITICAL or IMPORTANT):** Proceed to step L12. Create a follow-up item for style findings after merge (step L14).
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

3. Add `needs-human-review` to the item: `RECORD_STATUS` (commit summary `chore(plans): <ITEM_ID> needs human review`) adding the label, updating the TODO line, and appending an Activity entry headed `Needs human review` listing the unresolved findings and the PR number.

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

If CI still fails after one fix attempt, escalate (add `needs-human-review` per step L11, with an Activity entry describing the CI failure) and skip to step L16 with result ESCALATED.

### L13. Merge

Before merging, handle remaining style-only findings from round 2 by creating a follow-up item (step L14).

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

After the merge, `git fetch origin` so the following status commits build on the merged `main`.

### L14. Follow-Up Item for Style Findings

If style-only findings remain after round 2, create a follow-up item in the same status commit as the L15 close (or its own status commit when L15 does not close the item).

- If this item has a parent (`PARENT_ID` set): create it as a new sibling phase `<PARENT_DIR>/<next PP>-address-review-findings-pr-<PR_NUMBER>.md` with `parent: "<PARENT_ID>"` and `id: "<PARENT_ID>-<PP>"`, where `PP` is one more than the highest existing phase number in `PARENT_DIR`. Add it to the parent's `## Implementation Plan` checklist.
- Otherwise: create it as a new top-level item `<PLANS_DIR>/<next NNN>-address-review-findings-pr-<PR_NUMBER>.md`.

Front matter: `title: "fix: address review findings from PR #<PR_NUMBER>"`, `status: todo`, `labels: []`, `related: ["<ITEM_ID>"]`. Body:

```markdown
## Context

PR #<PR_NUMBER> (item <ITEM_ID>) was merged with style-only review findings remaining.

## Findings

<paste remaining style findings>

## Activity
```

Add its TODO line. Record the follow-up identifier as `FOLLOW_UP_ID`.

### L15. AC-Gate and Close Item

Before marking done, check whether the PR lists deferred acceptance criteria:

```bash
PR_BODY=$(gh pr view $PR_NUMBER --json body --jq .body)
```

Parse for a `## Deferred Acceptance Criteria` heading with non-empty bullets.

**If deferred ACs exist:**

- Do NOT set status to done
- `RECORD_STATUS` (commit summary `chore(plans): <ITEM_ID> merged with deferred acceptance criteria`): add `needs-human-review` to labels, keep `status: in-progress`, append an Activity entry headed `Merged with deferred acceptance criteria` listing the deferred items and the PR number, update the TODO line
- Record result as MERGED_WITH_DEFERRED_ACS

**If no deferred ACs:**

`RECORD_STATUS` (commit summary `chore(plans): <ITEM_ID> done`) with:

- `status: done`
- `labels`: existing labels minus `implementing`
- `updated`: today
- Tick every acceptance-criteria checkbox in the item that the PR satisfied
- If `PARENT_ID` is set, tick this item's line in the parent's `## Implementation Plan` checklist
- Tick the TODO line and show `done`; for a top-level item, move its block to `## Done`

Include the L14 follow-up item in this same status commit when one is needed.

### L16. Post Summary

Calculate elapsed time and post a summary:

```bash
ISSUE_END_TIME=$(date +%s)
ELAPSED=$((ISSUE_END_TIME - ISSUE_START_TIME))
MINUTES=$((ELAPSED / 60))
SECONDS=$((ELAPSED % 60))
```

Append an Activity entry on `ITEM_PATH` headed `Implementation complete` (commit summary `chore(plans): <ITEM_ID> summary`). This may be folded into the L15 status commit when both happen back to back:

```
## Implementation Complete

- PR: #<PR_NUMBER>
- Result: <MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED>
- Review rounds: <count>
- Wall clock time: <MINUTES>m <SECONDS>s
- Follow-up: <FOLLOW_UP_ID> (if any)
- Reviewer escalation: <ESCALATION_ITEM_ID> <ESCALATION_ITEM_PATH> (if reviewer-output escalation created one)
- Escalation errors: <ESCALATION_ERRORS> (if any best-effort step failed)
- Pending status PRs: <URLs> (if branch protection forced the RECORD_STATUS fallback)
```

Return the same summary as the worker's result when running as a dispatched sub-agent.

---

## Parent Mode

Orchestrator for a primary plan (or any item) with child phase files. The orchestrator runs in the top-level harness the user invoked and dispatches each child to a worker sub-agent of that **same harness** (inheriting the session model unless `--model` overrides it), per the "Arguments and Model Selection" resolution rules. It never hands implementation to a different harness.

### P1. Settle the Orchestrator Branch

The orchestrator never commits code and never commits plan files to its own branch — all status goes through `RECORD_STATUS`, and all code goes to phase feature branches. It only needs a stable branch to return to between dispatches, because workers share this worktree and switch it to their feature branches. Run this step at the top of **every** orchestrator invocation, including re-invocations and resumptions.

Define `IDENT_LC` as `ITEM_ID` lowercased. Classify the **current** branch:

- **`main`** (the skill was invoked from the primary checkout): stay here. `PRIMARY_BRANCH=main`. Keep it current with `git pull --ff-only origin main` after each status commit and merge.
- **A phase branch of this plan** — the name contains `/<IDENT_LC>-<two digits>-` (a crashed worker left this worktree on its feature branch). Switch to the plan branch: check out `plan/<IDENT_LC>-<slug>` if it exists locally, otherwise create it with `git checkout -b plan/<IDENT_LC>-<slug> origin/main`. `PRIMARY_BRANCH` is that plan branch.
- **Any other branch** (a harness session branch such as `claude/<slug>`, or an existing plan branch): stay here. `PRIMARY_BRANCH` is the current branch. Rebase it onto `origin/main` so the worktree reflects current plan files:

  ```bash
  git fetch origin
  git rebase origin/main
  ```

  If the rebase fails due to conflicts: capture `CONFLICTS=$(git diff --name-only --diff-filter=U)` **before** `git rebase --abort`, then abort. Add `needs-human-review` to the item with an Activity entry headed `Orchestrator branch conflicts` listing `$CONFLICTS`, and stop the skill.

Record `PRIMARY_BRANCH`.

### P2. Start Wall Clock Timer

```bash
PLAN_START_TIME=$(date +%s)
```

### P3. List Children

```bash
git fetch origin
git ls-tree --name-only origin/main "<ITEM_DIR>/" | grep '\.md$' | sort
```

For each child file, read its front matter from `origin/main` and record:

- `id` (e.g., `007-02`)
- `path`
- `title`
- `status` (`todo`, `in-progress`, `done`, `canceled`)

Children are processed in file-name order (phase number order). Skip any child whose status is `done` or `canceled`.

If no open children remain, post an Activity entry headed `All phases complete` on the item and continue to P10.

### P4. Review Existing Progress

Inspect the state left by prior orchestrator sessions. The orchestrator **must not** re-implement what already landed on `main`.

```bash
git log --oneline origin/main..HEAD
git diff --stat origin/main..HEAD
gh pr list --state all --search "Plan: <ITEM_ID>-" --json number,title,state,headRefName
```

Classify each open child from P3 as **already-implemented** when either of the following holds:

- Its status on `origin/main` is `done` or `canceled` (P3 already filters these out, but list them in the state entry for visibility).
- A merged PR exists whose body references `Plan: <child id>` — a prior session merged the work but crashed before the L15 status commit. In that case, run the L15 close for it now (status `done`, TODO tick, parent checklist tick) so the plan files match reality, then skip it.

Build a `SKIP_SUB_ISSUES` set containing every already-implemented child identifier. The dispatch step (P6) **must consult `SKIP_SUB_ISSUES` and skip any child in it** — do not dispatch a worker for those identifiers and do not delete their existing branch state.

Post an Activity entry on the item headed `Orchestration state`:

```
- Branch: <PRIMARY_BRANCH>
- Already-implemented phases (skipped): <list of identifiers, or "none">
- Remaining phases to dispatch: <list of identifiers>
```

### P5. Transition Labels

Replace `planning` with `implementing` on the item and set `status: in-progress`. Fold this into the same `RECORD_STATUS` as the P4 Activity entry (commit summary `chore(plans): start plan <ITEM_ID>`). Update the TODO line.

### P6. Dispatch Phases

Process phases **sequentially** (each phase depends on the previous one). Each open child is dispatched to a worker sub-agent of the orchestrator's own harness — the model comes from `MODEL_OVERRIDE` or the session default, never from labels. The dispatched worker runs the full leaf lifecycle for that one phase and returns a short result summary.

**Honor the `SKIP_SUB_ISSUES` set built in P4**: if a child's identifier appears in `SKIP_SUB_ISSUES`, skip it entirely — do not run pre-dispatch cleanup, do not dispatch a worker, and do not delete its branch or PR. Move on to the next child.

#### Pre-Dispatch: Detect and Discard Partial Work

Before dispatching each phase, check for leftover state from any prior attempt. Partial work is present when a local branch, remote branch, or open PR exists for the phase.

Derive the branch prefix from the child identifier (e.g., `007-02` → prefix `feat/007-02-`):

```bash
SUB_PREFIX="feat/<SUB_ID>-"
SUB_BRANCH=$(git branch --list "${SUB_PREFIX}*" | head -1 | xargs)
SUB_OPEN_PR=$(gh pr list --state open \
  --json number,headRefName \
  --jq ".[] | select(.headRefName | startswith(\"${SUB_PREFIX}\")) | .number" \
  | head -1)
```

If partial work exists (`SUB_BRANCH` or `SUB_OPEN_PR` is non-empty):

1. Close the open PR without merging (if one exists):
   ```bash
   gh pr close "$SUB_OPEN_PR" --comment "$(cat <<'EOF'
Discarding partial work — restarting implementation from scratch.

---
🤖 <AGENT_SIGNATURE> — automated comment posted by an AI agent on behalf of this repository's operator. Replies here reach an agent, not the human directly.
EOF
)" 2>/dev/null || true
   ```
2. Delete the remote branch:
   ```bash
   git push origin --delete "$SUB_BRANCH" 2>/dev/null || true
   ```
3. If `$SUB_BRANCH` is the current branch (a previous orchestrator session crashed while it was checked out), switch back to `PRIMARY_BRANCH` first — `git branch -D` will refuse to delete a checked-out branch:
   ```bash
   if [ "$(git rev-parse --abbrev-ref HEAD)" = "$SUB_BRANCH" ]; then
     git checkout "$PRIMARY_BRANCH"
   fi
   ```
4. Delete the local branch:
   ```bash
   git branch -D "$SUB_BRANCH" 2>/dev/null || true
   ```
5. Refresh `origin/main`. **Do not run `git checkout main`** unless `PRIMARY_BRANCH` is `main`.
   ```bash
   git fetch origin
   ```
6. Reset the child: `RECORD_STATUS` (commit summary `chore(plans): reset <SUB_ID> for a clean attempt`) with `status: todo`, `implementing` removed from labels, `branch: ""`, `pr: ""`, an Activity entry headed `Discarding partial work` with body `Discarding partial work from a previous attempt. Starting a clean implementation.`, and the TODO line updated.

**Dispatch per phase** — always a sub-agent of the orchestrator's own harness, with the model from the "Arguments and Model Selection" resolution (`MODEL_OVERRIDE` if set, else the session default). Labels play no part. Never dispatch implementation to a different harness than the one the user invoked.

In every worker template below (both harnesses, initial dispatch and retries), the skill invocation inside the prompt must propagate the overrides per "Arguments and Model Selection": append ` --model <MODEL_OVERRIDE>` when `MODEL_OVERRIDE` is set and ` --reviewer <REVIEWER_OVERRIDE>` when `REVIEWER_OVERRIDE` is set. CLI-level flags on `codex exec` do not reach the child skill's argument parsing — only flags written into the `/simple-workflow:implement-issue` invocation text do.

If `PRIMARY_RUNTIME=claude`, spawn a Claude Code sub-agent:

```
Agent(
  description: "Implement <SUB_ID>",
  model: "<RESOLVED_MODEL — omit entirely for the session default>",
  prompt: "Invoke /simple-workflow:implement-issue <SUB_ID><propagated flags> to implement
  this phase end-to-end. The skill handles branching, implementation, code review, CI, merge,
  and plan-file status updates. Run to completion. Return a short summary: result (MERGED |
  MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED), PR number, and any follow-up item identifier."
)
```

If `PRIMARY_RUNTIME=codex`, spawn the worker as a `codex exec` subprocess — still the Codex harness. Resolve `WORKER_MODEL` as `MODEL_OVERRIDE` when set; otherwise, if the orchestrator knows the model its own session is running, use that; otherwise leave `WORKER_MODEL` unset and omit `--model` so the worker uses the Codex CLI's configured default (a fresh `codex exec` reads configuration — it cannot see a model selected interactively in the invoking session, so propagate the session model explicitly whenever it is known). Run it under the same foreground/completion-gate rules as every `codex exec` call in this skill:

```bash
set --
[ -n "${WORKER_MODEL:-}" ] && set -- --model "$WORKER_MODEL"
codex exec "$@" --dangerously-bypass-approvals-and-sandbox \
  "Invoke /simple-workflow:implement-issue <SUB_ID><propagated flags> to implement this
phase end-to-end. The skill handles branching, implementation, code review, CI, merge, and
plan-file status updates. Run to completion. Return a short summary: result (MERGED |
MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED), PR number, and any follow-up item identifier."
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
5. Re-dispatch the phase in the **same harness** with its original model resolution (`MODEL_OVERRIDE` if set, else session default). Never switch harnesses to dodge a usage limit — that would move implementation out of the harness the user invoked.

Never prompt the user for guidance on usage limits — resolve autonomously.

#### Orchestrator Constraint: Never Take Over Implementation Work

**The orchestrator must never implement phase work directly.** It does not write code, create commits, create branches, or perform any leaf-mode steps itself — not even to "help" a stuck worker finish. If a worker fails or gets stuck, the orchestrator's only permitted response is to clean up and dispatch a replacement worker using the same routing rules. The one exception is the P4 close-out of a phase whose PR already merged, which is a plan-file status commit, not implementation.

#### Detecting a Stuck Worker

A worker has **failed to complete** if it returns without a valid result summary (MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED) — for example, it got stuck mid-implementation and did not finish.

Use a retry loop with up to **3 total attempts** per phase:

1. If the worker's returned output does not contain a valid result summary, it got stuck.
2. Note where it got stuck and why (e.g. "wrote files but did not commit", "opened PR but did not wait for CI").
3. Return this worktree to `PRIMARY_BRANCH` if the worker left it elsewhere, and refresh `origin/main`:
   ```bash
   git checkout "$PRIMARY_BRANCH"
   git fetch origin
   ```
4. Dispatch a replacement worker in the same harness with the same model resolution (`MODEL_OVERRIDE` if set, else session default), using the same phase identifier plus a warning. Do not describe implementation steps — just provide context and let the skill decide what to do.

   If `PRIMARY_RUNTIME=claude` (omit `model` for the session default, as always):
   ```
   Agent(
     description: "Implement <SUB_ID> (retry <N>)",
     model: "<RESOLVED_MODEL — omit entirely for the session default>",
     prompt: "Invoke /simple-workflow:implement-issue <SUB_ID><propagated flags>.

     Warning: A previous attempt did not complete. Point: <e.g. 'wrote files but did not commit'>. Reason: <e.g. 'worker returned without a result summary'>.

     Invoke the skill and let it run to completion. Return a short result summary: result (MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED), PR number, and any follow-up item identifier."
   )
   ```

   If `PRIMARY_RUNTIME=codex` (resolve `WORKER_MODEL` exactly as in the initial P6 dispatch, using the same portable `set --` form for the optional flag):
   ```bash
   set --
   [ -n "${WORKER_MODEL:-}" ] && set -- --model "$WORKER_MODEL"
   codex exec "$@" --dangerously-bypass-approvals-and-sandbox \
     "Invoke /simple-workflow:implement-issue <SUB_ID><propagated flags>.

Warning: A previous attempt did not complete. Point: <e.g. 'wrote files but did not commit'>.
Reason: <e.g. 'worker returned without a result summary'>.

Invoke the skill and let it run to completion. Return a short result summary: result (MERGED |
MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED), PR number, and any follow-up item identifier."
   ```
5. Repeat until a valid result summary is returned or 3 attempts are exhausted.

**If all 3 attempts fail to return a valid result summary:**

a. In one `RECORD_STATUS` (commit summary `chore(plans): <SUB_ID> needs human review after 3 attempts`): add `needs-human-review` to the phase and to the parent item, update both TODO lines, and append an Activity entry on the parent headed `Phase abandoned`:
   ```
   Phase <SUB_ID> could not be completed after 3 attempts — each attempt got stuck before finishing. Marked for human review.
   ```

b. **Abort** — stop the skill immediately. Do not process further phases.

After each worker completes, detect the result by reading the phase's front matter from `origin/main` (cross-check against the worker's returned summary):

| Phase status | Labels | Result |
|--------------|--------|--------|
| `done` | — | MERGED |
| `in-progress` | has `needs-human-review` | Check Activity for "deferred acceptance criteria" → MERGED_WITH_DEFERRED_ACS; otherwise → ESCALATED |
| Any other state | — | FAILED |

Record per-phase timing and results.

After each phase, return this worktree to `PRIMARY_BRANCH` if the worker left it on the feature branch, and refresh `origin/main`. The next worker will branch directly from `origin/main` in its L2 step, so no local `main` is ever required.

```bash
git checkout "$PRIMARY_BRANCH"
git fetch origin
[ "$PRIMARY_BRANCH" = "main" ] && git pull --ff-only origin main
```

### P7. Sweep for Follow-Up Items

After all original children are processed, re-list `<ITEM_DIR>/*.md` on `origin/main`.

Compare against the original list. Any new open child is a follow-up created during review (L14).

Process follow-ups with the **full P6 pre-dispatch sequence** — including the *Pre-Dispatch: Detect and Discard Partial Work* step — then dispatch one worker per follow-up using the same-harness dispatch rules from P6.

### P8. Nested Parent Items

If a phase or follow-up is itself a parent (its own `<path minus .md>/` directory holds child files), the dispatched worker invoked in P6 will simply re-enter this skill in Parent Mode and orchestrate its own children with the same rules — including the P6 pre-dispatch partial-work cleanup for each grand-child. Because P6 workers are sub-agents of the same harness, every nested orchestrator also runs in the harness the user originally invoked. Nested orchestration composes naturally — no special handling is needed for depth.

Nested parent orchestrators follow the same resolution: `MODEL_OVERRIDE` (propagated via `--model`) first, otherwise inherit the session-configured model.

### P9. Post Summary

Calculate total elapsed time:

```bash
PLAN_END_TIME=$(date +%s)
PLAN_ELAPSED=$((PLAN_END_TIME - PLAN_START_TIME))
PLAN_MINUTES=$((PLAN_ELAPSED / 60))
PLAN_SECONDS=$((PLAN_ELAPSED % 60))
```

Append an Activity entry on the item headed `Plan execution complete` (fold it into the P10 status commit):

```
Total wall clock time: <PLAN_MINUTES>m <PLAN_SECONDS>s

### Phases

- <SUB_ID> <title>: MERGED | MERGED_WITH_DEFERRED_ACS | ESCALATED | FAILED
  - PR: #<PR_NUMBER>
  - Wall clock time: <minutes>m <seconds>s
  - Follow-up: <FOLLOW_UP_ID> (if any)
[...repeat for each phase]

### Follow-Up Items

- <FOLLOW_UP_ID> <title>: MERGED | ESCALATED | FAILED
  - PR: #<PR_NUMBER>
  - Wall clock time: <minutes>m <seconds>s
[...or "No follow-up items were created."]
```

If any phase is ESCALATED or MERGED_WITH_DEFERRED_ACS, add:

```
**Action required**: Some phases need human attention before this plan can close.
```

### P10. Close Parent

Re-read every child's status from `origin/main`.

**If every child is `done` or `canceled`:**

`RECORD_STATUS` (commit summary `chore(plans): plan <ITEM_ID> done`) with:

- `status: done`
- `labels`: existing minus `implementing`
- `updated`: today
- Tick every line in the item's `## Implementation Plan` checklist and any satisfied acceptance criteria
- The P9 Activity entry
- Tick the TODO line, show `done`, and move the plan's block to `## Done`

**Otherwise** (children remain `in-progress` due to deferred ACs or escalation):

Leave the item `in-progress`. `RECORD_STATUS` adding `needs-human-review` alongside `implementing` so it surfaces in the TODO list, with the P9 Activity entry.

Report the final summary to the user, including the URL of any status PR left open by the `RECORD_STATUS` fallback.

---

## Merge Authority

| Situation | Action |
|-----------|--------|
| Clean review, CI green, no deferred ACs | Merge, done |
| All findings fixed, clean re-review, CI green, no deferred ACs | Merge, done |
| Style-only findings after round 2, CI green, no deferred ACs | Merge, create follow-up, done |
| PR lists deferred ACs (any mergeable scenario) | Merge PR, but leave item in-progress + `needs-human-review` |
| CRITICAL/IMPORTANT unresolved after 2 fix rounds + final gate | Do NOT merge, `needs-human-review` |
| CI failures unresolved after 1 fix attempt | Do NOT merge, `needs-human-review` |
| Selected reviewer path produces no usable output after its permitted attempts | Do NOT merge; capture diagnostics, write an escalation plan item, and apply `needs-human-review` |

## Prerequisites

- **Plans folder**: a directory of markdown plan items created by `/simple-workflow:plan-issue` (or by hand following the front matter schema), on `main`
- **GitHub CLI**: `gh` authenticated with repo access
- **Git**: Clean working directory; push access to `main` for status commits (or PR merge rights for the fallback)
- **Same-harness implementation**: The orchestrator and every implementation worker run in the harness the user invoked. Only code review crosses harnesses. `--reviewer` is the only reviewer-routing override. `--model` affects implementation only, and only the model — never the harness.
- **Codex access**: Required for review when the user invoked the skill in Claude Code: the `codex` CLI and `jq` must resolve the latest frontier slug from `codex debug models`; `codex exec --model "$CODEX_FRONTIER_MODEL"` is preferred, and a model-selectable Codex MCP server may be used after an inconclusive CLI review. When the user invoked the skill in Codex, `codex exec` is also how Parent Mode spawns its same-harness workers.
- **Claude access**: The `claude` CLI must be on `PATH` and authenticated when a Codex implementation is automatically paired with the latest Claude Opus. L8 proves this end-to-end with a bounded `--print --output-format json` probe before the first round; `jq` is required to parse the result envelope. Reviewer-output failure escalates to human review with a redacted debug bundle written into a best-effort escalation plan item; it never falls back to the implementation model family.

## File Operations Cheat Sheet

| Action | How |
|--------|-----|
| Resolve plans folder | `CLAUDE.md` / `AGENTS.md` `Plans directory:` line or `## Plans` section; else `plans/` |
| Find item by identifier | File under `PLANS_DIR` whose front matter `id` matches (`<NNN>-*.md` or `<NNN>-*/<PP>-*.md`) |
| Read current status | `git fetch origin && git show origin/main:<path>` |
| List children | `git ls-tree --name-only origin/main "<path minus .md>/"` |
| Check blockers | Read each `blocked_by` identifier's status from `origin/main` |
| Set status / labels / branch / pr | Edit front matter inside the `RECORD_STATUS` worktree |
| Post activity | Append `### <timestamp> — <heading>` entry to `## Activity` with the footer |
| Create follow-up / escalation item | New file with front matter, in the `RECORD_STATUS` worktree |
| Record status | `RECORD_STATUS`: detached worktree at `origin/main`, commit, push `HEAD:main`, PR fallback |
| Update TODO | Edit the matching line in `<PLANS_DIR>/TODO.md` in the same status commit |
