---
name: plan-issue
description: Create an implementation plan for a Linear issue. Use this skill when the user provides a Linear issue (URL or identifier like APP-123) and wants a plan broken into phases, with each phase tracked as a Linear sub-issue. Creates a NEW primary issue with the structured plan and relates it back to the original. Accepts an optional --model argument to pin implementation to a specific model; otherwise implementation inherits the session-configured model. Triggers on phrases like "plan this issue", "plan issue for", "break this Linear issue into phases", or any request to produce a phased plan against a Linear issue.
version: 2.6.0
---

# Plan Issue

You are a principal engineer. Explore the codebase, produce an implementation plan broken into phases, and write it into a new Linear primary issue. Create sub-issues (children of the new primary issue) for each phase. Relate the new primary issue back to the original input issue, then mark the original Done.

The input issue represents the task "plan this feature." When planning is complete, that task is done. The new primary issue represents the task "implement this feature" and has its own lifecycle.

Plan the Linear issue **{{SKILL_INPUT}}**.

## Arguments

`{{SKILL_INPUT}}` contains the issue reference and optional flags:

```
<issue> [--model <fable|opus|sonnet|haiku|codex>]
```

- `<issue>` — Linear issue identifier (`APP-123`) or URL.
- `--model <name>` (also accepted as `model=<name>`) — pins implementation of this plan to a specific model. When present, record it as `MODEL_OVERRIDE` and apply it as a routing label to the primary issue and every phase sub-issue (steps 7–8), so `implement-issue` dispatches all work to that model. When absent, apply **no** routing label — implementation workers then inherit the model configured for the Claude Code session (e.g., Fable or Opus). Inheriting the session model is the default and preferred behavior; only pin a model when the user explicitly asks for one.

Strip any flags from `{{SKILL_INPUT}}` before parsing the issue reference in step 1.

## Linear Conventions

This skill operates against Linear using `mcp__linear-server__*` tools:

- **Issue** = Linear issue (the unit of work).
- **Sub-issue** = Linear child issue (`parentId` set to the new primary issue).
- Linear has native parent/child relationships — do not use markdown task-list parsing.
- Always send real newlines in `body` / `description` values — never literal `\n` escape sequences.

## Workflow

### 1. Resolve the Input Issue

Parse `{{SKILL_INPUT}}` to extract the Linear issue identifier. Accept either form:

- Identifier: `APP-123`
- URL: `https://linear.app/<workspace>/issue/APP-123/<slug>`

Fetch the issue via `mcp__linear-server__get_issue` with `id: "<IDENTIFIER>"`.

Record:

- `ORIGINAL_ID` — Linear UUID
- `ORIGINAL_IDENTIFIER` — e.g., `APP-123`
- `ORIGINAL_TITLE`
- `TEAM_KEY` — e.g., `APP`
- `TEAM_ID`
- `EXISTING_LABELS` — preserved when adding `planning`
- `EXISTING_BODY` — the original description (preserved in the new issue)

### 2. Mark the Original as Being Planned

Add the `planning` label to the original issue before exploring the codebase. This signals to operators that an agent owns this issue.

Ensure the `planning` label exists for the team via `mcp__linear-server__list_issue_labels`. If missing, create it via `mcp__linear-server__create_issue_label`.

Update the original issue via `mcp__linear-server__save_issue`:

- `issue: "<ORIGINAL_IDENTIFIER>"`
- `state: "In Progress"`
- `labels: ["planning", ...existing label names]`

Post a comment via `mcp__linear-server__save_comment`:

- `issue: "<ORIGINAL_IDENTIFIER>"`
- `body`: "Planning this issue."

### 3. Read Project Conventions

Before exploring the codebase, read the project's configuration files to understand conventions:

1. Read `CLAUDE.md` if it exists — for project conventions, testing strategy, build commands
2. Read `AGENTS.md` if it exists — for architecture, package structure, code generation workflows
3. Read `CONTRIBUTING.md` if it exists — for commit message format, PR conventions

These files determine how to structure phases (e.g., whether the project uses proto-first development, what testing strategy to follow, what build toolchain to use). Do not hardcode assumptions — derive them from the project.

### 4. Clarify Acceptance Criteria

Review the issue body and the user's prompt. If the acceptance criteria is ambiguous or incomplete, ask the user targeted clarifying questions before exploring the codebase. If the prompt is sufficiently clear, infer the acceptance criteria and proceed.

### 5. Explore the Codebase

Explore the relevant areas of the codebase to understand the architecture, existing patterns, and test conventions. Use the information from project configuration files (step 3) to guide exploration.

### 6. Draft the Implementation Plan

Break the implementation into sequential phases. Each phase should be a self-contained unit of work that leaves the codebase in a working state.

**Principles:**

- Phase ordering should follow the project's natural dependency graph (e.g., interface changes before implementation, backend before frontend).
- Each phase includes the testing approach appropriate for the project (read from CLAUDE.md / AGENTS.md).
- Every plan ends with a cleanup phase to remove stale code and documentation.
- Phases should be small enough for one agent to implement in a single session, and large enough to be a meaningful PR. Prefer 2–6 phases; merge trivial phases into their neighbors rather than creating one-line-change sub-issues.
- **Each sub-issue must be self-contained.** The implementing agent sees only the sub-issue description — it has no access to this planning conversation. Write each description so a fresh agent can implement it without further context: exact file paths, function/type names discovered during exploration, the verification commands to run, and the specific patterns to follow.
- State inter-phase dependencies explicitly in each sub-issue's Dependencies section, by identifier. Phases are implemented sequentially in creation order, so order the sub-issues so every dependency comes before its dependents.
- Quote concrete acceptance criteria from the original issue verbatim where possible, and distribute every criterion to exactly one phase — no criterion may be left unassigned.

### 7. Create the New Primary Issue

Create a new Linear issue that serves as the implementation primary issue. Call `mcp__linear-server__save_issue` with:

- `team: "<TEAM_KEY>"`
- `title: "<ORIGINAL_TITLE>"` (or a refined version that reflects the plan)
- `description`: the master plan markdown (see template below)
- `labels: ["<MODEL_OVERRIDE>"]` — only when `MODEL_OVERRIDE` is set. First ensure a label named exactly `<MODEL_OVERRIDE>` (e.g., `opus`, `fable`) exists for the team via `mcp__linear-server__list_issue_labels`, creating it via `mcp__linear-server__create_issue_label` if missing. When `MODEL_OVERRIDE` is unset, apply no routing label so implementation inherits the session model.

Record the new issue's `identifier` (e.g., `APP-124`) and `id` (UUID) as `PRIMARY_IDENTIFIER` and `PRIMARY_ID`.

**Master plan template** (substitute real content, real newlines):

```markdown
## Problem

<Describe the problem or motivation from the original issue.>

## Acceptance Criteria

- [ ] <criterion 1>
- [ ] <criterion 2>
- [ ] ...

## Implementation Plan

This issue is the primary implementation issue. Each phase below is tracked as a sub-issue (child of this issue). Implement phases in order — each phase must leave the codebase in a working state.

- <!-- Phase sub-issues will be listed here after creation -->

## Implementation Instructions

To implement all phases, invoke `/linear-workflow:implement-issue <PRIMARY_IDENTIFIER>`.
To implement a single phase, invoke `/linear-workflow:implement-issue <PHASE_IDENTIFIER>`.
Workers inherit the model configured in the Claude Code session unless a routing label or a `--model <name>` argument overrides it.
Each phase branches from `origin/main` and merges to `main` via its own independent PR.

## Original Issue

Planned from <ORIGINAL_IDENTIFIER>.
```

If the original issue had a non-empty body, preserve it in the `## Problem` section or under a `## Original Description` heading.

### 8. Create Sub-Issues for Each Phase

For each phase, create a Linear sub-issue as a child of the new primary issue.

Sub-issues need no base-branch metadata: `implement-issue` creates every phase's feature branch directly from `origin/main` regardless of which branch the agent harness checked out in its worktree, so each phase produces an independent PR against `main`.

Call `mcp__linear-server__save_issue` with:

- `team: "<TEAM_KEY>"`
- `parentId: "<PRIMARY_ID>"`
- `title: "<conventional-commit-prefix>(<scope>): <phase title>"`
- `labels: ["<MODEL_OVERRIDE>"]` — only when `MODEL_OVERRIDE` is set (label already ensured in step 7); omit otherwise so the phase inherits the session model at implementation time
- `description`:

```markdown
## Parent

Part of <PRIMARY_IDENTIFIER>

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

<List any phase sub-issues this phase depends on.>
```

Record each created sub-issue's `identifier` and `id`.

After creating all phase sub-issues, add Linear dependency relationships between them so the execution order is visible in Linear itself:

- Link phases as a sequential chain: phase 1 blocks phase 2, phase 2 blocks phase 3, etc.
- For each adjacent pair `(PREV, CURR)`, call `mcp__linear-server__save_issue` on the previous phase issue with:
  - `issue: "<PREV_IDENTIFIER>"`
  - `blocks: ["<CURR_IDENTIFIER>"]`
- Equivalent `blockedBy` relationships will appear on downstream phases in Linear.

Do **not** create blocking relationships between the parent primary issue and any phase sub-issue. The parent issue should remain independently movable to `In Progress` while children are implemented over time.

### 9. Update the Primary Issue with Phase References

After all sub-issues are created, update the primary issue description to list them. Call `mcp__linear-server__save_issue` with `issue: "<PRIMARY_IDENTIFIER>"` and `description`. Only the `## Implementation Plan` section changes. The updated section reads:

```markdown
## Implementation Plan

This issue is the primary implementation issue. Each phase below is tracked as a sub-issue (child of this issue). Implement phases in order — each phase must leave the codebase in a working state.

- [ ] <PHASE_1_IDENTIFIER> — <phase 1 title>
- [ ] <PHASE_2_IDENTIFIER> — <phase 2 title>
- [ ] <PHASE_3_IDENTIFIER> — <phase 3 title>
```

Linear renders each identifier as a cross-link. The checkboxes are informational — Linear tracks sub-issue completion natively.

### 10. Link and Close the Original Issue

Link the original issue to the new primary issue via `mcp__linear-server__save_issue`:

- `issue: "<ORIGINAL_IDENTIFIER>"`
- `relatedTo: ["<PRIMARY_IDENTIFIER>"]`

Post a comment on the original issue:

- `issue: "<ORIGINAL_IDENTIFIER>"`
- `body`:

```
Planning complete. Implementation tracked in <PRIMARY_IDENTIFIER>.
```

Mark the original issue Done:

- `issue: "<ORIGINAL_IDENTIFIER>"`
- `state: "Done"`
- `labels: [... existing labels minus "planning"]`

If the team does not have a state named exactly `Done`, use `mcp__linear-server__list_issue_statuses` and pick the `completed`-type state.

### 11. Report to the User

After all issues are created, report a summary:

- Original issue identifier and URL (now Done)
- New primary issue identifier, title, and URL
- Each phase sub-issue identifier, title, and URL
- Brief note on sequencing rationale
- Model routing: the `<MODEL_OVERRIDE>` label applied (if any), or a note that implementation will inherit the session-configured model
- Reminder: use `/linear-workflow:implement-issue <PRIMARY_IDENTIFIER>` to execute the full plan

## Key Principles

- **Planning label first**: Add `planning` before exploring, so operators see the issue is owned.
- **Infer before asking**: Try to infer acceptance criteria from the prompt. Only ask when truly ambiguous.
- **Project-agnostic**: Read conventions from CLAUDE.md / AGENTS.md. Do not hardcode paths, build commands, or phase templates.
- **New issue, not overwrite**: Create a new primary issue for implementation. The original issue's task was "plan this" — mark it Done.
- **Related, not child**: Link original → new primary via `relatedTo`, not `parentId`.
- **Native parent/child**: Sub-issues use Linear's `parentId` under the new primary issue.
- **Self-contained phases**: Each phase leaves the codebase compiling and tests passing, and each sub-issue description carries everything a fresh agent needs to implement it.
- **Cleanup phase**: Every plan ends with a cleanup phase.
- **Session model by default**: Apply a model routing label only when the user passes `--model`. Otherwise leave routing labels off so implementation inherits the model configured for the Claude Code session.
- **Harness-agnostic**: Plans assume nothing about the agent harness that will implement them (Claude Code remote sessions, local worktrees, Cyrus, etc.). `implement-issue` branches every phase from `origin/main` inside whatever worktree the harness provides, so issue descriptions carry no harness-specific routing metadata.
- **Sub-issue dependency graph**: Add `blocks` relationships between consecutive phase sub-issues only; never set phase sub-issues to block the parent issue.

## Linear API Cheat Sheet

| Action | Tool | Key arguments |
|--------|------|---------------|
| Fetch issue | `mcp__linear-server__get_issue` | `id` |
| List labels | `mcp__linear-server__list_issue_labels` | `team` |
| Create label | `mcp__linear-server__create_issue_label` | `name`, `team` |
| Update issue | `mcp__linear-server__save_issue` | `issue`, `state` / `labels` / `description` / `relatedTo` |
| Create issue | `mcp__linear-server__save_issue` | `team`, `title`, `description` (no `issue` field) |
| Create sub-issue | `mcp__linear-server__save_issue` | `team`, `parentId`, `title`, `description` |
| Post comment | `mcp__linear-server__save_comment` | `issue`, `body` |
| List statuses | `mcp__linear-server__list_issue_statuses` | `team` |
