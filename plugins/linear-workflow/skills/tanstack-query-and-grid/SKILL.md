---
name: tanstack-query-and-grid
description: Implement or refactor TanStack Query and TanStack Table grid behavior in holos-console. Use for query hooks, query key factories, mutation invalidation, ResourceGrid toolbar/search/filter/sort URL state, keepPreviousData, server-state coordination, and grid list cache behavior. Do not use for Go backend changes, protobuf schema work, or unrelated visual design.
version: 1.0.0
---

# TanStack Query And Grid

Use this skill for `holos-console` work that changes TanStack Query server
state, query-key factories, mutation invalidation, fan-out reads, or
ResourceGrid/TanStack Table behavior.

## Trigger Phrases

- "refactor query hooks"
- "add a query key factory"
- "fix mutation invalidation"
- "coordinate server state with grid filters"
- "add ResourceGrid toolbar state"
- "implement grid search/filter/sort URL state"
- "use keepPreviousData for list queries"
- "refactor TanStack Table columns or row actions"

## Non-Triggers

- Do not use for Go backend changes, controller logic, protobuf API design, or
  persistence-layer work.
- Do not use for unrelated CSS-only polish, visual rebrands, or marketing UI.
- Do not use for full page workflow implementation unless the core risk is
  TanStack Query, TanStack Table, or ResourceGrid state; use
  `enterprise-k8s-frontend` for broader operator-facing UI work.

## Architectural Lock List

Preserve the existing frontend stack and state boundaries:

- Vite + React only; do not introduce Next.js, Remix, server components, or a
  parallel frontend framework.
- Use TanStack Router for route/search state and navigation.
- Use TanStack Query for server state and cache invalidation.
- Use TanStack Table through ResourceGrid for dense resource tables.
- Use shadcn/Radix primitives, Tailwind, and lucide icons for UI controls.
- Use ConnectRPC through existing transport/query modules.
- Do not add ad-hoc query-key arrays outside the key factory module.

Source docs to read first:

- `docs/agents/tanstack-query-conventions.md`
- `docs/agents/data-grid-architecture.md`
- `docs/agents/frontend-architecture.md`

## Workflow

1. Read the issue, then read `CLAUDE.md` or `AGENTS.md` in the target repo for
   commands, test strategy, and code review expectations.
2. Read `docs/agents/tanstack-query-conventions.md` before editing query
   modules, mutation hooks, or invalidation behavior.
3. Read `docs/agents/data-grid-architecture.md` before changing ResourceGrid
   rows, toolbar state, filter/sort state, action cells, or table columns.
4. Start with `frontend/src/queries/keys.ts`. Every hand-written query key and
   invalidation call must use `keys.<resource>.<scope>(...)` or a documented
   prefix factory.
5. Keep transport and client creation inside query hooks. Routes and components
   should call hooks instead of constructing ConnectRPC clients directly.
6. Gate read hooks with authentication and all required identifiers. For
   optional reads, combine caller intent with the same auth/identifier guard.
7. In mutation hooks, invalidate the smallest cache scope that refreshes every
   visible stale surface. Keep navigation and toast copy in the caller.
8. Use `placeholderData: keepPreviousData` for list-style ResourceGrid reads
   when stale rows are less disruptive than blanking the table during route or
   search-param changes. Do not use it for sensitive detail payloads.
9. Keep client-only grid search, kind filters, and sort state in TanStack Router
   search params. Add server-used filters to query keys.
10. Preserve row navigation rules: `detailHref` for named resources with detail
    pages, TanStack Router `Link` for in-app cells, and `stopPropagation()` for
    row action controls.
11. Add focused tests at the lowest useful boundary: query-hook tests for cache
    and invalidation behavior, ResourceGrid tests for shared table behavior,
    and route tests for page-specific mapping and mocked query hooks.
12. Run the repo's documented checks, normally `make test-ui` and any lint or
    build target required by `CLAUDE.md` or `AGENTS.md`.

## Anti-Patterns

- Adding query keys as inline array literals in routes, components, or query
  modules.
- Invalidating broad cache scopes when a list/detail key would refresh the
  visible data.
- Putting toast copy, navigation, or route decisions inside mutation hooks.
- Prefetching list-row details by default without an issue-level latency
  decision and documented query scope.
- Applying keep-previous-data to secret values or other sensitive detail reads.
- Adding ResourceGrid fields to the base row type when only one route needs a
  kind-specific column.
- Using raw anchors for in-app navigation or letting row action clicks also
  trigger row navigation.
