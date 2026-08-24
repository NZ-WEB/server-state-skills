# TanStack Vue Query anti-pattern checklist

Do not approve a change containing these patterns.

## Manual server cache in Pinia

```ts
// Bad: addresses, loading, and errors duplicate Query's lifecycle.
const addresses = shallowRef([])
const isLoading = ref(false)
async function loadAddresses() { /* call API and manage flags */ }
```

Use domain query options with `useQuery()`. Keep only client-owned application state in Pinia.

## Executing Vue queries inside serverState

```ts
// Bad: reusable configuration is coupled to one execution site.
export function useAddresses() {
  return useQuery({ queryKey: ['addresses'], queryFn: () => api.list() })
}
```

Export `useAddressesQueryOptions()` and let a consumer choose `useQuery`, prefetch, or another valid use.

## Incomplete/exported keys or raw QueryClient access

```ts
// Bad: storeId is missing and cache mechanics leak out of the domain.
export const catalogKey = ['catalog']
queryClient.invalidateQueries({ queryKey: catalogKey })
```

Use private domain key factories and named cache-service methods.

## UI, routing, or transport logic in serverState

```ts
// Bad: unrelated feature behavior inside cache lifecycle.
onSuccess: () => {
  toast.add({ title: 'Saved' })
  router.push('/addresses')
  return fetch('/api/addresses')
}
```

ServerState calls the existing transport adapter and maintains cache correctness only.

## Optimistic write without rollback

```ts
// Bad: an error can leave the cached list incorrect.
onMutate: addressId => queryClient.setQueryData(['addresses'], removeAddress(addressId))
```

Require cancellation, snapshot, rollback in `onError`, and reconciliation in `onSettled`.

## Final checklist

- Are query/mutation options configuration rather than hook execution?
- Are keys private, domain-first, and complete for response parameters?
- Does a cache service own necessary cache changes behind domain names?
- Is Vue reactivity and prefetching chosen by a consumer, not the server-state layer?
- Is serverState independent of Pinia stores, components, router, notifications, forms, and transport implementation?
