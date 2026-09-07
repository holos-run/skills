---
name: plan-issue
description: v1.0.0 — Create a phased implementation plan as plain markdown files in the repository's plans folder, with no external tracker. Use this skill when the user provides a feature description, a plan item identifier (like 007), or a path to a markdown note and wants a plan broken into phases, each phase tracked as its own markdown file. Writes a NEW primary plan file plus one phase file per phase, updates the plans TODO list, and commits and pushes the result so status lives in git. Accepts an optional --model argument recorded in the plan's implementation instructions; otherwise implementation inherits the session-configured model. Every activity entry the skill writes ends with an agent-attribution footer naming the harness, model, and reasoning effort (for example `claude fable-5 high`) so colleagues know they are reading agent output, not the human account owner. Triggers on phrases like "plan this", "plan a feature", "break this into phases", "create a plan for", or any request to produce a phased plan tracked in the repository's plans folder.
version: 1.0.0
# Guardrail: whenever version changes, update the leading vX.Y.Z prefix in description in the same PR.
---

# Plan Issue

You are a principal engineer. Explore the codebase, produce an implementation plan broken into phases, and write it into a new primary plan file in the repository's plans folder. Create one phase file per phase under a directory named after the primary plan file. Record every phase in the plans TODO list, then commit and push the result so the plan is visible in git immediately.

There is no external issue tracker. Plain markdown files in the repository are the source of truth for planning and status.

Plan the work described by **{{SKILL_INPUT}}**.

## Arguments

`{{SKILL_INPUT}}` contains the plan input and optional flags:

```
<input> [--model <name>]
```

- `<input>` — one of:
  - A plan item identifier such as `007` or `007-02`. Resolves to the matching file in the plans folder.
  - A path to a markdown file, inside or outside the plans folder, describing the feature (an idea note, a design doc, a bug report).
  - Free text describing the feature to plan. Anything that is neither an identifier nor an existing file path is free text.
- `--model <name>` (also accepted as `model=<name>`) — pins implementation of this plan to a specific model. When present, record it as `MODEL_OVERRIDE` and write it into the primary plan file's Implementation Instructions (step 8) as a `--model <name>` flag on the suggested `implement-issue` invocations. Never write model names into labels — `implement-issue` does not read labels for routing, and implementation always runs in whatever harness invokes it; `--model` only selects a model within that harness. When absent, record nothing — implementation workers then inherit the model configured for the session. Inheriting the session model is the default and preferred behavior; only pin a model when the user explicitly asks for one.

Strip any flags from `{{SKILL_INPUT}}` before resolving the input in step 2.

## Plan File Conventions

This skill operates on markdown files in the repository. It uses no MCP tools and no network service other than `git push`.

- **Plans folder** (`PLANS_DIR`) — the directory that holds every plan file, resolved in step 1.
- **Item** = one markdown file in `PLANS_DIR` with YAML front matter (the unit of work). Equivalent to an issue in a tracker.
- **Primary plan** = a top-level item: `<PLANS_DIR>/<NNN>-<slug>.md`, where `NNN` is a zero-padded sequence number of at least three digits.
- **Phase** = a child item of a primary plan: `<PLANS_DIR>/<NNN>-<slug>/<PP>-<phase-slug>.md`, where `PP` is a zero-padded two-digit phase number. A child's directory is always the parent's file path minus `.md`.
- **Identifier** = `NNN` for a primary plan, `NNN-PP` for a phase, `NNN-PP-QQ` for a nested phase. Identifiers appear in branch names (`feat/<id>-<slug>`), commit trailers (`Refs: <id>`), and TODO lines.
- **TODO list** = `<PLANS_DIR>/TODO.md`, one line per item, maintained on every status change.
- **Activity** = the `## Activity` section at the end of each item. Entries there replace tracker comments. Every entry this skill writes must end with the agent-attribution footer defined below.
- **`main`** in this skill means the repository's default branch. Substitute `master` or another name when the repository uses one.

### Front matter schema

Every item file starts with this front matter. Keys not listed here are preserved but ignored.

```yaml
---
id: 007-02
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

Quote `id`, `parent`, and every identifier in lists so YAML never parses `007` as the integer `7`. Update `updated` whenever any other field changes.

### TODO.md format

`<PLANS_DIR>/TODO.md` is a nested task list. Primary plans are top-level bullets; phases are indented beneath their parent, in phase order. Done and canceled items are ticked. When a primary plan and all of its phases are done or canceled, move its whole block to the `## Done` section at the bottom.

```markdown
# TODO

Managed by simple-workflow. One line per plan item, ticked when done or canceled. The front matter of each linked file is authoritative; fix this list to match it if they disagree.

## Open

- [ ] **007** Add login — `in-progress` `implementing` — [007-add-login.md](007-add-login.md)
  - [x] **007-01** feat(db): add user schema — `done` — [007-add-login/01-user-schema.md](007-add-login/01-user-schema.md)
  - [ ] **007-02** feat(auth): add login handler — `in-progress` `implementing` — [007-add-login/02-login-handler.md](007-add-login/02-login-handler.md)
  - [ ] **007-03** chore: cleanup — `todo` — [007-add-login/03-cleanup.md](007-add-login/03-cleanup.md)

## Done

- [x] **006** Rotate signing keys — `done` — [006-rotate-signing-keys.md](006-rotate-signing-keys.md)
```

Line format: `- [ ] **<id>** <title> — \`<status>\`[ \`<label>\`...] — [<relative path>](<relative path>)`. Labels appear as additional backticked tokens after the status. Create the file with the header, an empty `## Open` section, and an empty `## Done` section when it does not exist.

## Recording Status Updates

Plan files live on `main`. Every change this skill makes to `PLANS_DIR` lands on `main` through a **status commit**: a commit built on the current `origin/main` tip in a temporary detached worktree, then pushed straight to `main`. This never checks out `main` in the current worktree (which may be a harness-provisioned worktree where `main` is unavailable) and never mixes plan-file edits with whatever branch the session happens to be on.

Procedure (`RECORD_STATUS`), run for each logical status change:

```bash
git fetch origin
STATUS_WT=$(mktemp -d)
if [ -z "$STATUS_WT" ] || [ ! -d "$STATUS_WT" ]; then
  echo "mktemp -d failed: cannot stage a status commit" >&2
  exit 1
fi
git worktree add --detach "$STATUS_WT" origin/main
# Edit files under "$STATUS_WT/$PLANS_DIR" with the file tools — create, update
# front matter, append Activity entries, update TODO.md. Never edit anything
# outside PLANS_DIR here.
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
   If the merge is refused (required reviews or checks), leave the PR open, report its URL, and continue — the status is recorded on the branch and pending on `main`.
4. **Anything else fails** — remove the worktree, report the error to the user, and stop. Never retry by committing to the current branch instead.

Honor the repository's commit signing configuration when committing (for example `git commit -S` where signed commits are required). Never pass `--no-gpg-sign`.

If the session's current branch is `main` itself (the skill was invoked from the primary checkout rather than a worktree), fast-forward it after every successful status commit so the working tree reflects the new plan files:

```bash
[ "$(git rev-parse --abbrev-ref HEAD)" = "main" ] && git pull --ff-only origin main
```

## Agent Attribution

Colleagues reading plan files must be able to tell at a glance that an activity entry came from an AI agent operating on behalf of a human, not from the human account owner. Every Activity entry this skill writes carries an attribution footer.

At the start of the skill, resolve these once and reuse them for the whole invocation:

- `AGENT_HARNESS` — the harness running this skill. Ask the native host identity first: Claude Code → `claude`, Codex → `codex`. Only if identity is unavailable, fall back to environment markers (`CODEX_THREAD_ID` alone → `codex`; `CLAUDE_CODE_ENTRYPOINT` or `CLAUDECODE` without `CODEX_THREAD_ID` → `claude`); otherwise use `unknown-harness`.
- `AGENT_MODEL` — the model slug this session is actually running (for example `fable-5`, `opus-4-6`, `solstice-alpha`), taken from the harness's native self-identity. Use `unknown-model` when it cannot be determined — never guess a plausible slug.
- `AGENT_EFFORT` — the reasoning-effort setting (for example `low`, `medium`, `high`, `xhigh`) when the harness exposes one for this session; omit the token entirely when unknown.

Compose the signature by joining the parts with single spaces:

```
AGENT_SIGNATURE = <AGENT_HARNESS> <AGENT_MODEL>[ <AGENT_EFFORT>]
```

Examples: `claude fable-5 high`, `codex solstice-alpha high`, `claude sonnet-4-5`.

**Footer requirement.** Every Activity entry this skill appends must end with a blank line and this line:

```
🤖 <AGENT_SIGNATURE> — automated entry written by an AI agent on behalf of this repository's operator.
```

Activity entry format (append to the end of the `## Activity` section; create the section if missing):

```markdown
### <ISO-8601 UTC timestamp> — <short heading>

<body>

🤖 <AGENT_SIGNATURE> — automated entry written by an AI agent on behalf of this repository's operator.
```

Item bodies (Problem, Acceptance Criteria, Implementation Plan, and so on) are exempt — only Activity entries carry the footer.

## Workflow

### 1. Resolve the Plans Folder

Determine `PLANS_DIR`, relative to the repository root, in this order:

1. **Indicated by the repository.** Read `CLAUDE.md` and `AGENTS.md` at the repository root. Use the first match of either:
   - A line of the form `Plans directory: <path>`, `Plans folder: <path>`, or `plans_dir: <path>` (case-insensitive, optional backticks around the path).
   - A `## Plans` section whose first fenced code block or first inline code span names a directory.
2. **Fallback.** Use `plans` at the repository root. Create it in the first status commit if it does not exist.

Record `PLANS_DIR`. Also record `NEXT_ID`: scan `PLANS_DIR` for entries whose names start with digits followed by a hyphen (`[0-9]{3,}-`), take the highest number, add one, and zero-pad to at least three digits. If no numbered entries exist, `NEXT_ID` is `001`.

### 2. Resolve the Input

Strip flags from `{{SKILL_INPUT}}`. Classify the remainder:

- **Identifier** (`^[0-9]{3,}(-[0-9]{2})*$`): locate the item file in `PLANS_DIR` whose front matter `id` matches. If none exists, tell the user and stop.
- **Existing file path**: read it.
- **Free text**: use it verbatim as the description.

Record:

- `ORIGINAL_PATH` — the file path when the input was an identifier or file; empty for free text
- `ORIGINAL_ID` — the front matter `id` when `ORIGINAL_PATH` is inside `PLANS_DIR` and has front matter; empty otherwise
- `ORIGINAL_TITLE` — the front matter `title`, the first `#` heading, or a title inferred from free text
- `ORIGINAL_BODY` — the markdown body (front matter removed), or the free text
- `EXISTING_LABELS` — the front matter `labels` when `ORIGINAL_ID` is set; otherwise empty

Read the item from `origin/main` after `git fetch origin` (`git show origin/main:<path>`) when it lives in `PLANS_DIR`, so the status seen is the one on `main`.

### 3. Mark the Original as Being Planned

Only when `ORIGINAL_ID` is set. This signals to operators that an agent owns the item.

Run `RECORD_STATUS` with these edits to `ORIGINAL_PATH`:

- `status: in-progress`
- `labels`: add `planning` to `EXISTING_LABELS`
- `updated`: today
- Append an Activity entry headed `Planning this item.` with the attribution footer
- Update the item's TODO line to show `in-progress` `planning`

Commit summary: `chore(plans): start planning <ORIGINAL_ID>`.

When `ORIGINAL_ID` is empty (free text or a file outside `PLANS_DIR`), there is nothing to mark; continue.

### 4. Read Project Conventions

Before exploring the codebase, read the project's configuration files to understand conventions:

1. Read `CLAUDE.md` if it exists — for project conventions, testing strategy, build commands
2. Read `AGENTS.md` if it exists — for architecture, package structure, code generation workflows
3. Read `CONTRIBUTING.md` if it exists — for commit message format, PR conventions

These files determine how to structure phases (e.g., whether the project uses proto-first development, what testing strategy to follow, what build toolchain to use). Do not hardcode assumptions — derive them from the project.

### 5. Clarify Acceptance Criteria

Review `ORIGINAL_BODY` and the user's prompt. If the acceptance criteria are ambiguous or incomplete, ask the user targeted clarifying questions before exploring the codebase. If the prompt is sufficiently clear, infer the acceptance criteria and proceed.

### 6. Explore the Codebase

Explore the relevant areas of the codebase to understand the architecture, existing patterns, and test conventions. Use the information from project configuration files (step 4) to guide exploration.

### 7. Draft the Implementation Plan

Break the implementation into sequential phases. Each phase should be a self-contained unit of work that leaves the codebase in a working state.

**Principles:**

- Phase ordering should follow the project's natural dependency graph (e.g., interface changes before implementation, backend before frontend).
- Each phase includes the testing approach appropriate for the project (read from CLAUDE.md / AGENTS.md).
- Every plan ends with a cleanup phase to remove stale code and documentation.
- Phases should be small enough for one agent to implement in a single session, and large enough to be a meaningful PR. Prefer 2–6 phases; merge trivial phases into their neighbors rather than creating one-line-change phases.
- **Each phase must be self-contained.** The implementing agent sees only the phase file — it has no access to this planning conversation. Write each file so a fresh agent can implement it without further context: exact file paths, function/type names discovered during exploration, the verification commands to run, and the specific patterns to follow.
- State inter-phase dependencies explicitly in each phase's `blocked_by` front matter and Dependencies section, by identifier. Phases are implemented sequentially in file order, so number the phases so every dependency comes before its dependents.
- Quote concrete acceptance criteria from the original description verbatim where possible, and distribute every criterion to exactly one phase — no criterion may be left unassigned.

### 8. Write the Plan Files

Allocate `PLAN_ID = NEXT_ID` and `PLAN_SLUG` from the title (lowercase, spaces to hyphens, special characters stripped, truncated to about 40 characters). The primary plan file is `<PLANS_DIR>/<PLAN_ID>-<PLAN_SLUG>.md` and the phase directory is `<PLANS_DIR>/<PLAN_ID>-<PLAN_SLUG>/`.

Run one `RECORD_STATUS` that performs all of the following edits together, so the plan appears on `main` atomically. Commit summary: `chore(plans): add plan <PLAN_ID> <title>`.

**a. Create the primary plan file.** When `MODEL_OVERRIDE` is set, append ` --model <MODEL_OVERRIDE>` to both `implement-issue` invocations in the Implementation Instructions section; when unset, leave them bare so implementation inherits the session model.

```markdown
---
id: "<PLAN_ID>"
title: "<ORIGINAL_TITLE or a refined version that reflects the plan>"
status: todo
labels: []
blocked_by: []
related: ["<ORIGINAL_ID>"]   # omit the entry when ORIGINAL_ID is empty
branch: ""
pr: ""
created: <today>
updated: <today>
---

# <title>

## Problem

<Describe the problem or motivation from the original description.>

## Acceptance Criteria

- [ ] <criterion 1>
- [ ] <criterion 2>
- [ ] ...

## Implementation Plan

This file is the primary implementation plan. Each phase below is tracked as its own file in `<PLAN_ID>-<PLAN_SLUG>/`. Implement phases in order — each phase must leave the codebase in a working state.

- [ ] <PLAN_ID>-01 — <phase 1 title>
- [ ] <PLAN_ID>-02 — <phase 2 title>
- [ ] <PLAN_ID>-03 — <phase 3 title>

## Implementation Instructions

To implement all phases, invoke `/simple-workflow:implement-issue <PLAN_ID>`.
To implement a single phase, invoke `/simple-workflow:implement-issue <PLAN_ID>-<PP>`.
Workers run as sub-agents of the harness that invokes `implement-issue` and inherit its session model unless a `--model <name>` argument overrides it.
Each phase branches from `origin/main` and merges to `main` via its own independent PR. Status updates are committed directly to the plan files on `main`.

## Original Description

Planned from <ORIGINAL_ID, ORIGINAL_PATH, or "the user's request">.

<ORIGINAL_BODY, verbatim, when non-empty>

## Activity

### <timestamp> — Plan created

Planned <phase count> phases. See the Implementation Plan section.

🤖 <AGENT_SIGNATURE> — automated entry written by an AI agent on behalf of this repository's operator.
```

**b. Create one phase file per phase** at `<PLANS_DIR>/<PLAN_ID>-<PLAN_SLUG>/<PP>-<phase-slug>.md`, numbering from `01`. Set `blocked_by` to the previous phase's identifier (empty for the first phase) so the sequential chain is explicit; add further identifiers when a phase depends on an earlier, non-adjacent phase. Never make a phase block its parent. When `MODEL_OVERRIDE` is set, append ` --model <MODEL_OVERRIDE>` to the invocation in the `## Parent` section so the pin survives direct single-phase implementation.

```markdown
---
id: "<PLAN_ID>-<PP>"
title: "<conventional-commit-prefix>(<scope>): <phase title>"
status: todo
labels: []
parent: "<PLAN_ID>"
blocked_by: ["<PLAN_ID>-<PP minus 1>"]
related: []
branch: ""
pr: ""
created: <today>
updated: <today>
---

# <title>

## Parent

Part of <PLAN_ID> (`<PLANS_DIR>/<PLAN_ID>-<PLAN_SLUG>.md`). To implement this phase directly, invoke `/simple-workflow:implement-issue <PLAN_ID>-<PP>`.

## Goal

<One-paragraph description of what this phase accomplishes and why.>

## Acceptance Criteria

- [ ] <specific, testable criterion>
- [ ] <specific, testable criterion>
- [ ] Tests pass

## Implementation Notes

<Key files to read or modify. Patterns to follow. Pitfalls to avoid.>

### Files to modify

- `path/to/file.ext` — description of change

### Testing approach

<Describe the test strategy: which test files to create/modify, what to mock.>

### Dependencies

<List the phases this phase depends on, by identifier, or "None".>

## Activity
```

**c. Update `TODO.md`.** Add a block under `## Open` for the new plan with one nested line per phase, all unticked with status `todo`, following the TODO.md format above. Create `TODO.md` if it does not exist.

**d. Close the original item** when `ORIGINAL_ID` is set:

- `status: done`
- `labels`: `EXISTING_LABELS` minus `planning`
- `related`: add `<PLAN_ID>`
- `updated`: today
- Append an Activity entry headed `Planning complete` with body `Implementation tracked in <PLAN_ID> (<relative path of the primary plan file>).` and the attribution footer
- Tick its TODO line and show `done`; if it is a top-level item, move its block to `## Done`

Record the relative path of every created file.

### 9. Report to the User

After the status commit lands, report a summary:

- Original item identifier and path (now done), or a note that the plan was created from free text
- New primary plan identifier, title, and path
- Each phase identifier, title, and path
- The commit that recorded the plan (or the pending status PR URL when branch protection forced the fallback)
- Brief note on sequencing rationale
- Model pin: the `--model <MODEL_OVERRIDE>` flag recorded in the implementation instructions (if any), or a note that implementation will inherit the session-configured model
- Reminder: use `/simple-workflow:implement-issue <PLAN_ID>` to execute the full plan — append ` --model <MODEL_OVERRIDE>` to this reminder when `MODEL_OVERRIDE` is set, so the documented handoff preserves the pin

## Key Principles

- **Files are the tracker**: Plan items, phases, status, labels, dependencies, and activity all live in markdown files under `PLANS_DIR`, on `main`. Nothing is stored anywhere else.
- **Commit and push as you go**: Every status change is its own status commit on `main`, made through a temporary detached worktree. Never leave plan changes uncommitted or on a side branch.
- **Planning label first**: When planning an existing item, add `planning` before exploring, so operators see the item is owned.
- **Agent attribution on every Activity entry**: End every entry with the `🤖 <AGENT_SIGNATURE>` footer so colleagues know they are reading agent output, not the human account owner.
- **Infer before asking**: Try to infer acceptance criteria from the prompt. Only ask when truly ambiguous.
- **Project-agnostic**: Read conventions from CLAUDE.md / AGENTS.md. Do not hardcode paths, build commands, or phase templates. The plans folder location itself comes from the repository or defaults to top-level `plans/`.
- **New plan, not overwrite**: Create a new primary plan file for implementation. The original item's task was "plan this" — mark it done and relate it to the new plan.
- **Directory is the parent/child relationship**: A phase's directory is its parent's file path minus `.md`; the `parent` front matter field repeats it for grep-ability.
- **Self-contained phases**: Each phase leaves the codebase compiling and tests passing, and each phase file carries everything a fresh agent needs to implement it.
- **Cleanup phase**: Every plan ends with a cleanup phase.
- **Session model by default**: Record a `--model` flag in the implementation instructions only when the user passes `--model`. Never write model names into labels — implementation always runs in the harness that invokes `implement-issue`, inheriting its session model unless the flag overrides it.
- **Harness-agnostic**: Plans assume nothing about the agent harness that will implement them. `implement-issue` branches every phase from `origin/main` inside whatever worktree the harness provides, so plan files carry no harness-specific routing metadata.
- **Sequential dependency chain**: Set `blocked_by` between consecutive phases only (plus any explicit non-adjacent dependency); never set a phase to block its parent.

## File Operations Cheat Sheet

| Action | How |
|--------|-----|
| Resolve plans folder | `CLAUDE.md` / `AGENTS.md` `Plans directory:` line or `## Plans` section; else `plans/` |
| Next identifier | Highest `[0-9]{3,}-` prefix in `PLANS_DIR`, plus one, zero-padded to 3 digits |
| Find item by identifier | File under `PLANS_DIR` whose front matter `id` matches (path also encodes it) |
| Read current status | `git fetch origin && git show origin/main:<path>` |
| Create / update item | Edit front matter and body inside the `RECORD_STATUS` worktree |
| Post activity | Append `### <timestamp> — <heading>` entry to `## Activity` with the footer |
| Record status | `RECORD_STATUS`: detached worktree at `origin/main`, commit, push `HEAD:main`, PR fallback |
| Update TODO | Edit the matching line in `<PLANS_DIR>/TODO.md` in the same status commit |
