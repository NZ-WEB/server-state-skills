# Pinia Colada patterns

Use these patterns as a starting point. Adapt names and cache behavior to the domain; do not add pieces that the application does not need.

## Query options

```ts
import { defineQueryOptions } from '@pinia/colada'
import { useAddressesApi } from '@/http-transport/addresses'

const queryKeys = {
  all: () => ['addresses'] as const,
  list: () => [...queryKeys.all(), 'list'] as const,
  detail: (addressId: string) => [...queryKeys.list(), 'detail', addressId] as const,
}

export function useAddressesQueryOptions() {
  const addressesApi = useAddressesApi()

  const list = defineQueryOptions({
    key: queryKeys.list(),
    query: () => addressesApi.list(),
  })

  const detail = defineQueryOptions((addressId: string) => ({
    key: queryKeys.detail(addressId),
    query: () => addressesApi.get(addressId),
  }))

  return { detail, list }
}
```

The `queryKeys` object is intentionally module-private. No type is imported into this module merely to describe a transport DTO, a domain model, or a component's view model; transport function signatures supply the necessary types.

## Cache service and mutation options

```ts
import { useQueryCache } from '@pinia/colada'
import { useAddressesQueryOptions } from './queryOptions'

export function useAddressesCacheService() {
  const queryCache = useQueryCache()
  const { list } = useAddressesQueryOptions()

  function refreshAddresses() {
    return queryCache.invalidateQueries({ key: list.key }, 'all')
  }

  return { refreshAddresses }
}
```

```ts
import { defineMutationOptions } from '@pinia/colada'
import { useAddressesApi } from '@/http-transport/addresses'
import { useAddressesCacheService } from './cacheService'

export function useAddressesMutationOptions() {
  const addressesApi = useAddressesApi()
  const { refreshAddresses } = useAddressesCacheService()

  const create = defineMutationOptions({
    mutation: addressesApi.create,
    onSuccess: () => refreshAddresses(),
  })

  return { create }
}
```

The cache service exposes the domain action. Its list invalidation intentionally also matches child detail keys. It deliberately does not expose `queryCache`, generic keys, or a generic `invalidate` method.

## Consumption outside the layer

Query options are configuration, not an instruction to execute a request at a fixed place. The consuming context decides whether it needs a reactive query, an action mutation, or a warm cache for a likely next screen.

```ts
// A component, page composable, or Pinia application store.
import { useQuery } from '@pinia/colada'
import { useAddressesQueryOptions } from '@/serverState'

const { list } = useAddressesQueryOptions()
const addressesQuery = useQuery(list)
```

A Pinia store can consume those options to coordinate client-owned application state. The server-state modules themselves remain independent of the store.

## Dependent query at the consumer

```ts
import { useQuery } from '@pinia/colada'
import { computed } from 'vue'
import { useCatalogQueryOptions } from '@/serverState'

const catalogQueryOptions = useCatalogQueryOptions()
const selectedStoreId = computed(() => activeStore.value?.id ?? null)

const catalogQuery = useQuery(() => ({
  ...catalogQueryOptions.list({ storeId: selectedStoreId.value! }),
  enabled: selectedStoreId.value !== null,
}))
```

The domain options retain the key and request definition. The consumer contributes only local orchestration: this query is enabled when its selected store exists.

## Prefetch with the same options

Use the same options to warm a cache for a likely next navigation. This belongs to a route-level or feature-level consumer, not the `serverState` domain.

```ts
import { useQueryCache } from '@pinia/colada'
import { useCatalogQueryOptions } from '@/serverState'

const queryCache = useQueryCache()
const catalogQueryOptions = useCatalogQueryOptions()

async function prefetchCatalog(storeId: string) {
  const options = catalogQueryOptions.list({ storeId })
  const entry = queryCache.ensure(options)

  await queryCache.refresh(entry, options)
}
```

`ensure()` reuses or creates the entry described by the options, and `refresh()` fetches only when its data is stale. A later `useQuery(options)` reads that same cache entry. Use `queryCache.fetch(entry, options)` only when the product explicitly requires a network request even for fresh data.

## Optimistic update decision

Before adding an optimistic update, establish the exact cached resources affected and the rollback data required. The implementation needs all four operations: snapshot/cache read, optimistic write, rollback on error, and final reconciliation. Keep those cache-specific operations private behind the domain cache service. If any step cannot be correct for the feature, do not add an optimistic update; invalidate after success instead.

## Optimistic removal

This example uses the demo's existing address removal flow. The cache service owns cancellation, the snapshot, the optimistic cache write, the rollback, and reconciliation. The component only invokes the mutation.

```ts
// cacheService.ts
import { useQueryCache } from '@pinia/colada'
import { useAddressesQueryOptions } from './queryOptions'

export function useAddressesCacheService() {
  const queryCache = useQueryCache()
  const { list } = useAddressesQueryOptions()

  function refreshAddresses() {
    return queryCache.invalidateQueries({ key: list.key }, 'all')
  }

  function optimisticallyRemoveAddress(addressId: string) {
    queryCache.cancelQueries({ key: list.key })

    const previousAddresses = queryCache.getQueryData(list.key)

    queryCache.setQueryData(list.key, addresses =>
      addresses?.filter(address => address.id !== addressId) ?? [],
    )

    return { previousAddresses }
  }

  function restoreAddresses(snapshot: ReturnType<typeof optimisticallyRemoveAddress>) {
    if (snapshot.previousAddresses)
      queryCache.setQueryData(list.key, snapshot.previousAddresses)
  }

  return { optimisticallyRemoveAddress, refreshAddresses, restoreAddresses }
}
```

```ts
// mutationOptions.ts
import { defineMutationOptions } from '@pinia/colada'
import { useAddressesApi } from '@/http-transport/addresses'
import { useAddressesCacheService } from './cacheService'

export function useAddressesMutationOptions() {
  const addressesApi = useAddressesApi()
  const addressesCache = useAddressesCacheService()

  const remove = defineMutationOptions({
    onMutate: addressId => addressesCache.optimisticallyRemoveAddress(addressId),
    mutation: addressesApi.remove,
    onError: (_error, _addressId, snapshot) => {
      if (snapshot)
        addressesCache.restoreAddresses(snapshot)
    },
    onSettled: () => addressesCache.refreshAddresses(),
  })

  return { remove }
}
```

`onMutate` appears before `mutation` so Pinia Colada can infer the rollback context. `onSettled` refetches after both success and failure: a successful request reconciles server-calculated data, while a failed request confirms the restored cache. Keep only domain-specific cache actions public; do not expose the snapshot or `QueryCache` to components.
