---
name: plan-issue
description: Create an implementation plan for a Linear ticket. Use this skill when the user provides a Linear ticket (URL or identifier like APP-123) and wants a plan broken into phases, with each phase tracked as a Linear sub-ticket. Creates a NEW primary issue with the structured plan and relates it back to the original. Triggers on phrases like "plan this ticket", "plan ticket for", "break this Linear issue into phases", or any request to produce a phased plan against a Linear ticket.
version: 2.0.0
---

# Plan Issue

You are a principal engineer. Explore the codebase, produce an implementation plan broken into phases, and write it into a new Linear primary issue. Create sub-tickets (children of the new primary issue) for each phase. Relate the new primary issue back to the original input issue, then mark the original Done.

The input issue represents the task "plan this feature." When planning is complete, that task is done. The new primary issue represents the task "implement this feature" and has its own lifecycle.

Plan the Linear ticket **{{SKILL_INPUT}}**.

## Linear Conventions

This skill operates against Linear using `mcp__linear-server__*` tools:

- **Ticket** = Linear issue (the unit of work).
- **Sub-ticket** = Linear child issue (`parentId` set to the new primary issue).
- Linear has native parent/child relationships — do not use markdown task-list parsing.
- Always send real newlines in `body` / `description` values — never literal `\n` escape sequences.

## Workflow

### 1. Resolve the Input Ticket

Parse `{{SKILL_INPUT}}` to extract the Linear ticket identifier. Accept either form:

- Identifier: `APP-123`
- URL: `https://linear.app/<workspace>/issue/APP-123/<slug>`

Fetch the ticket via `mcp__linear-server__get_issue` with `id: "<IDENTIFIER>"`.

Record:

- `ORIGINAL_ID` — Linear UUID
- `ORIGINAL_IDENTIFIER` — e.g., `APP-123`
- `ORIGINAL_TITLE`
- `TEAM_KEY` — e.g., `APP`
- `TEAM_ID`
- `EXISTING_LABELS` — preserved when adding `planning`
- `EXISTING_BODY` — the original description (preserved in the new issue)

### 2. Mark the Original as Being Planned

Add the `planning` label to the original ticket before exploring the codebase. This signals to operators that an agent owns this ticket.

Ensure the `planning` label exists for the team via `mcp__linear-server__list_issue_labels`. If missing, create it via `mcp__linear-server__create_issue_label`.

Update the original issue via `mcp__linear-server__save_issue`:

- `issue: "<ORIGINAL_IDENTIFIER>"`
- `state: "In Progress"`
- `labels: ["planning", ...existing label names]`

Post a comment via `mcp__linear-server__save_comment`:

- `issue: "<ORIGINAL_IDENTIFIER>"`
- `body`: "Planning this ticket."

### 3. Read Project Conventions

Before exploring the codebase, read the project's configuration files to understand conventions:

1. Read `CLAUDE.md` if it exists — for project conventions, testing strategy, build commands
2. Read `AGENTS.md` if it exists — for architecture, package structure, code generation workflows
3. Read `CONTRIBUTING.md` if it exists — for commit message format, PR conventions

These files determine how to structure phases (e.g., whether the project uses proto-first development, what testing strategy to follow, what build toolchain to use). Do not hardcode assumptions — derive them from the project.

### 4. Clarify Acceptance Criteria

Review the ticket body and the user's prompt. If the acceptance criteria is ambiguous or incomplete, ask the user targeted clarifying questions before exploring the codebase. If the prompt is sufficiently clear, infer the acceptance criteria and proceed.

### 5. Explore the Codebase

Explore the relevant areas of the codebase to understand the architecture, existing patterns, and test conventions. Use the information from project configuration files (step 3) to guide exploration.

### 6. Draft the Implementation Plan

Break the implementation into sequential phases. Each phase should be a self-contained unit of work that leaves the codebase in a working state.

**Principles:**

- Phase ordering should follow the project's natural dependency graph (e.g., interface changes before implementation, backend before frontend).
- Each phase includes the testing approach appropriate for the project (read from CLAUDE.md / AGENTS.md).
- Every plan ends with a cleanup phase to remove stale code and documentation.
- Phases should be small enough for one agent to implement in a single session.

### 7. Create the New Primary Issue

Create a new Linear issue that serves as the implementation primary issue. Call `mcp__linear-server__save_issue` with:

- `team: "<TEAM_KEY>"`
- `title: "<ORIGINAL_TITLE>"` (or a refined version that reflects the plan)
- `description`: the master plan markdown (see template below)

Record the new issue's `identifier` (e.g., `APP-124`) and `id` (UUID) as `PRIMARY_IDENTIFIER` and `PRIMARY_ID`.

**Master plan template** (substitute real content, real newlines):

```markdown
## Problem

<Describe the problem or motivation from the original ticket.>

## Acceptance Criteria

- [ ] <criterion 1>
- [ ] <criterion 2>
- [ ] ...

## Implementation Plan

This ticket is the primary implementation issue. Each phase below is tracked as a sub-ticket (child of this issue). Implement phases in order — each phase must leave the codebase in a working state.

- <!-- Phase sub-tickets will be listed here after creation -->

## Implementation Instructions

To implement all phases, invoke `/linear-workflow:implement-issue <PRIMARY_IDENTIFIER>`.
To implement a single phase, invoke `/linear-workflow:implement-issue <PHASE_IDENTIFIER>`.

## Original Ticket

Planned from <ORIGINAL_IDENTIFIER>.
```

If the original ticket had a non-empty body, preserve it in the `## Problem` section or under a `## Original Description` heading.

### 8. Create Sub-Tickets for Each Phase

For each phase, create a Linear sub-ticket as a child of the new primary issue. Call `mcp__linear-server__save_issue` with:

- `team: "<TEAM_KEY>"`
- `parentId: "<PRIMARY_ID>"`
- `title: "<conventional-commit-prefix>(<scope>): <phase title>"`
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

<List any phase sub-tickets this phase depends on.>
```

Record each created sub-ticket's `identifier` and `id`.

### 9. Update the Primary Issue with Phase References

After all sub-tickets are created, update the primary issue description to list them. Call `mcp__linear-server__save_issue` with `issue: "<PRIMARY_IDENTIFIER>"` and `description` where the `## Implementation Plan` section reads:

```markdown
## Implementation Plan

This ticket is the primary implementation issue. Each phase below is tracked as a sub-ticket (child of this issue). Implement phases in order — each phase must leave the codebase in a working state.

- [ ] <PHASE_1_IDENTIFIER> — <phase 1 title>
- [ ] <PHASE_2_IDENTIFIER> — <phase 2 title>
- [ ] <PHASE_3_IDENTIFIER> — <phase 3 title>
```

Linear renders each identifier as a cross-link. The checkboxes are informational — Linear tracks sub-ticket completion natively.

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

After all tickets are created, report a summary:

- Original ticket identifier and URL (now Done)
- New primary issue identifier, title, and URL
- Each phase sub-ticket identifier, title, and URL
- Brief note on sequencing rationale
- Reminder: use `/linear-workflow:implement-issue <PRIMARY_IDENTIFIER>` to execute the full plan

## Key Principles

- **Planning label first**: Add `planning` before exploring, so operators see the ticket is owned.
- **Infer before asking**: Try to infer acceptance criteria from the prompt. Only ask when truly ambiguous.
- **Project-agnostic**: Read conventions from CLAUDE.md / AGENTS.md. Do not hardcode paths, build commands, or phase templates.
- **New issue, not overwrite**: Create a new primary issue for implementation. The original issue's task was "plan this" — mark it Done.
- **Related, not child**: Link original → new primary via `relatedTo`, not `parentId`.
- **Native parent/child**: Sub-tickets use Linear's `parentId` under the new primary issue.
- **Self-contained phases**: Each phase leaves the codebase compiling and tests passing.
- **Cleanup phase**: Every plan ends with a cleanup phase.

## Linear API Cheat Sheet

| Action | Tool | Key arguments |
|--------|------|---------------|
| Fetch ticket | `mcp__linear-server__get_issue` | `id` |
| List labels | `mcp__linear-server__list_issue_labels` | `team` |
| Create label | `mcp__linear-server__create_issue_label` | `name`, `team` |
| Update issue | `mcp__linear-server__save_issue` | `issue`, `state` / `labels` / `description` / `relatedTo` |
| Create issue | `mcp__linear-server__save_issue` | `team`, `title`, `description` (no `issue` field) |
| Create sub-ticket | `mcp__linear-server__save_issue` | `team`, `parentId`, `title`, `description` |
| Post comment | `mcp__linear-server__save_comment` | `issue`, `body` |
| List statuses | `mcp__linear-server__list_issue_statuses` | `team` |
