---
name: tanstack-query-docs
description: |
  Official TanStack Query (React Query) v5 API reference with version awareness.
  Use for: useQuery, useMutation, useInfiniteQuery, QueryClient methods,
  configuration options, TypeScript types, migration guides, and troubleshooting.
  Includes version checking, changelog awareness, and GitHub issue resolution.
version: 5.0.0
license: MIT
---

# TanStack Query v5 API Reference

Official API documentation for TanStack Query (React Query) v5.

---

## IMPORTANT: Version Checking

**Before helping with TanStack Query issues, ALWAYS check the installed version:**

```bash
# Check installed version
npm list @tanstack/react-query
# or
bun pm ls | grep react-query
```

**Latest stable version:** Check https://github.com/TanStack/query/releases

### If Version Mismatch Detected

1. **Fetch changelog** to understand breaking changes:
   ```bash
   # View releases and changelogs
   gh release list -R TanStack/query --limit 10
   gh release view v5.x.x -R TanStack/query
   ```

2. **Key v4 → v5 breaking changes:**
   - `cacheTime` → `gcTime`
   - `isLoading` → `isPending` (for status)
   - `isLoading` now means `isPending && isFetching`
   - Object-only syntax for hooks (no positional args)
   - `onSuccess`/`onError`/`onSettled` removed from useQuery (use mutation callbacks or global handlers)

3. **Upgrade guide:** https://tanstack.com/query/latest/docs/framework/react/guides/migrating-to-v5

---

## Troubleshooting: Check GitHub Issues

**When facing an issue, ALWAYS check GitHub issues first:**

```bash
# Search for similar issues
gh issue list -R TanStack/query --search "your error message" --state all

# View specific issue
gh issue view <number> -R TanStack/query

# Check if it's a known bug with workaround
gh issue list -R TanStack/query --label "bug" --state open
```

**Common issue patterns:**
- Infinite re-renders → Usually missing `queryKey` dependencies or unstable references
- Data undefined after success → Check if using `select` incorrectly
- Mutations not invalidating → Missing `await` on `invalidateQueries`
- Memory leaks → Check `gcTime` and component unmount handling

---

## Setup

### QueryClientProvider

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,  // 5 minutes - data considered fresh
      gcTime: 1000 * 60 * 30,    // 30 minutes - unused cache retention
      retry: 3,                   // Retry failed requests 3 times
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
      refetchOnWindowFocus: true,
      refetchOnReconnect: true,
    },
    mutations: {
      retry: 1,
      onError: (error) => console.error('Mutation error:', error),
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

### React Native Setup (REQUIRED)

React Native doesn't have `window` focus events. You MUST configure these manually:

```tsx
// app/_layout.tsx or App.tsx
import { useEffect } from 'react'
import { AppState, Platform } from 'react-native'
import type { AppStateStatus } from 'react-native'
import NetInfo from '@react-native-community/netinfo'
import {
  QueryClient,
  QueryClientProvider,
  focusManager,
  onlineManager,
} from '@tanstack/react-query'

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,
      gcTime: 1000 * 60 * 30,
      retry: 2,
    },
  },
})

// 1. Online status management
onlineManager.setEventListener((setOnline) => {
  return NetInfo.addEventListener((state) => {
    setOnline(!!state.isConnected && !!state.isInternetReachable)
  })
})

// 2. App focus management
function onAppStateChange(status: AppStateStatus) {
  if (Platform.OS !== 'web') {
    focusManager.setFocused(status === 'active')
  }
}

export default function RootLayout() {
  useEffect(() => {
    const subscription = AppState.addEventListener('change', onAppStateChange)
    return () => subscription.remove()
  }, [])

  return (
    <QueryClientProvider client={queryClient}>
      {/* Your app */}
    </QueryClientProvider>
  )
}
```

### DevTools

```tsx
// Development only - automatically excluded in production builds
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <YourApp />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```

**React Native DevTools options:**
- Native macOS App: https://github.com/LovesWorking/rn-better-dev-tools
- Flipper Plugin: https://github.com/bgaleotti/react-query-native-devtools
- Reactotron Plugin: https://github.com/hsndmr/reactotron-react-query

---

## useQuery

### Full Signature

```tsx
const {
  // Data
  data,                    // TData | undefined - The fetched data
  dataUpdatedAt,           // number - Timestamp of last data update

  // Error
  error,                   // TError | null - Error if query failed
  errorUpdatedAt,          // number - Timestamp of last error
  failureCount,            // number - Number of consecutive failures
  failureReason,           // TError | null - Reason for last failure

  // Status flags
  status,                  // 'pending' | 'success' | 'error'
  fetchStatus,             // 'fetching' | 'paused' | 'idle'

  // Derived booleans
  isPending,               // status === 'pending'
  isSuccess,               // status === 'success'
  isError,                 // status === 'error'
  isFetching,              // fetchStatus === 'fetching'
  isPaused,                // fetchStatus === 'paused'
  isLoading,               // isPending && isFetching (NEW in v5!)
  isRefetching,            // isFetching && !isPending
  isFetched,               // Query has been fetched at least once
  isFetchedAfterMount,     // Query fetched after component mount
  isStale,                 // Data is stale
  isPlaceholderData,       // Currently showing placeholder data

  // Actions
  refetch,                 // () => Promise - Manually refetch
  promise,                 // Promise<TData> - For React.use() (experimental)
} = useQuery({
  queryKey,
  queryFn,
  // ... options
})
```

### Core Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `queryKey` | `unknown[]` | **Required** | Unique key for this query. Changes trigger refetch. |
| `queryFn` | `(context) => Promise<TData>` | **Required** | Function that fetches data. Must return a Promise. |
| `enabled` | `boolean` | `true` | Set `false` to disable automatic fetching. |
| `staleTime` | `number` | `0` | Time in ms data is considered fresh. |
| `gcTime` | `number` | `5 * 60 * 1000` | Time unused data stays in cache. |
| `retry` | `boolean \| number \| fn` | `3` | Retry failed queries. |
| `retryDelay` | `number \| fn` | Exponential | Delay between retries. |
| `refetchInterval` | `number \| false \| fn` | `false` | Auto-refetch interval. |
| `refetchOnMount` | `boolean \| 'always'` | `true` | Refetch when component mounts. |
| `refetchOnWindowFocus` | `boolean \| 'always'` | `true` | Refetch on window focus. |
| `refetchOnReconnect` | `boolean \| 'always'` | `true` | Refetch on network reconnect. |
| `networkMode` | `'online' \| 'always' \| 'offlineFirst'` | `'online'` | Network behavior. |
| `throwOnError` | `boolean \| fn` | `false` | Throw to Error Boundary. |

### Data Options

| Option | Type | Description |
|--------|------|-------------|
| `select` | `(data) => TSelected` | Transform data (doesn't affect cache). Only re-runs if data or function changes. |
| `placeholderData` | `TData \| fn` | Placeholder while pending. **Not cached.** |
| `initialData` | `TData \| fn` | Initial data. **Cached.** Considered stale by default. |
| `initialDataUpdatedAt` | `number` | Timestamp of initialData for staleness calculation. |

### QueryFunctionContext

```tsx
const { data } = useQuery({
  queryKey: ['todos', { status, page }] as const,
  queryFn: async ({ queryKey, signal, meta }) => {
    const [, { status, page }] = queryKey

    const response = await fetch(`/api/todos?status=${status}&page=${page}`, {
      signal, // AbortSignal for cancellation
    })

    if (!response.ok) throw new Error('Failed to fetch')
    return response.json()
  },
  meta: {
    // Custom metadata accessible in queryFn
    errorMessage: 'Failed to load todos'
  },
})
```

### Status vs FetchStatus (IMPORTANT!)

| `status` | Meaning |
|----------|---------|
| `'pending'` | No data yet (never fetched successfully) |
| `'success'` | Data available |
| `'error'` | Query failed |

| `fetchStatus` | Meaning |
|---------------|---------|
| `'fetching'` | Request in flight |
| `'paused'` | Request paused (offline) |
| `'idle'` | No request running |

**Common Combinations:**

| Scenario | `isPending` | `isFetching` | `isLoading` |
|----------|-------------|--------------|-------------|
| First load | `true` | `true` | `true` |
| Background refetch | `false` | `true` | `false` |
| Disabled, no data | `true` | `false` | `false` |
| Success, idle | `false` | `false` | `false` |
| Offline, paused | varies | `false` | `false` |

---

## useMutation

### Full Signature

```tsx
const {
  // Data
  data,                    // TData | undefined - Mutation result
  variables,               // TVariables - Variables passed to mutate
  submittedAt,             // number - Timestamp of submission

  // Error
  error,                   // TError | null - Error if failed
  failureCount,            // number - Consecutive failures
  failureReason,           // TError | null - Last failure reason

  // Status
  status,                  // 'idle' | 'pending' | 'success' | 'error'
  isIdle,                  // status === 'idle'
  isPending,               // status === 'pending'
  isSuccess,               // status === 'success'
  isError,                 // status === 'error'
  isPaused,                // Mutation paused (offline)

  // Actions
  mutate,                  // (variables, options?) => void
  mutateAsync,             // (variables, options?) => Promise<TData>
  reset,                   // () => void - Reset to idle state
} = useMutation({
  mutationFn,
  // ... options
})
```

### Options

| Option | Type | Description |
|--------|------|-------------|
| `mutationFn` | `(variables) => Promise<TData>` | **Required.** Function that performs mutation. |
| `mutationKey` | `unknown[]` | Optional key for the mutation. |
| `onMutate` | `(variables, context) => Promise<TContext>` | Called before mutation. Return value passed to other callbacks. |
| `onSuccess` | `(data, variables, onMutateResult, context) => void` | Called on success. |
| `onError` | `(error, variables, onMutateResult, context) => void` | Called on error. |
| `onSettled` | `(data, error, variables, onMutateResult, context) => void` | Called on settle (success or error). |
| `retry` | `boolean \| number \| fn` | Retry failed mutations. |
| `gcTime` | `number` | Cache time for mutation result. |
| `networkMode` | `'online' \| 'always' \| 'offlineFirst'` | Network behavior. |

### Lifecycle Callbacks (Order Matters!)

```tsx
useMutation({
  mutationFn: updateTodo,

  // 1. Called BEFORE mutation executes
  onMutate: async (variables, context) => {
    // Cancel outgoing refetches
    await context.client.cancelQueries({ queryKey: ['todos'] })

    // Snapshot current data
    const previousData = context.client.getQueryData(['todos'])

    // Optimistically update
    context.client.setQueryData(['todos'], (old) => [...old, variables])

    // Return context for rollback
    return { previousData }
  },

  // 2. Called if mutation FAILS
  onError: (error, variables, onMutateResult, context) => {
    // Rollback optimistic update
    context.client.setQueryData(['todos'], onMutateResult.previousData)
  },

  // 3. Called if mutation SUCCEEDS
  onSuccess: (data, variables, onMutateResult, context) => {
    // Can update cache with server response
    console.log('Success!', data)
  },

  // 4. Called ALWAYS (success or error)
  onSettled: (data, error, variables, onMutateResult, context) => {
    // Refetch to ensure consistency
    context.client.invalidateQueries({ queryKey: ['todos'] })
  },
})
```

### mutate vs mutateAsync

```tsx
// PREFER: mutate - handles errors automatically
const onSubmit = () => {
  mutation.mutate(data, {
    onSuccess: (result) => {
      navigation.navigate('Detail', { id: result.id })
    },
    // Per-call callbacks fire AFTER hook-level callbacks
  })
}

// AVOID: mutateAsync - requires manual error handling
const onSubmit = async () => {
  try {
    const result = await mutation.mutateAsync(data)
    navigation.navigate('Detail', { id: result.id })
  } catch (error) {
    // MUST handle errors manually or you get unhandled rejection!
  }
}
```

---

## useInfiniteQuery

### Full Signature

```tsx
const {
  // Data
  data,                      // { pages: TData[], pageParams: TPageParam[] }

  // Pagination state
  hasNextPage,               // boolean - More pages forward?
  hasPreviousPage,           // boolean - More pages backward?
  isFetchingNextPage,        // boolean - Fetching next?
  isFetchingPreviousPage,    // boolean - Fetching previous?
  isFetchNextPageError,      // boolean - Error fetching next?
  isFetchPreviousPageError,  // boolean - Error fetching previous?

  // Pagination actions
  fetchNextPage,             // (options?) => Promise
  fetchPreviousPage,         // (options?) => Promise

  // ... all useQuery return values
} = useInfiniteQuery({
  queryKey,
  queryFn: ({ pageParam }) => fetchPage(pageParam),
  initialPageParam,
  getNextPageParam,
  getPreviousPageParam,      // Optional
  maxPages,                  // Optional - limit pages in cache
})
```

### Required Options

| Option | Type | Description |
|--------|------|-------------|
| `queryKey` | `unknown[]` | Unique key for this query. |
| `queryFn` | `({ pageParam }) => Promise<TData>` | Fetch function receiving `pageParam`. |
| `initialPageParam` | `TPageParam` | First page parameter. |
| `getNextPageParam` | `(lastPage, allPages, lastPageParam, allPageParams) => TPageParam \| undefined` | Return next page param or `undefined` if no more pages. |

### Optional Options

| Option | Type | Description |
|--------|------|-------------|
| `getPreviousPageParam` | `fn` | For bidirectional pagination. |
| `maxPages` | `number` | Max pages to keep in cache. Older pages removed when exceeded. |

### Example: Cursor-based Pagination

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
    return res.json() // { items: [...], nextCursor: '...' }
  },
  initialPageParam: null as string | null,
  getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
})

// Flatten pages for rendering
const allItems = data?.pages.flatMap(page => page.items) ?? []
```

### Example: Offset-based Pagination

```tsx
const {
  data,
  fetchNextPage,
  hasNextPage,
} = useInfiniteQuery({
  queryKey: ['posts'],
  queryFn: async ({ pageParam }) => {
    const res = await fetch(`/api/posts?offset=${pageParam}&limit=10`)
    return res.json() // { items: [...], total: 100 }
  },
  initialPageParam: 0,
  getNextPageParam: (lastPage, allPages) => {
    const totalFetched = allPages.flatMap(p => p.items).length
    return totalFetched < lastPage.total ? totalFetched : undefined
  },
})
```

### FlatList Integration (React Native)

```tsx
function PostList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
  } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam }) => fetchPosts(pageParam),
    initialPageParam: 0,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  })

  const posts = data?.pages.flatMap(page => page.items) ?? []

  if (isLoading) return <LoadingSpinner />

  return (
    <FlatList
      data={posts}
      renderItem={({ item }) => <PostItem post={item} />}
      keyExtractor={(item) => item.id}
      onEndReached={() => {
        if (hasNextPage && !isFetchingNextPage) {
          fetchNextPage()
        }
      }}
      onEndReachedThreshold={0.5}
      ListFooterComponent={
        isFetchingNextPage ? <ActivityIndicator /> : null
      }
    />
  )
}
```

---

## Dependent Queries

Use `enabled` to chain queries:

```tsx
// 1. First query
const { data: user } = useQuery({
  queryKey: ['user', email],
  queryFn: () => fetchUser(email),
})

// 2. Dependent query - waits for user
const { data: projects, isPending, fetchStatus } = useQuery({
  queryKey: ['projects', user?.id],
  queryFn: () => fetchProjects(user!.id),
  enabled: !!user?.id, // Only runs when user.id exists
})

// Check states:
// isPending && fetchStatus === 'idle' → Waiting for dependency
// isPending && fetchStatus === 'fetching' → Actually loading
```

---

## Parallel Queries

### useQueries

```tsx
const results = useQueries({
  queries: userIds.map(id => ({
    queryKey: ['user', id],
    queryFn: () => fetchUser(id),
    staleTime: 1000 * 60,
  })),
})

// Each result has full useQuery return value
results.forEach((result, index) => {
  if (result.isPending) console.log(`User ${userIds[index]} loading...`)
  if (result.isSuccess) console.log(`User: ${result.data.name}`)
})
```

### useQueries with combine

```tsx
const { data, isPending } = useQueries({
  queries: ids.map(id => ({
    queryKey: ['post', id],
    queryFn: () => fetchPost(id),
  })),
  combine: (results) => ({
    data: results.map(r => r.data).filter(Boolean),
    isPending: results.some(r => r.isPending),
  }),
})
```

---

## QueryClient Methods

### invalidateQueries

Mark queries as stale and optionally refetch.

```tsx
// Invalidate all queries
queryClient.invalidateQueries()

// Invalidate by prefix (fuzzy match)
queryClient.invalidateQueries({ queryKey: ['todos'] })

// Invalidate exact key only
queryClient.invalidateQueries({
  queryKey: ['todos', { status: 'done' }],
  exact: true
})

// Control what gets refetched
queryClient.invalidateQueries({
  queryKey: ['todos'],
  refetchType: 'active',  // 'active' | 'inactive' | 'all' | 'none'
})

// IMPORTANT: Await to wait for refetch completion
await queryClient.invalidateQueries({ queryKey: ['todos'] })
```

### setQueryData

Synchronously update cache.

```tsx
// Set data directly
queryClient.setQueryData(['todos'], newTodos)

// Update with function (MUST be immutable!)
queryClient.setQueryData(['todos'], (old) => {
  if (!old) return [newTodo]
  return [...old, newTodo] // Don't mutate old!
})

// Update specific item
queryClient.setQueryData(['todo', todoId], updatedTodo)
```

### getQueryData

Read from cache synchronously.

```tsx
const todos = queryClient.getQueryData(['todos'])
const todo = queryClient.getQueryData(['todo', todoId])

// Returns undefined if not in cache
```

### ensureQueryData

Get cached data or fetch if missing.

```tsx
// Returns cached data or fetches
const data = await queryClient.ensureQueryData({
  queryKey: ['todos'],
  queryFn: fetchTodos,
})
```

### prefetchQuery

Prefetch before needed.

```tsx
// Prefetch on hover, route load, etc.
await queryClient.prefetchQuery({
  queryKey: ['todo', todoId],
  queryFn: () => fetchTodo(todoId),
  staleTime: 1000 * 60 * 5, // Won't refetch if fresh
})
```

### cancelQueries

Cancel in-flight queries.

```tsx
await queryClient.cancelQueries({ queryKey: ['todos'] })
```

### removeQueries

Remove from cache entirely.

```tsx
queryClient.removeQueries({ queryKey: ['todos'] })
```

### resetQueries

Reset to initial state.

```tsx
queryClient.resetQueries({ queryKey: ['todos'] })
```

---

## queryOptions Helper

Type-safe, reusable query configurations:

```tsx
// queries/todos.ts
import { queryOptions } from '@tanstack/react-query'

export const todoQueries = {
  all: () => queryOptions({
    queryKey: ['todos'],
    queryFn: fetchAllTodos,
  }),

  list: (filters: TodoFilters) => queryOptions({
    queryKey: ['todos', 'list', filters],
    queryFn: () => fetchTodos(filters),
    staleTime: 1000 * 60 * 5,
  }),

  detail: (id: string) => queryOptions({
    queryKey: ['todos', 'detail', id],
    queryFn: () => fetchTodo(id),
  }),
}
```

**Usage:**

```tsx
// In component - fully typed
const { data } = useQuery(todoQueries.detail(id))

// Prefetching - fully typed
await queryClient.prefetchQuery(todoQueries.detail(id))

// Getting cached data - fully typed!
const todo = queryClient.getQueryData(todoQueries.detail(id).queryKey)
//    ^? Todo | undefined
```

---

## Optimistic Updates

### Full Pattern with Rollback

```tsx
const queryClient = useQueryClient()

const mutation = useMutation({
  mutationFn: updateTodo,

  onMutate: async (newTodo, context) => {
    // 1. Cancel any outgoing refetches
    await context.client.cancelQueries({ queryKey: ['todos'] })

    // 2. Snapshot the previous value
    const previousTodos = context.client.getQueryData(['todos'])

    // 3. Optimistically update to the new value
    context.client.setQueryData(['todos'], (old) =>
      old?.map(t => t.id === newTodo.id ? { ...t, ...newTodo } : t)
    )

    // 4. Return a context with the snapshot
    return { previousTodos }
  },

  onError: (err, newTodo, onMutateResult, context) => {
    // 5. Rollback on error
    if (onMutateResult?.previousTodos) {
      context.client.setQueryData(['todos'], onMutateResult.previousTodos)
    }
  },

  onSettled: (data, error, variables, onMutateResult, context) => {
    // 6. Always refetch after error or success
    context.client.invalidateQueries({ queryKey: ['todos'] })
  },
})
```

---

## Error Handling

### Local Error State

```tsx
const { data, error, isError } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
})

if (isError) {
  return <ErrorMessage error={error} />
}
```

### Error Boundaries with throwOnError

```tsx
// Throw all errors to boundary
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  throwOnError: true,
})

// Selective: only 5xx errors to boundary
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  throwOnError: (error) => error.response?.status >= 500,
})
```

### Global Error Handler

```tsx
import { QueryCache, MutationCache, QueryClient } from '@tanstack/react-query'

const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error, query) => {
      // Only show toast for background errors (has cached data)
      if (query.state.data !== undefined) {
        toast.error(`Background update failed: ${error.message}`)
      }
    },
  }),
  mutationCache: new MutationCache({
    onError: (error) => {
      toast.error(`Operation failed: ${error.message}`)
    },
  }),
})
```

---

## Network Mode

| Mode | Behavior |
|------|----------|
| `'online'` (default) | Fetch only when online. Pauses when offline. |
| `'always'` | Always fetch, ignore network state. Use for local-first. |
| `'offlineFirst'` | Try first request. If offline, pause retries. Good with service workers. |

```tsx
const { data, fetchStatus } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  networkMode: 'offlineFirst',
})

// Show offline indicator
if (fetchStatus === 'paused') {
  return <Banner>You're offline. Showing cached data.</Banner>
}
```

---

## TypeScript

### Typed useQuery

```tsx
type Todo = { id: string; title: string; completed: boolean }

// Type inference from queryFn
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: async (): Promise<Todo[]> => {
    const res = await fetch('/api/todos')
    return res.json()
  },
})
// data is Todo[] | undefined

// Explicit generics
const { data } = useQuery<Todo[], Error>({
  queryKey: ['todos'],
  queryFn: fetchTodos,
})
```

### Typed useMutation

```tsx
type CreateTodoInput = { title: string }
type Todo = { id: string; title: string }

const mutation = useMutation<Todo, Error, CreateTodoInput>({
  mutationFn: (input) => createTodo(input),
})

mutation.mutate({ title: 'New Todo' })
// mutation.data is Todo | undefined
```

### Type-Safe Query Keys

```tsx
const { data } = useQuery({
  queryKey: ['todos', userId, { status }] as const,
  queryFn: async ({ queryKey }) => {
    const [, id, filters] = queryKey
    //     ^string  ^{ status: string }
    return fetchTodos(id, filters)
  },
})
```

---

## v4 → v5 Migration Quick Reference

| v4 | v5 |
|----|-----|
| `cacheTime` | `gcTime` |
| `status: 'loading'` | `status: 'pending'` |
| `isLoading` (status) | `isPending` |
| `isInitialLoading` | `isLoading` (= `isPending && isFetching`) |
| `useQuery(key, fn, opts)` | `useQuery({ queryKey, queryFn, ...opts })` |
| `onSuccess` in useQuery | Removed. Use global callbacks or effects. |
| `onError` in useQuery | Removed. Use global callbacks or effects. |

---

## Common Debugging Commands

```bash
# Check installed version
npm list @tanstack/react-query

# View latest releases
gh release list -R TanStack/query --limit 5

# Search issues for your error
gh issue list -R TanStack/query --search "your error" --state all

# View specific issue with comments
gh issue view 1234 -R TanStack/query --comments

# Check if issue is fixed in newer version
gh release view v5.x.x -R TanStack/query
```

---

*Source: TanStack Query v5 Documentation*
*https://tanstack.com/query/latest*
*GitHub: https://github.com/TanStack/query*
