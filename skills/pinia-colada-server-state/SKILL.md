---
name: pinia-colada-server-state
description: Implement, migrate, or review Pinia Colada server state in Vue applications. Use for server state, serverState, client cache, query options, mutations, cache services, or moving API data out of Pinia stores; not for transport/API design.
---

# Pinia Colada Server State

Use Pinia Colada as the client-side cache and projection of data whose source of truth is on the server. Keep Pinia for application state: state whose source of truth is the client itself. These are separate state models, not substitutes for each other.

This skill covers creating or refactoring the server-state layer, migrating server data out of Pinia stores, and reviewing violations of that boundary. It does not design a transport/API layer, generate API clients, or configure Orval.

## Required boundary

- The transport layer owns HTTP, fetch, Axios, WebSockets, DTOs, API types, serialization, and request details.
- `serverState` owns Pinia Colada query options, mutation options, and the smallest cache operations needed by the application.
- Components, page composables, and Pinia application stores consume the server-state layer. A Pinia store may use queries or mutations when application state needs them.
- `serverState` must not import or call stores, component composables, routers, notification services, or form helpers. It must not contain UI behavior, routing, form cleanup, business workflows, direct HTTP implementations, DTO types, API types, display types, or domain-model type imports.

Keep the layer thin, declarative, and readable. Do not introduce helpers, abstractions, or cache operations before there is a concrete app need.

## File layout and public API

Use one domain per folder and re-export its public API through its own `index.ts`; re-export domains from `src/serverState/index.ts`.

```text
src/serverState/
  addresses/
    queryOptions.ts
    mutationOptions.ts      # only when mutations exist
    cacheService.ts         # only when a cache operation is needed
    index.ts
  stores/
    queryOptions.ts
    index.ts
  index.ts
```

Name the domain composables `use<Domain>QueryOptions`, `use<Domain>MutationOptions`, and `use<Domain>CacheService`. They expose query/mutation options and meaningful cache actions, not raw query keys or a raw `QueryCache`.

Keep query keys private to their domain module. Consumers obtain keys indirectly by calling the relevant query-options composable and selecting an option. Do not export key constants or a general key factory.

## Options describe configuration; consumers decide execution

`queryOptions` and `mutationOptions` describe reusable Pinia Colada configuration: the resource key, request, freshness behavior, mutation, and cache lifecycle. They do not decide where or when a request runs.

The consuming page, component composable, Pinia application store, or route-level orchestration selects the appropriate use: `useQuery`, `useMutation`, a dependent query, prefetching, or an explicit refresh. Do not call `useQuery` or `useMutation` inside `serverState` merely to hide that decision. A Pinia store may consume these options, but `serverState` must never depend on that store.

Read [the implementation patterns](references/patterns.md) before adding a domain, a mutation, cache logic, an optimistic update, query consumption, or prefetching. For migrations and reviews, use the [anti-pattern checklist](references/anti-patterns.md).

## Query-key rules

- Construct keys through private factories in the domain module.
- The first segment is always the domain's parent key, enabling whole-domain invalidation.
- Include every parameter that changes a response, using the names and values meaningful to the domain.
- For CRUD resources, place item keys beneath the list key, so invalidating the list naturally covers items: `['addresses', 'list', 'detail', addressId]`.
- Use Pinia Colada's `defineQueryOptions` to bind a key to the request. A key never escapes the server-state layer.

Set `staleTime` only when the default does not match the resource's freshness requirement. When setting it, choose and document it based on the product behavior, not a generic number.

## Mutations and cache services

Put post-mutation cache behavior behind a cache service. A mutation's `onSuccess` calls a named cache-service method; it does not manipulate `useQueryCache()` directly. Add only cache-service methods that the UI actually requires, for example `refreshAddresses()` rather than a generic invalidation API.

Optimistic updates are an explicit case, not a default. Use them when immediate UI feedback is required and a correct rollback is feasible. Keep the snapshot, optimistic write, rollback on error, and final reconciliation/invalidation contained in the server-state layer through the cache service. If those guarantees are not clear, use the simpler success invalidation path.

## Implementation, migration, and review

For a new feature, first identify whether each value is application state or server state. Put server resources in a domain of `src/serverState`; leave client-owned choices, transient UI state, and application coordination in a Pinia store or component.

For a migration, move API data, loading/error flags, request lifecycle, cache lifetime, and refetch behavior from the Pinia store to Pinia Colada. Retain only client-owned state in the store. Then update consumers to use the query/mutation options, preserving existing user-visible behavior.

During a code review, flag and correct these violations:

- server API data, manual loading/error flags, or cache TTLs maintained in a Pinia store;
- a query key without the domain parent or response-defining parameters;
- exported query keys, direct cache access from a mutation, or cache operations that lack a concrete need;
- `serverState` imports of stores, components, router, notifications, forms, transport implementations, DTO/API/display/domain types;
- direct fetch/Axios calls inside `serverState`;
- notification, routing, form-reset, component, or unrelated business-flow logic inside the server-state layer.

Prefer the smallest correct correction. Do not turn a focused migration or review into a transport redesign or a generic cache framework.
