---
name: enterprise-k8s-frontend
description: Build or refactor dense operator-facing Kubernetes frontend workflows in holos-console. Use for resource list/detail views, create/edit/delete flows, route-backed filters, ResourceGrid pages, selected organization/project context, and ConnectRPC-backed React UI work. Do not use for Go backend changes, protobuf API design, cluster runtime logic, infrastructure, or marketing pages.
version: 1.0.0
---

# Enterprise Kubernetes Frontend

Use this skill when implementing production UI in `holos-console` for
operators who manage Kubernetes resources, organizations, projects, folders,
templates, deployments, secrets, or policies.

## Trigger Phrases

- "build a resource list view in holos-console"
- "build a resource detail view in holos-console"
- "add a create/edit/delete frontend flow"
- "wire a Kubernetes resource page to ConnectRPC"
- "add ResourceGrid support for a new resource"
- "implement route-backed filters or selected project/org frontend state"

## Non-Triggers

- Do not use for Go backend handlers, controllers, reconciler logic, database
  migrations, protobuf schema design, or server authorization rules.
- Do not use for generic landing pages, marketing pages, or visual redesigns
  outside the authenticated operator console.
- Do not use for isolated TanStack Query key or mutation refactors unless the
  UI/resource workflow is also changing; use `tanstack-query-and-grid` instead.

## Architectural Lock List

Keep frontend work inside the existing Vite application:

- Vite + React only; do not introduce Next.js, Remix, server components, or a
  parallel frontend framework.
- Use TanStack Router for file-based routes and route search state.
- Use TanStack Query for server state; do not add local caches for RPC data.
- Use TanStack Table through the shared ResourceGrid patterns for dense lists.
- Use shadcn/Radix primitives, Tailwind, and lucide icons; do not introduce a
  second component system.
- Use ConnectRPC through the existing transport provider and query modules.
- Do not edit generated route tree files by hand.

Source docs to read first:

- `docs/agents/frontend-architecture.md`
- `docs/agents/data-grid-architecture.md`
- `docs/agents/tanstack-query-conventions.md`

## Workflow

1. Read the issue, then read `CLAUDE.md` or `AGENTS.md` in the target repo for
   local commands and test expectations.
2. Read `docs/agents/frontend-architecture.md` and identify the route,
   selected-entity, ConnectRPC, and build/test constraints that apply.
3. If the work adds or changes a dense resource list, read
   `docs/agents/data-grid-architecture.md` before touching components.
4. Locate the owning route under `frontend/src/routes/`, the query hook under
   `frontend/src/queries/`, and any shared UI primitive under
   `frontend/src/components/`.
5. Keep route params authoritative over selected-entity stores. Sync URL params
   into stores from layout routes only; creation pages may read fallback stores
   but must not write selected org/project state.
6. Use TanStack Router search params for user-visible filter, search, sort, and
   navigation state. Preserve `returnTo` behavior with
   `buildReturnTo()` / `resolveReturnTo()` where create flows return to a
   caller page.
7. Map API objects into stable UI row/view models at the route or feature
   boundary. Keep ResourceGrid rows compact, scan-friendly, and navigable with
   `detailHref` when a detail page exists.
8. Stop event propagation from row action buttons before opening menus,
   dialogs, or editors.
9. Add route-level tests for new pages. Mock `@/queries/*` modules rather than
   mocking ConnectRPC clients directly at route boundaries.
10. Run the repo's documented frontend checks, normally `make test-ui` and any
    lint/build command required by `CLAUDE.md` or `AGENTS.md`.

## Anti-Patterns

- Adding ad-hoc fetch calls or direct ConnectRPC clients in route components.
- Duplicating selected organization/project state in component-local state when
  route params already define the context.
- Reintroducing short route prefixes such as `/orgs/...` when the documented
  convention is full plural nouns.
- Building card-heavy marketing layouts for operational resource management.
- Hand-editing `frontend/src/routeTree.gen.ts`.
- Using raw `<a href>` for in-app route cells; use TanStack Router `Link`.
