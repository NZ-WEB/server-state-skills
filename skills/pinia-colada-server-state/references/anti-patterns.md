# Anti-pattern checklist

Use this checklist when implementing, migrating, or reviewing Pinia Colada server state. Each example is intentionally wrong.

## Do not put server state in a Pinia store

```ts
// Bad: cache, request lifecycle, and error state are reimplemented manually.
export const useAddressesStore = defineStore('addresses', () => {
  const addresses = shallowRef([])
  const isLoading = ref(false)
  const error = shallowRef<Error | null>(null)

  async function load() {
    isLoading.value = true
    try {
      addresses.value = await useAddressesApi().list()
    }
    catch (cause) {
      error.value = cause as Error
    }
    finally {
      isLoading.value = false
    }
  }

  return { addresses, error, isLoading, load }
})
```

Use `useAddressesQueryOptions()` with `useQuery()` instead. Keep only client-owned application state, such as `activeAddressId`, in Pinia.

## Do not execute a query inside the server-state layer

```ts
// Bad: this fixes one consumer's lifecycle choice inside the reusable layer.
export function useAddresses() {
  const api = useAddressesApi()
  return useQuery({
    key: ['addresses'],
    query: () => api.list(),
  })
}
```

Export `useAddressesQueryOptions()` from `serverState`; call `useQuery()` at the component, store, or orchestration boundary that owns when the query is used.

## Do not use incomplete or exported keys

```ts
// Bad: different stores overwrite one another; the key escapes the domain.
export const catalogKey = ['catalog']

const list = defineQueryOptions(({ storeId }) => ({
  key: catalogKey,
  query: () => catalogApi.list(storeId),
}))
```

Use private key factories. A query key starts with its domain and contains every response-defining parameter, for example `['catalog', 'list', storeId]`.

## Do not bypass the cache service in mutation callbacks

```ts
// Bad: mutation options expose cache mechanics instead of domain behavior.
const remove = defineMutationOptions({
  mutation: addressesApi.remove,
  onSuccess: () => queryCache.invalidateQueries({ key: ['addresses'] }),
})
```

Keep `queryCache` private to `useAddressesCacheService()` and call a meaningful operation such as `refreshAddresses()` from the mutation callback.

## Do not put UI and application workflows in serverState

```ts
// Bad: serverState now depends on UI and routing concerns.
onSuccess: () => {
  toast.add({ title: 'Address removed' })
  router.push('/addresses')
  form.reset()
}
```

The consuming feature handles notifications, navigation, and form cleanup. Server-state mutation callbacks only maintain cache correctness.

## Do not implement the transport layer again

```ts
// Bad: serverState owns a request implementation and a DTO type.
interface AddressDto { id: string }

const list = defineQueryOptions({
  key: ['addresses', 'list'],
  query: () => fetch('/api/addresses').then(response => response.json() as Promise<AddressDto[]>),
})
```

Call the existing transport adapter instead. Keep fetch/Axios, DTOs, API types, and serialization below `serverState`.

## Do not add optimistic updates without a rollback plan

```ts
// Bad: the UI can remain wrong when the mutation fails.
onMutate: addressId => {
  queryCache.setQueryData(list.key, addresses =>
    addresses?.filter(address => address.id !== addressId) ?? [],
  )
},
mutation: addressesApi.remove,
```

Before using optimistic updates, implement cancellation, snapshot, optimistic write, rollback in `onError`, and final reconciliation. See [Optimistic removal](patterns.md#optimistic-removal).

## Final review checklist

Do not approve the change until all answers are yes:

- Is each API resource represented by query options rather than manual Pinia state?
- Are keys private, domain-first, and complete for their response parameters?
- Does `serverState` use a transport adapter without importing UI, routing, stores, DTOs, or display/domain types?
- Does each mutation delegate cache work to its domain cache service?
- Does an optimistic mutation have snapshot, rollback, and reconciliation?
- Does the consumer, rather than `serverState`, choose query execution, mutation invocation, prefetching, navigation, notifications, and form behavior?
