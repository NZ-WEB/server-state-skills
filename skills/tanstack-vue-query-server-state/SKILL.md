---
name: tanstack-vue-query-server-state
description: Implement, migrate, or review TanStack Vue Query server state in Vue applications. Use for server state, serverState, client cache, queryOptions, mutationOptions, query-client cache services, or moving API data out of Pinia stores; not for transport/API design or React Query.
---

# TanStack Vue Query Server State

Use TanStack Vue Query for server state: the client-side cache and projection of server-owned data. Keep Pinia for application state whose source of truth is the client. These are separate state models.

This skill covers creating/refactoring `serverState`, migrating server data out of Pinia stores, and reviewing boundary violations. It does not design transports, generate clients, or configure Orval. For React, use `tanstack-react-query-server-state` instead.

## Boundary and layout

- The transport layer owns fetch/Axios/WebSocket details, DTOs, API types, serialization, and request implementation.
- `serverState` owns TanStack Query configuration and only the cache operations required by the app.
- Components, page composables, route orchestration, and Pinia application stores consume the layer. A Pinia store may consume query/mutation options; `serverState` never imports that store.
- `serverState` never contains components, routing, notifications, forms, UI workflows, business workflows, direct HTTP calls, DTO/API/display/domain type imports, or references to stores/component composables.

Use one domain per folder and re-export it through its own `index.ts`; re-export domains from `src/serverState/index.ts`.

```text
src/serverState/
  addresses/
    queryOptions.ts
    mutationOptions.ts      # only when mutations exist
    cacheService.ts         # only when cache behavior is needed
    index.ts
  index.ts
```

## Options describe configuration; consumers decide execution

`queryOptions` and `mutationOptions` describe reusable configuration: `queryKey`, `queryFn`, freshness behavior, `mutationFn`, and cache lifecycle. They do not decide where or when a request runs.

The page, component composable, route orchestration, or Pinia store selects `useQuery`, `useMutation`, a dependent query, prefetching, or an explicit refresh. Do not call `useQuery` or `useMutation` inside `serverState` merely to hide that decision.

In Vue, make options through domain composables named `use<Domain>QueryOptions` and `use<Domain>MutationOptions`. `use<Domain>CacheService` exposes named cache actions, not a raw `QueryClient`. Read [patterns](references/patterns.md) before implementing queries, mutations, dependent queries, prefetching, or optimistic updates; use the [anti-pattern checklist](references/anti-patterns.md) for reviews.

## Query keys, mutations, and cache services

- Keep key factories private to the domain module. Query keys never escape the layer.
- The first key segment is always the domain parent. Include every response-defining parameter.
- For CRUD resources, nest an item beneath the list key, for example `['addresses', 'list', 'detail', addressId]`, so list invalidation matches its child keys.
- Use `queryOptions()` and `mutationOptions()` to preserve type inference across hooks and QueryClient APIs.
- Specify `staleTime` only when the default conflicts with product freshness requirements; explain a non-default value in product terms.
- A mutation callback delegates cache work to `use<Domain>CacheService()`. Add only concrete actions such as `refreshAddresses()` or `optimisticallyRemoveAddress()`.
- Use optimistic cache changes only when immediate feedback is needed and cancellation, snapshot, rollback, and reconciliation can all be correct.

Match the installed TanStack Query major version before choosing an imperative prefetch API. The reference shows the v5 API and points out the v6 replacement.

## Review and migration

Move API data, manual loading/error flags, TTL, refetching, and request lifecycle from Pinia stores to TanStack Vue Query. Retain only client-owned application state in Pinia. Flag incomplete keys, exported key factories, raw QueryClient access outside a domain cache service, and any serverState dependency on UI/application/transport concerns. Prefer the smallest correction; do not turn a focused migration into a transport redesign or generic cache framework.
