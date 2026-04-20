---
name: setup-linear-labels
description: One-time setup skill to create the Role label group and the four canonical role labels (Planner, Orchestrator, Implementer, Maintainer) in a Linear team. Run this once per Linear team before using /plan-primary-issue, /implement-primary-issue, or /implement-sub-issue. Triggers on phrases like "set up linear labels", "create role labels", "initialize role group", or when the user reports that role labels are missing.
version: 1.0.0
---

# Setup Linear Role Labels

One-time setup for a Linear team. Creates the canonical `Role` label group plus its four child labels — `Planner`, `Orchestrator`, `Implementer`, `Maintainer` — that the `linear-workflow` skills use to route tickets to the correct Cyrus agent role.

The other three skills in this plugin (`/plan-primary-issue`, `/implement-primary-issue`, `/implement-sub-issue`) assume these labels already exist and will not create them on the fly — that check used to burn tokens on every skill invocation. Running this setup skill once per Linear team is the replacement.

## Arguments

`{{SKILL_INPUT}}` is the Linear team key (e.g., `HOL`) or team UUID. If empty, ask the user for the team key.

## Workflow

### 1. Resolve the Team

Parse `{{SKILL_INPUT}}` to extract the team key (e.g., `HOL`). If missing, ask the user which Linear team to set up.

Call `mcp__linear-server__get_team` with `query: "<TEAM_KEY>"` to confirm the team exists and capture:

- `TEAM_ID` — UUID (needed for `teamId` on `create_issue_label`)
- `TEAM_KEY` — e.g., `HOL`
- `TEAM_NAME`

### 2. Create the `Role` Label Group

Call `mcp__linear-server__create_issue_label`:

- `name: "Role"`
- `isGroup: true`
- `teamId: "<TEAM_ID>"`
- `description: "Role labels drive Cyrus routing — exactly one of Planner, Orchestrator, Implementer, or Maintainer per ticket."`
- `color: "#5e6ad2"`

If the group already exists, Linear returns an error — treat that as a no-op and proceed.

### 3. Create the Four Role Labels

Create each child label under the `Role` group. For every label, call `mcp__linear-server__create_issue_label`:

| Name | Color | Description |
|---|---|---|
| `Planner` | `#26b5ce` | Routed to planner Cyrus agent (scoper mode). |
| `Orchestrator` | `#5e6ad2` | Ticket is owned by the orchestrator agent; drives plan execution. |
| `Implementer` | `#f2c94c` | Routed to implementer Cyrus agent (builder mode). |
| `Maintainer` | `#f7c8c1` | Routed to maintainer Cyrus agent (cleanup/follow-up work). |

For each row, the call is:

- `name: "<name>"`
- `parent: "Role"`
- `teamId: "<TEAM_ID>"`
- `color: "<color>"`
- `description: "<description>"`

If a child label already exists, skip it — this skill is idempotent.

### 4. Verify

Call `mcp__linear-server__list_issue_labels` with `team: "<TEAM_KEY>"` and filter the results to confirm all four labels exist with `parent: "Role"`.

Report to the user:

- Team name + key
- Each of the four labels, with a `✓` if it exists and was verified
- Next step: configure the Cyrus `~/.cyrus/config.json` `routingLabels` / `labelPrompts` entries to match these names (see the README for a sample stanza)

## Cyrus Configuration Reference

After the labels exist in Linear, wire them into Cyrus so each role routes to the correct agent. Sample `~/.cyrus/config.json` stanza:

```json
{
  "repositories": [
    {
      "name": "holos-console",
      "repositoryPath": "/home/jeff/workspace/holos-run/holos-console",
      "baseBranch": "main",
      "routingLabels": ["Implementer", "Maintainer", "Orchestrator", "Planner"],
      "labelPrompts": {
        "scoper": { "labels": ["Planner", "Orchestrator"], "allowedTools": "readOnly" },
        "builder": { "labels": ["Implementer"], "allowedTools": "all" },
        "debugger": { "labels": ["Maintainer"], "allowedTools": "safe" }
      }
    }
  ]
}
```

## Key Principles

- **Run once per team**: This skill is idempotent, but there's no reason to run it more than once per Linear team unless a label was deleted.
- **Canonical names**: `Planner`, `Orchestrator`, `Implementer`, `Maintainer` — capitalized, no `role/` prefix, under the `Role` parent group.
- **Team-scoped labels**: Each team gets its own set. Do not create these as workspace labels — per-team labels keep routing and reporting clean when multiple teams use the plugin.
- **Colors are suggestions**: The colors in the table match the existing HOL team labels for consistency. Feel free to change them, but keep the names verbatim — skills match on name.

## Linear API Cheat Sheet

| Action | Tool | Key arguments |
|--------|------|---------------|
| Fetch team | `mcp__linear-server__get_team` | `query: "<TEAM_KEY>"` |
| Create label group | `mcp__linear-server__create_issue_label` | `name`, `isGroup: true`, `teamId` |
| Create child label | `mcp__linear-server__create_issue_label` | `name`, `parent`, `teamId`, `color`, `description` |
| List labels | `mcp__linear-server__list_issue_labels` | `team: "<TEAM_KEY>"` |
