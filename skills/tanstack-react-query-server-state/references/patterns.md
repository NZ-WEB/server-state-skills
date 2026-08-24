# TanStack React Query patterns

These examples use TanStack Query v5-style imperative prefetching. Inspect the installed version before copying that API: v6 replaces `prefetchQuery()` with `queryClient.query()`.

## Domain query options

```ts
import { queryOptions } from '@tanstack/react-query'
import { addressesApi } from '@/http-transport/addresses'

const queryKeys = {
  all: () => ['addresses'] as const,
  list: () => [...queryKeys.all(), 'list'] as const,
  detail: (addressId: string) => [...queryKeys.list(), 'detail', addressId] as const,
}

export const addressesQueryOptions = {
  list: () => queryOptions({
    queryKey: queryKeys.list(),
    queryFn: addressesApi.list,
  }),
  detail: (addressId: string) => queryOptions({
    queryKey: queryKeys.detail(addressId),
    queryFn: () => addressesApi.get(addressId),
  }),
}
```

React query options are plain functions: they do not call React hooks. The key factory remains private.

## React consumer: useQuery, dependent query, and prefetch

```tsx
import { useQuery, useQueryClient } from '@tanstack/react-query'
import { addressesQueryOptions } from '@/serverState'

function AddressDetails({ addressId }: { addressId: string | null }) {
  const addressQuery = useQuery({
    ...addressesQueryOptions.detail(addressId ?? ''),
    enabled: addressId !== null,
  })

  return <AddressCard address={addressQuery.data} />
}

function AddressLink({ addressId }: { addressId: string }) {
  const queryClient = useQueryClient()

  function prefetchAddress() {
    void queryClient.prefetchQuery(addressesQueryOptions.detail(addressId))
    // TanStack Query v6: void queryClient.query(addressesQueryOptions.detail(addressId))
  }

  return <button onFocus={prefetchAddress} onMouseEnter={prefetchAddress}>Open address</button>
}
```

Hooks execute at the component top level. The consumer chooses its local `enabled` condition and prefetch timing; a later `useQuery()` reads the same cache entry.

## Cache service and mutation options

```ts
// cacheService.ts
import { useQueryClient } from '@tanstack/react-query'
import { addressesQueryOptions } from './queryOptions'

export function useAddressesCacheService() {
  const queryClient = useQueryClient()

  function refreshAddresses() {
    return queryClient.invalidateQueries({
      queryKey: addressesQueryOptions.list().queryKey,
    })
  }

  return { refreshAddresses }
}
```

```ts
// mutationOptions.ts
import { mutationOptions } from '@tanstack/react-query'
import { addressesApi } from '@/http-transport/addresses'
import { useAddressesCacheService } from './cacheService'

export function useAddressesMutationOptions() {
  const { refreshAddresses } = useAddressesCacheService()

  const create = () => mutationOptions({
    mutationFn: addressesApi.create,
    onSuccess: () => refreshAddresses(),
  })

  return { create }
}
```

Call the resulting `useAddressesMutationOptions()` only from a React component or a custom hook, then pass `create()` into `useMutation()`.

## Optimistic removal

```ts
// cacheService.ts: add these domain-specific operations
async function optimisticallyRemoveAddress(addressId: string) {
  const options = addressesQueryOptions.list()
  await queryClient.cancelQueries({ queryKey: options.queryKey })
  const previousAddresses = queryClient.getQueryData(options.queryKey)

  queryClient.setQueryData(options.queryKey, addresses =>
    addresses?.filter(address => address.id !== addressId) ?? [],
  )

  return { previousAddresses }
}

function restoreAddresses(snapshot: Awaited<ReturnType<typeof optimisticallyRemoveAddress>>) {
  if (snapshot.previousAddresses)
    queryClient.setQueryData(addressesQueryOptions.list().queryKey, snapshot.previousAddresses)
}
```

```ts
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

The cache service owns cache correctness. The component invokes the mutation and handles UI-specific effects.
