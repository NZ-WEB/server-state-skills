# Server State Skills

Reusable agent skills for implementing, migrating, and reviewing client-side server-state layers in Vue and React applications.

The repository follows the [`skills`](https://github.com/vercel-labs/skills) discovery convention: every installable skill lives below [`skills/`](skills/).

## Install

List the skills in this repository:

```bash
npx skills add NZ-WEB/server-state-skills --list
```

Install one skill:

```bash
npx skills add NZ-WEB/server-state-skills --skill pinia-colada-server-state
```

Install all skills:

```bash
npx skills add NZ-WEB/server-state-skills --all
```

The `skills` CLI installs into the appropriate project or global directory for the selected coding agent. Use its `--agent`, `--global`, and `--copy` options when needed.

## Skills

### Pinia Colada Server State

[`pinia-colada-server-state`](skills/pinia-colada-server-state/) is for Vue applications using [Pinia Colada](https://pinia-colada.esm.dev/). It helps an agent:

- create or refactor a domain-oriented `serverState` layer;
- migrate API resources, request lifecycle, and cache behavior out of Pinia stores;
- keep Pinia limited to client-owned application state;
- define private domain-first query keys, query options, mutation options, and minimal cache services;
- review boundary violations, cache-key mistakes, and unsafe optimistic updates.

The skill treats query and mutation options as reusable configuration. Components, page composables, and application stores decide whether and when to call `useQuery`, `useMutation`, dependent queries, prefetching, or explicit refreshes.

### TanStack Vue Query Server State

[`tanstack-vue-query-server-state`](skills/tanstack-vue-query-server-state/) is the Vue-specific TanStack Query skill. It covers:

- `queryOptions()` and `mutationOptions()` as type-safe reusable configuration;
- Vue composables that expose domain options and cache services;
- `useQuery` consumption with reactive and dependent parameters;
- cache warming through the Query Client;
- migration from manual Pinia server caches and review of Vue-specific anti-patterns.

The skill includes both TanStack Query v5-style prefetch examples and a reminder to verify the installed major before using an imperative Query Client API.

### TanStack React Query Server State

[`tanstack-react-query-server-state`](skills/tanstack-react-query-server-state/) is the React-specific TanStack Query skill. It covers:

- hook-free domain query-options factories;
- custom hooks only where `useQueryClient()` is required;
- `useQuery`, enabled/dependent queries, and event-driven prefetching;
- mutation cache lifecycle and optimistic rollback;
- server-state migrations and Rules of Hooks violations during review.

It explicitly separates reusable configuration from hook execution: components, feature hooks, and route orchestration decide when a query or mutation runs.

## Shared architecture

All three skills use the same server-state boundary:

```text
transport layer
    ↓
serverState domains
    ↓
components, feature/page orchestration, and client application state
```

The transport layer owns request implementation, DTOs, API types, and serialization. The server-state layer owns query/mutation configuration and only the cache operations the application requires. User-interface behavior, routing, notifications, form cleanup, and client-owned state remain outside the server-state layer.

## Repository layout

```text
skills/
  pinia-colada-server-state/
  tanstack-vue-query-server-state/
  tanstack-react-query-server-state/
```

Each skill contains a `SKILL.md` entrypoint, optional `agents/openai.yaml` metadata, and focused references with implementation patterns and anti-pattern checklists.

## License

[MIT](LICENSE)
