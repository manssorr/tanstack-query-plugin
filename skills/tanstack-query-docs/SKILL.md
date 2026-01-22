---
name: tanstack-query-docs
description: |
  Official TanStack Query (React Query) v5 API reference documentation.
  Use for: useQuery, useMutation, useInfiniteQuery, QueryClient methods,
  configuration options, TypeScript types, and API signatures.
version: 5.0.0
license: MIT
---

# TanStack Query v5 API Reference

Official API documentation for TanStack Query (React Query) v5.

---

## Setup

### QueryClientProvider

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,  // 5 minutes
      gcTime: 1000 * 60 * 30,    // 30 minutes
      retry: 3,
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
    },
  },
})

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <YourApp />
    </QueryClientProvider>
  )
}
```

---

## useQuery

### Signature

```tsx
const {
  data,
  dataUpdatedAt,
  error,
  errorUpdatedAt,
  failureCount,
  failureReason,
  fetchStatus,
  isError,
  isFetched,
  isFetchedAfterMount,
  isFetching,
  isLoading,
  isPaused,
  isPending,
  isPlaceholderData,
  isRefetchError,
  isRefetching,
  isStale,
  isSuccess,
  promise,
  refetch,
  status,
} = useQuery({
  queryKey,
  queryFn,
  // Options below
})
```

### Core Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `queryKey` | `unknown[]` | Required | Unique key for this query |
| `queryFn` | `(context) => Promise<TData>` | Required | Function that fetches data |
| `enabled` | `boolean` | `true` | Set `false` to disable automatic fetching |
| `staleTime` | `number` | `0` | Time in ms data is considered fresh |
| `gcTime` | `number` | `5 * 60 * 1000` | Time unused data stays in cache |
| `retry` | `boolean \| number \| fn` | `3` | Retry failed queries |
| `retryDelay` | `number \| fn` | Exponential | Delay between retries |

### Data Options

| Option | Type | Description |
|--------|------|-------------|
| `select` | `(data) => TSelected` | Transform data (doesn't affect cache) |
| `placeholderData` | `TData \| fn` | Placeholder while pending (not cached) |
| `initialData` | `TData \| fn` | Initial data (cached, considered stale) |
| `initialDataUpdatedAt` | `number` | Timestamp of initialData |

### Refetch Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `refetchInterval` | `number \| false \| fn` | `false` | Auto-refetch interval |
| `refetchOnMount` | `boolean \| 'always'` | `true` | Refetch when component mounts |
| `refetchOnWindowFocus` | `boolean \| 'always'` | `true` | Refetch on window focus |
| `refetchOnReconnect` | `boolean \| 'always'` | `true` | Refetch on network reconnect |

### Network Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `networkMode` | `'online' \| 'always' \| 'offlineFirst'` | `'online'` | Network behavior |

### Return Values

| Property | Type | Description |
|----------|------|-------------|
| `data` | `TData \| undefined` | The fetched data |
| `error` | `TError \| null` | Error if query failed |
| `status` | `'pending' \| 'success' \| 'error'` | Query status |
| `fetchStatus` | `'fetching' \| 'paused' \| 'idle'` | Network request status |
| `isPending` | `boolean` | `status === 'pending'` |
| `isSuccess` | `boolean` | `status === 'success'` |
| `isError` | `boolean` | `status === 'error'` |
| `isFetching` | `boolean` | `fetchStatus === 'fetching'` |
| `isPaused` | `boolean` | `fetchStatus === 'paused'` |
| `isLoading` | `boolean` | `isPending && isFetching` |
| `isRefetching` | `boolean` | `isFetching && !isPending` |
| `refetch` | `() => Promise` | Manually trigger refetch |

### Example

```tsx
const { data, isPending, error } = useQuery({
  queryKey: ['todos', filters],
  queryFn: () => fetchTodos(filters),
  staleTime: 1000 * 60 * 5,
  select: (data) => data.filter(todo => !todo.completed),
})
```

---

## useMutation

### Signature

```tsx
const {
  data,
  error,
  isError,
  isIdle,
  isPending,
  isPaused,
  isSuccess,
  failureCount,
  failureReason,
  mutate,
  mutateAsync,
  reset,
  status,
  variables,
} = useMutation({
  mutationFn,
  // Options below
})
```

### Options

| Option | Type | Description |
|--------|------|-------------|
| `mutationFn` | `(variables) => Promise<TData>` | Function that performs mutation |
| `mutationKey` | `unknown[]` | Optional key for the mutation |
| `onMutate` | `(variables, context) => Promise<TContext>` | Called before mutation |
| `onSuccess` | `(data, variables, onMutateResult, context) => void` | Called on success |
| `onError` | `(error, variables, onMutateResult, context) => void` | Called on error |
| `onSettled` | `(data, error, variables, onMutateResult, context) => void` | Called on settle |
| `retry` | `boolean \| number \| fn` | Retry failed mutations |
| `gcTime` | `number` | Cache time for mutation result |
| `networkMode` | `'online' \| 'always' \| 'offlineFirst'` | Network behavior |

### Return Values

| Property | Type | Description |
|----------|------|-------------|
| `mutate` | `(variables, options?) => void` | Trigger mutation |
| `mutateAsync` | `(variables, options?) => Promise<TData>` | Trigger mutation (returns Promise) |
| `data` | `TData \| undefined` | Mutation result data |
| `error` | `TError \| null` | Error if mutation failed |
| `isPending` | `boolean` | Mutation in progress |
| `isSuccess` | `boolean` | Mutation succeeded |
| `isError` | `boolean` | Mutation failed |
| `isIdle` | `boolean` | Mutation not started |
| `variables` | `TVariables` | Variables passed to mutate |
| `reset` | `() => void` | Reset mutation state |

### Lifecycle Callbacks

```tsx
useMutation({
  mutationFn: addTodo,
  onMutate: async (variables, context) => {
    // Called before mutation executes
    // Return value available as onMutateResult
    return { backup: 'data' }
  },
  onError: (error, variables, onMutateResult, context) => {
    // Called if mutation fails
    // onMutateResult is what onMutate returned
  },
  onSuccess: (data, variables, onMutateResult, context) => {
    // Called if mutation succeeds
  },
  onSettled: (data, error, variables, onMutateResult, context) => {
    // Called regardless of success/error
  },
})
```

### Per-Call Callbacks

```tsx
mutate(todo, {
  onSuccess: (data, variables, onMutateResult, context) => {
    // Fires AFTER hook-level onSuccess
  },
  onError: (error, variables, onMutateResult, context) => {
    // Fires AFTER hook-level onError
  },
  onSettled: (data, error, variables, onMutateResult, context) => {
    // Fires AFTER hook-level onSettled
  },
})
```

---

## useInfiniteQuery

### Signature

```tsx
const {
  data,
  fetchNextPage,
  fetchPreviousPage,
  hasNextPage,
  hasPreviousPage,
  isFetchingNextPage,
  isFetchingPreviousPage,
  ...queryResult
} = useInfiniteQuery({
  queryKey,
  queryFn: ({ pageParam }) => fetchPage(pageParam),
  initialPageParam,
  getNextPageParam,
  getPreviousPageParam,
})
```

### Required Options

| Option | Type | Description |
|--------|------|-------------|
| `queryKey` | `unknown[]` | Unique key for this query |
| `queryFn` | `({ pageParam }) => Promise<TData>` | Fetch function receiving pageParam |
| `initialPageParam` | `TPageParam` | First page parameter |
| `getNextPageParam` | `(lastPage, allPages, lastPageParam, allPageParams) => TPageParam \| undefined` | Get next page param |

### Optional Options

| Option | Type | Description |
|--------|------|-------------|
| `getPreviousPageParam` | `(firstPage, allPages, firstPageParam, allPageParams) => TPageParam \| undefined` | Get previous page param |
| `maxPages` | `number` | Max pages to keep in cache |

### Return Values

| Property | Type | Description |
|----------|------|-------------|
| `data.pages` | `TData[]` | Array of all fetched pages |
| `data.pageParams` | `TPageParam[]` | Array of page params used |
| `fetchNextPage` | `() => Promise` | Fetch next page |
| `fetchPreviousPage` | `() => Promise` | Fetch previous page |
| `hasNextPage` | `boolean` | More pages available (forward) |
| `hasPreviousPage` | `boolean` | More pages available (backward) |
| `isFetchingNextPage` | `boolean` | Fetching next page |
| `isFetchingPreviousPage` | `boolean` | Fetching previous page |

### Example

```tsx
const {
  data,
  fetchNextPage,
  hasNextPage,
  isFetchingNextPage,
} = useInfiniteQuery({
  queryKey: ['projects'],
  queryFn: async ({ pageParam }) => {
    const res = await fetch(`/api/projects?cursor=${pageParam}`)
    return res.json()
  },
  initialPageParam: 0,
  getNextPageParam: (lastPage) => lastPage.nextCursor,
})

// Flatten pages for rendering
const allItems = data?.pages.flatMap(page => page.items) ?? []
```

---

## QueryClient Methods

### invalidateQueries

Mark queries as stale and optionally refetch.

```tsx
// Invalidate all queries
queryClient.invalidateQueries()

// Invalidate by key
queryClient.invalidateQueries({ queryKey: ['todos'] })

// Invalidate exact key
queryClient.invalidateQueries({ queryKey: ['todos', 1], exact: true })

// Control refetch behavior
queryClient.invalidateQueries({
  queryKey: ['todos'],
  refetchType: 'active',  // 'active' | 'inactive' | 'all' | 'none'
})

// Await invalidation (wait for refetch)
await queryClient.invalidateQueries({ queryKey: ['todos'] })
```

### setQueryData

Synchronously update cache.

```tsx
// Set data directly
queryClient.setQueryData(['todos'], newTodos)

// Update with function
queryClient.setQueryData(['todos'], (old) => [...old, newTodo])

// Update specific item
queryClient.setQueryData(['todos', todoId], updatedTodo)
```

### getQueryData

Read from cache.

```tsx
const todos = queryClient.getQueryData(['todos'])
const todo = queryClient.getQueryData(['todos', todoId])
```

### prefetchQuery

Prefetch data before it's needed.

```tsx
await queryClient.prefetchQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
})
```

### cancelQueries

Cancel in-flight queries.

```tsx
await queryClient.cancelQueries({ queryKey: ['todos'] })
```

### removeQueries

Remove queries from cache.

```tsx
queryClient.removeQueries({ queryKey: ['todos'] })
```

### resetQueries

Reset queries to initial state.

```tsx
queryClient.resetQueries({ queryKey: ['todos'] })
```

---

## Optimistic Updates

### Full Pattern

```tsx
const queryClient = useQueryClient()

useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo, context) => {
    // 1. Cancel outgoing refetches
    await context.client.cancelQueries({ queryKey: ['todos'] })

    // 2. Snapshot previous value
    const previousTodos = context.client.getQueryData(['todos'])

    // 3. Optimistically update
    context.client.setQueryData(['todos'], (old) => [...old, newTodo])

    // 4. Return rollback data
    return { previousTodos }
  },
  onError: (err, newTodo, onMutateResult, context) => {
    // 5. Rollback on error
    context.client.setQueryData(['todos'], onMutateResult.previousTodos)
  },
  onSettled: (data, error, variables, onMutateResult, context) => {
    // 6. Always refetch after
    context.client.invalidateQueries({ queryKey: ['todos'] })
  },
})
```

---

## TypeScript

### Typed Query

```tsx
type Todo = { id: string; title: string; completed: boolean }

const { data } = useQuery<Todo[], Error>({
  queryKey: ['todos'],
  queryFn: fetchTodos,
})
// data is Todo[] | undefined
```

### Typed Mutation

```tsx
type CreateTodoInput = { title: string }
type Todo = { id: string; title: string }

const mutation = useMutation<Todo, Error, CreateTodoInput>({
  mutationFn: (input) => createTodo(input),
})

mutation.mutate({ title: 'New Todo' })
// mutation.data is Todo | undefined
```

### QueryFunctionContext

```tsx
const { data } = useQuery({
  queryKey: ['todos', userId] as const,
  queryFn: async ({ queryKey }) => {
    const [, userId] = queryKey // typed as [string, string]
    return fetchTodos(userId)
  },
})
```

---

## Query Key Matching

```tsx
// These all match ['todos', 'list', { status: 'done' }]

// Prefix matching (default)
queryClient.invalidateQueries({ queryKey: ['todos'] })
queryClient.invalidateQueries({ queryKey: ['todos', 'list'] })

// Exact matching
queryClient.invalidateQueries({
  queryKey: ['todos', 'list', { status: 'done' }],
  exact: true,
})

// Predicate matching
queryClient.invalidateQueries({
  predicate: (query) => query.queryKey[0] === 'todos',
})
```

---

## Network Mode

| Mode | Behavior |
|------|----------|
| `'online'` (default) | Only fetch when online, pause when offline |
| `'always'` | Always fetch, ignore network state |
| `'offlineFirst'` | Try first request, pause retries if offline |

```tsx
const { data, fetchStatus } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  networkMode: 'offlineFirst',
})

// fetchStatus: 'fetching' | 'paused' | 'idle'
if (fetchStatus === 'paused') {
  // Show offline indicator
}
```

---

## Status vs FetchStatus

| `status` | Meaning |
|----------|---------|
| `'pending'` | No data yet |
| `'success'` | Data available |
| `'error'` | Query failed |

| `fetchStatus` | Meaning |
|---------------|---------|
| `'fetching'` | Request in flight |
| `'paused'` | Request paused (offline) |
| `'idle'` | No request running |

**Combinations:**
- `isPending && isFetching` = Loading for first time
- `isSuccess && isFetching` = Background refetch
- `isPending && !isFetching` = Disabled query, no data

---

## Global Callbacks

```tsx
const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error, query) => {
      // Global error handler for all queries
      console.error('Query error:', error)
    },
    onSuccess: (data, query) => {
      // Global success handler
    },
  }),
  mutationCache: new MutationCache({
    onError: (error, variables, context, mutation) => {
      // Global error handler for all mutations
      toast.error(error.message)
    },
  }),
})
```

---

*Source: TanStack Query v5 Documentation*
*https://tanstack.com/query/latest*
