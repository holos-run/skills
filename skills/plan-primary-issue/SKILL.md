---
name: plan-primary-issue
description: Create an implementation plan for a Linear ticket. Use this skill when the user provides a Linear ticket (URL or identifier like HOL-525) and wants a plan broken into phases, with each phase tracked as a Linear sub-ticket under the provided parent. Triggers on phrases like "plan this ticket", "plan ticket for", "break this Linear issue into phases", or any request to produce a phased plan against a Linear ticket.
version: 1.0.0
---

# Plan Ticket

You are a principal platform engineer. Explore the codebase and produce an implementation plan, broken down into phases for a platform engineer to implement. Write the master planning document into the provided Linear ticket's description, then create Linear sub-tickets (children of the provided ticket) for each phase. Ask the user for clarification if the acceptance criteria is incomplete or unclear; otherwise infer it from the prompt. Meet the acceptance criteria primarily through proto interfaces (backwards compatibility MUST be preserved), testing the RPC interfaces, and unit testing the frontend UI against mock RPC services. Use E2E tests for acceptance criteria as a last resort. Explore the codebase, then write your implementation plan for the Linear ticket **{{SKILL_INPUT}}**.

## Linear vs GitHub

This skill operates against Linear (not GitHub) using the `mcp__linear-server__*` tools:

- "Ticket" = Linear issue (the unit of work). The **provided** ticket becomes the parent/master for this plan.
- "Sub-ticket" = Linear child issue (`parentId` set to the provided ticket). Sub-tickets replace the GitHub sub-issue task-list pattern — Linear has native parent/child, so do not rely on checkbox parsing.
- "Master planning document" = the description (`body`) of the provided Linear ticket. Updating the ticket body with the full plan is the Linear analogue of creating a master GitHub issue with the plan body.

Always send real newlines in `body` / `description` values — never literal `\n` escape sequences.

## Workflow

### 1. Resolve the Ticket and Record the Agent Slot

Parse `{{SKILL_INPUT}}` to extract the Linear ticket identifier. Accept either form:

- Identifier: `HOL-525`
- URL: `https://linear.app/<workspace>/issue/HOL-525/<slug>`

Capture the identifier (e.g., `HOL-525`) for use in every subsequent Linear call.

Determine the **agent slot number** from the current working directory. Worktrees are named `holos-console-agent-<N>`:

```bash
SLOT=$(basename "$(pwd)" | sed -n 's/.*agent-\([0-9]\+\).*/agent-\1/p')
# e.g., /home/jeff/workspace/holos-run/holos-console-agent-1 -> SLOT=agent-1
```

If `$SLOT` is empty (unusual working directory), fall back to `SLOT="agent-unknown-$(basename "$(pwd)")"`.

Fetch the ticket so you have its title, team, current labels, and current body:

Call `mcp__linear-server__get_issue` with `id: "<IDENTIFIER>"`.

Record:
- `TICKET_ID` — Linear issue UUID (needed for `parentId` on sub-tickets)
- `TICKET_IDENTIFIER` — e.g., `HOL-525`
- `TICKET_TITLE`
- `TEAM_ID` / `TEAM_KEY` — needed to create sub-tickets on the same team
- `EXISTING_LABELS` — preserved when adding `planning`
- `EXISTING_BODY` — preserve or extend; never silently discard prior content

### 2. Mark the Ticket as Being Planned

Add the `planning` and `role/planner` labels and announce the agent slot on the ticket **before exploring the codebase**. This way operators can see at a glance which tickets an agent currently owns, and Cyrus's `routingLabels` matches `role/planner` to this repo's planner prompt (see [README — Multi-agent handoff](../../README.md)).

#### 2a. Add the `planning` and `role/planner` labels

For each label (`planning`, `role/planner`), call `mcp__linear-server__list_issue_labels` to find its ID for the ticket's team (prefer team-scoped; fall back to workspace). If missing, create it via `mcp__linear-server__create_issue_label` with `name: "<label>"` and `team: "<TEAM_KEY>"`.

Then update the issue labels via `mcp__linear-server__save_issue`:

- `issue: "<TICKET_IDENTIFIER>"`
- `labels: ["planning", "role/planner", ...existing label names]`

Preserve every existing label — `save_issue` replaces the full label set on update. Strip any existing `role/*` label before adding `role/planner` so the ticket is only routed to one role at a time.

#### 2b. Post the agent slot comment

Call `mcp__linear-server__save_comment` with:

- `issue: "<TICKET_IDENTIFIER>"`
- `body`:

  ```
  Planning this ticket.

  - Agent slot: <SLOT>
  ```

Send real newlines — not `\n` escape sequences.

### 3. Name the Session

Rename the current session so the human operator can see what this agent is working on:

```
/rename Plan <TICKET_IDENTIFIER> <ticket title>
```

For example: `/rename Plan HOL-525 Send id_token instead of access_token`.

### 4. Clarify Acceptance Criteria

Review the ticket body and the user's prompt. If the acceptance criteria is ambiguous or incomplete, ask the user targeted clarifying questions before exploring the codebase. If the prompt is sufficiently clear, infer the acceptance criteria and proceed.

### 5. Explore the Codebase

Explore the relevant areas of the codebase to understand:

- Existing proto definitions in `proto/holos/console/v1/`
- Related Go packages in `console/`
- Related frontend routes and components in `frontend/src/routes/` and `frontend/src/`
- Generated types in `gen/` and `frontend/src/gen/`
- Existing test patterns in `*_test.go` and `frontend/src/**/*.test.tsx`
- `AGENTS.md` for project conventions and architecture

Read `AGENTS.md` early — it describes the package structure, testing strategy, and code generation workflow that the plan must follow.

### 6. Draft the Implementation Plan

Break the implementation into sequential phases. Each phase should be a self-contained unit of work that leaves the codebase in a working state.

**Typical phase ordering** (skip phases that are not needed):

1. **Proto changes** — Add or modify RPC messages and service definitions in `proto/`. Run `make generate` to regenerate Go and TypeScript types. No implementation yet.
2. **Backend implementation** — Implement the Go handler(s), K8s backend logic, RBAC, and resolver changes. Include Go unit tests.
3. **Frontend implementation** — Add or modify React routes, components, and query hooks. Include UI unit tests with mocked query hooks.
4. **Integration / E2E** — Only if the acceptance criteria cannot be fully verified by unit tests.
5. **Cleanup** — Remove dead code, stale comments, and outdated documentation made stale by the implementation.

**Testing strategy per phase:**

- Proto/backend phases: table-driven Go tests using `k8s.io/client-go/kubernetes/fake`
- Frontend phases: Vitest + React Testing Library with `vi.mock()` on query hooks
- E2E only when full-stack round-trips are required (real K8s or OIDC flows)

**Backwards compatibility:** Proto interface changes MUST preserve backwards compatibility. Existing fields and message structures must not be removed or renumbered. New fields may be added.

### 7. Write the Master Planning Document to the Provided Ticket

Update the provided Linear ticket's description (`body`) with the full master plan. This description **is** the master planning document for this work.

Call `mcp__linear-server__save_issue` with:

- `issue: "<TICKET_IDENTIFIER>"`
- `description: "<master-plan-markdown>"`

Use this master plan template (substitute real content, real newlines):

```markdown
## Problem

<Describe the problem or motivation. Why is this needed?>

## Acceptance Criteria

- [ ] <criterion 1>
- [ ] <criterion 2>
- [ ] ...

## Implementation Plan

This ticket is the master planning document. Each phase below is tracked as a Linear sub-ticket (child of this ticket). Implement phases in order — each phase must leave the codebase in a working state.

- <!-- Phase sub-tickets will be listed here after creation in step 9 -->

## Implementation Instructions for Agents

To implement any phase sub-ticket of this plan, invoke the `/implement-sub-issue` skill (not `/implement-issue`) with the sub-ticket's Linear identifier. Do **not** use GitHub-issue-based skills for this plan — the phase tickets live in Linear.

## Notes

- Proto interface changes MUST preserve backwards compatibility.
- Tests over E2E: unit tests with mocked query hooks for UI; table-driven Go tests with fake K8s client for backend. E2E is a last resort.
- Every plan ends with a cleanup phase.
```

If the provided ticket already had a non-empty body, either merge it into the `## Problem` section or preserve it verbatim at the bottom under a `## Original Ticket Body` heading — never silently discard prior content.

### 8. Create Sub-Tickets for Each Phase

For each phase, create a Linear sub-ticket (child of the provided ticket). **Create sub-tickets without the `role/implementer` label first**, so Cyrus's implementer agent does not pick them up before the plan body is finalized in step 9. The label is applied in step 10 once the master plan is fully wired up — this avoids racing an implementer against an unfinished plan.

Call `mcp__linear-server__save_issue` with (no `issue` field — this creates a new issue):

- `team: "<TEAM_KEY>"` — same team as the parent ticket
- `parentId: "<TICKET_ID>"` — the UUID of the provided ticket (from step 1), making this a sub-ticket
- `title: "<conventional-commit-prefix>(<scope>): <phase title>"`
- `description`:

  ```markdown
  ## Parent Ticket

  Part of <TICKET_IDENTIFIER>

  ## Goal

  <One-paragraph description of what this phase accomplishes and why.>

  ## Acceptance Criteria

  - [ ] <specific, testable criterion>
  - [ ] <specific, testable criterion>
  - [ ] Tests pass: `make test`

  ## Implementation Notes

  <Key files to read or modify. Patterns to follow. Pitfalls to avoid.>

  ### Files to modify

  - `proto/holos/console/v1/foo.proto` — add `BarRequest` message
  - `console/rpc/foo.go` — implement `Bar` handler
  - `frontend/src/routes/foo/bar.tsx` — add route

  ### Testing approach

  <Describe the test strategy: which test files to create/modify, what to mock, table-driven vs unit.>

  ### Dependencies

  <List any phase sub-tickets this phase depends on, e.g. "Depends on <IDENTIFIER> (proto changes)".>

  ## How This Ticket Is Triggered

  This ticket is auto-triggered when the `role/implementer` label is applied by the `/implement-primary-issue` orchestrator. The implementer Cyrus agent (matched via `routingLabels: ["role/implementer"]`) will pick it up, run `/implement-sub-issue`, and post progress back here.

  To implement this ticket manually in a local Claude Code session, invoke `/implement-sub-issue <IDENTIFIER>`. Do not use `/implement-issue` — that is the GitHub variant.
  ```

Record each created sub-ticket's `identifier` (e.g., `HOL-601`) and `id` (UUID) so you can list them in the master document (step 9).

### 9. Update the Master Planning Document with Phase References

After all sub-tickets are created, update the parent ticket body to replace the placeholder phase list with real Linear identifiers. Call `mcp__linear-server__save_issue` with `issue: "<TICKET_IDENTIFIER>"` and `description: "<updated-markdown>"`, where the `## Implementation Plan` section now reads:

```markdown
## Implementation Plan

This ticket is the master planning document. Each phase below is tracked as a Linear sub-ticket (child of this ticket). Implement phases in order — each phase must leave the codebase in a working state.

- [ ] <PHASE_1_IDENTIFIER> — <phase 1 title>
- [ ] <PHASE_2_IDENTIFIER> — <phase 2 title>
- [ ] <PHASE_3_IDENTIFIER> — <phase 3 title>
...
```

Linear will render each identifier as a cross-link. The leading checkbox is informational — Linear tracks sub-ticket completion natively via parent/child state, and `/implement-primary-issue` iterates by fetching children rather than parsing checkboxes.

### 10. Leave Phase Sub-Tickets Unlabeled for Orchestrator Handoff

**Default (recommended):** Do **not** apply `role/implementer` to phase sub-tickets here. Leave them in their default state. The `/implement-primary-issue` orchestrator applies `role/implementer` to each sub-ticket one at a time as it dequeues it, so only one implementer agent runs at a time and the plan is executed in strict creation order.

**Opt-in eager mode:** If the operator explicitly wants all phase sub-tickets to become available to the implementer agent pool immediately (e.g., for independent phases that can run in parallel), apply `role/implementer` to each phase sub-ticket now:

```
mcp__linear-server__save_issue
  issue: "<SUB_IDENTIFIER>"
  labels: ["role/implementer", ...existing labels]
```

Ensure the `role/implementer` label exists for the team (create via `mcp__linear-server__create_issue_label` if missing). Prefer the default orchestrator-driven mode unless the user has explicitly asked for parallel phase execution — parallel phases multiply merge-conflict risk on shared files.

### 11. Report to the User

After all tickets are created, report a summary:

- Parent ticket identifier, title, and URL
- Each phase sub-ticket identifier, title, and URL
- Brief note on the sequencing rationale (why phases are ordered as they are)
- Reminder: use `/implement-primary-issue <PARENT_IDENTIFIER>` to execute the full plan, or `/implement-sub-issue <PHASE_IDENTIFIER>` to implement a single phase

## Key Principles

- **Planning label first**: Always add the `planning` label and post the agent-slot comment *before* exploring the codebase, so operators can see the ticket is owned.
- **Infer before asking**: Try to infer acceptance criteria from the prompt. Only ask for clarification when truly ambiguous.
- **Proto-first**: When the feature requires new RPC messages, always make proto changes a dedicated first phase.
- **Backwards compatibility**: Never remove or renumber existing proto fields. Only add new ones.
- **Tests over E2E**: Unit tests with mocked query hooks for UI; table-driven Go tests with fake K8s client for backend. E2E is a last resort.
- **Cleanup phase**: Every plan ends with a cleanup phase to remove stale code and documentation.
- **Self-contained phases**: Each phase should leave the codebase compiling and tests passing.
- **Native parent/child**: Use Linear's `parentId` for sub-tickets. Do not rely on markdown task-list parsing — that is the GitHub pattern.
- **Refer to `/implement-sub-issue`**: Every phase sub-ticket description and the master plan must instruct agents to use `/implement-sub-issue` (Linear), not `/implement-issue` (GitHub).

## Linear API Cheat Sheet

| Action | Tool | Key arguments |
|--------|------|---------------|
| Fetch ticket | `mcp__linear-server__get_issue` | `id: "<IDENTIFIER>"` |
| List labels | `mcp__linear-server__list_issue_labels` | `team: "<TEAM_KEY>"` |
| Create label | `mcp__linear-server__create_issue_label` | `name`, `team` |
| Update labels / body | `mcp__linear-server__save_issue` | `issue`, `labels` or `description` |
| Create sub-ticket | `mcp__linear-server__save_issue` | `team`, `parentId`, `title`, `description` |
| Post comment | `mcp__linear-server__save_comment` | `issue`, `body` |

Always send markdown bodies with real newlines, not `\n` literals.
