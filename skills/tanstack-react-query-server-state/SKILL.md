---
name: tanstack-react-query-server-state
description: Implement, migrate, or review TanStack React Query server state in React applications. Use for server state, serverState, client cache, queryOptions, mutationOptions, query-client cache services, or moving API data out of client stores; not for transport/API design or Vue Query.
---

# TanStack React Query Server State

Use TanStack React Query for server state: the client-side cache and projection of server-owned data. Keep React application state for client-owned truth. These are separate state models.

This skill covers creating/refactoring `serverState`, migrating server data out of client stores, and reviewing boundary violations. It does not design transports, generate clients, or configure Orval. For Vue, use `tanstack-vue-query-server-state` instead.

## Boundary and layout

- The transport layer owns fetch/Axios/WebSocket details, DTOs, API types, serialization, and request implementation.
- `serverState` owns TanStack Query configuration and only the cache operations required by the app.
- Components, feature hooks, route orchestration, and client stores consume the layer. `serverState` must not import those consumers.
- `serverState` never contains components, routing, notifications, forms, UI workflows, business workflows, direct HTTP calls, DTO/API/display/domain type imports, or references to stores/components.

Use a domain folder with `queryOptions.ts`, optional `mutationOptions.ts`, optional `cacheService.ts`, and `index.ts`; re-export domains from `src/serverState/index.ts`.

## Options describe configuration; consumers decide execution

`queryOptions` and `mutationOptions` describe reusable configuration: `queryKey`, `queryFn`, freshness behavior, `mutationFn`, and cache lifecycle. They do not decide where or when a request runs.

Because React hooks may run only in React components or custom hooks, query options are plain domain functions such as `addressesQueryOptions()`. A cache service that uses `useQueryClient()` is a custom hook named `use<Domain>CacheService`; mutation options that consume it are likewise hooks named `use<Domain>MutationOptions`.

The component, feature hook, or route orchestration selects `useQuery`, `useMutation`, a dependent query, prefetching, or an explicit refresh. Read [patterns](references/patterns.md) before implementing them and use the [anti-pattern checklist](references/anti-patterns.md) for reviews.

## Query keys, mutations, and cache services

- Keep key factories private to the domain module. Query keys never escape the layer.
- The first key segment is always the domain parent. Include every response-defining parameter.
- For CRUD resources, nest an item below the list key, for example `['addresses', 'list', 'detail', addressId]`.
- Use `queryOptions()` and `mutationOptions()` so hooks and QueryClient operations share type-safe configuration.
- Specify `staleTime` only when the default conflicts with product freshness requirements.
- Mutation callbacks delegate cache work to a named domain cache-service action, not raw `QueryClient` mechanics.
- Use optimistic cache changes only with cancellation, snapshot, rollback, and reconciliation.

Match the installed TanStack Query major version before using imperative prefetch APIs. The reference shows v5 and its v6 replacement.

## Review and migration

Move API data, manual loading/error flags, TTL, refetching, and request lifecycle out of client stores and into TanStack React Query. Retain only client-owned application state. Flag incomplete keys, exported keys, raw QueryClient usage outside cache services, conditional hook calls, and `serverState` dependencies on UI/application/transport concerns. Prefer the smallest correction.
