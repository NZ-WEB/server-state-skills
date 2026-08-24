# TanStack React Query anti-pattern checklist

Do not approve a change containing these patterns.

## Manual server cache in a client store

```ts
// Bad: API data and lifecycle are reimplemented manually.
const [addresses, setAddresses] = useState([])
const [isLoading, setIsLoading] = useState(false)
```

Use domain query options with `useQuery()`. Keep only client-owned application state in React state or a client store.

## Executing a query inside the server-state layer

```ts
// Bad: reusable configuration is coupled to one hook execution site.
export function useAddresses() {
  return useQuery({ queryKey: ['addresses'], queryFn: addressesApi.list })
}
```

Export plain `addressesQueryOptions` and let a React component or feature hook call `useQuery()`.

## Calling hooks conditionally or from a plain options function

```ts
// Bad: violates React's Rules of Hooks.
export function addressesQueryOptions() {
  const queryClient = useQueryClient()
  if (enabled)
    return useQuery({ queryKey: ['addresses'], queryFn: addressesApi.list })
}
```

Keep query options hook-free. A cache service may call `useQueryClient()`, but only as a custom hook invoked by a component or another custom hook.

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
  toast.success('Saved')
  navigate('/addresses')
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
- Are React hooks called only at a component/custom-hook top level?
- Are keys private, domain-first, and complete for response parameters?
- Does a cache service own necessary cache changes behind domain names?
- Is prefetching chosen by a component, feature hook, or route orchestration?
- Is serverState independent of components, routing, notifications, forms, client stores, and transport implementation?
