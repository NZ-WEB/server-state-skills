# TanStack Vue Query patterns

These examples use TanStack Query v5-style imperative prefetching. Inspect the installed version before copying that API: v6 replaces `prefetchQuery()` with `queryClient.query()`.

## Domain query options

```ts
import { queryOptions } from '@tanstack/vue-query'
import { useAddressesApi } from '@/http-transport/addresses'

const queryKeys = {
  all: () => ['addresses'] as const,
  list: () => [...queryKeys.all(), 'list'] as const,
  detail: (addressId: string) => [...queryKeys.list(), 'detail', addressId] as const,
}

export function useAddressesQueryOptions() {
  const addressesApi = useAddressesApi()

  const list = () => queryOptions({
    queryKey: queryKeys.list(),
    queryFn: () => addressesApi.list(),
  })

  const detail = (addressId: string) => queryOptions({
    queryKey: queryKeys.detail(addressId),
    queryFn: () => addressesApi.get(addressId),
  })

  return { detail, list }
}
```

The key factory remains private. The domain composable exposes options, not `useQuery()` results.

## Vue consumer: useQuery, dependent query, and prefetch

```ts
import { useQuery, useQueryClient } from '@tanstack/vue-query'
import { computed } from 'vue'
import { useCatalogQueryOptions } from '@/serverState'

const catalogOptions = useCatalogQueryOptions()
const selectedStoreId = computed(() => activeStore.value?.id ?? null)

const catalogQuery = useQuery(() => ({
  ...catalogOptions.list(selectedStoreId.value!),
  enabled: selectedStoreId.value !== null,
}))

const queryClient = useQueryClient()

async function prefetchCatalog(storeId: string) {
  await queryClient.prefetchQuery(catalogOptions.list(storeId))
  // TanStack Query v6: await queryClient.query(catalogOptions.list(storeId))
}
```

The consumer owns reactive Vue state and the decision to prefetch. A later `useQuery(catalogOptions.list(storeId))` reads the same cache entry.

## Cache service and mutation options

```ts
// cacheService.ts
import { useQueryClient } from '@tanstack/vue-query'
import { useAddressesQueryOptions } from './queryOptions'

export function useAddressesCacheService() {
  const queryClient = useQueryClient()
  const { list } = useAddressesQueryOptions()

  function refreshAddresses() {
    return queryClient.invalidateQueries({ queryKey: list().queryKey })
  }

  return { refreshAddresses }
}
```

```ts
// mutationOptions.ts
import { mutationOptions } from '@tanstack/vue-query'
import { useAddressesApi } from '@/http-transport/addresses'
import { useAddressesCacheService } from './cacheService'

export function useAddressesMutationOptions() {
  const addressesApi = useAddressesApi()
  const { refreshAddresses } = useAddressesCacheService()

  const create = () => mutationOptions({
    mutationFn: addressesApi.create,
    onSuccess: () => refreshAddresses(),
  })

  return { create }
}
```

## Optimistic removal

```ts
// cacheService.ts: add these domain-specific operations
async function optimisticallyRemoveAddress(addressId: string) {
  const options = list()
  await queryClient.cancelQueries({ queryKey: options.queryKey })
  const previousAddresses = queryClient.getQueryData(options.queryKey)

  queryClient.setQueryData(options.queryKey, addresses =>
    addresses?.filter(address => address.id !== addressId) ?? [],
  )

  return { previousAddresses }
}

function restoreAddresses(snapshot: Awaited<ReturnType<typeof optimisticallyRemoveAddress>>) {
  if (snapshot.previousAddresses)
    queryClient.setQueryData(list().queryKey, snapshot.previousAddresses)
}
```

```ts
// mutationOptions.ts
const remove = () => mutationOptions({
  mutationFn: addressesApi.remove,
  onMutate: addressId => addressesCache.optimisticallyRemoveAddress(addressId),
  onError: (_error, _addressId, snapshot) => {
    if (snapshot)
      addressesCache.restoreAddresses(snapshot)
  },
  onSettled: () => addressesCache.refreshAddresses(),
})
```

The cache service owns cancellation, snapshot, optimistic write, rollback, and reconciliation. Components invoke the mutation and handle UI-only effects.
