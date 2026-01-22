---
name: tanstack-query-react-native
description: |
  TanStack Query (React Query) expert patterns for React Native/Expo.
  Use for: data fetching, query keys, mutations, invalidation, optimistic updates,
  offline support, error handling, infinite scroll, cache management, TypeScript.
  Based on TkDodo's authoritative "Practical React Query" blog series.
version: 2.0.0
license: MIT
---

# TanStack Query for React Native

**Expert-level TanStack Query patterns for React Native/Expo apps.**

> This skill is based on TkDodo's comprehensive "Practical React Query" blog series.
> TkDodo is a core maintainer of TanStack Query.

---

## Quick Setup

```tsx
// app/_layout.tsx
import { QueryClient, QueryClientProvider, focusManager, onlineManager } from '@tanstack/react-query';
import { AppState, Platform } from 'react-native';
import NetInfo from '@react-native-community/netinfo';
import { useEffect } from 'react';

// 1. Create client with RN-optimized defaults
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 min - prevents excessive refetches
      gcTime: 1000 * 60 * 30,   // 30 min cache
      retry: 2,
    },
  },
});

// 2. Online status (auto-refetch on reconnect)
onlineManager.setEventListener((setOnline) => {
  return NetInfo.addEventListener((state) => {
    setOnline(!!state.isConnected && !!state.isInternetReachable);
  });
});

// 3. Focus management (refetch on app foreground)
function onAppStateChange(status: AppStateStatus) {
  if (Platform.OS !== 'web') {
    focusManager.setFocused(status === 'active');
  }
}

export default function RootLayout() {
  useEffect(() => {
    const sub = AppState.addEventListener('change', onAppStateChange);
    return () => sub.remove();
  }, []);

  return (
    <QueryClientProvider client={queryClient}>
      <Stack />
    </QueryClientProvider>
  );
}
```

---

## Core Concepts

### Query Keys as Dependency Arrays

> **TkDodo:** "Queries are declarative. If you have some state that changes your data,
> all you need to do is put it in the Query Key, because React Query will trigger
> a refetch automatically whenever the key changes."

```tsx
// WRONG: Using refetch to change parameters
const { refetch } = useQuery({ queryKey: ['todos'], queryFn: fetchTodos });
<Filters onApply={() => refetch(???)} /> // Don't do this!

// RIGHT: Query key drives the query
const [filters, setFilters] = useState<Filters>();
const { data } = useQuery({
  queryKey: ['todos', filters],
  queryFn: () => fetchTodos(filters),
});
<Filters onApply={setFilters} /> // This triggers automatic refetch
```

### Server State vs Client State

React Query manages **server state** (data you don't own, borrowed from the server).
Keep it separate from **client state** (UI state, form inputs).

```tsx
// WRONG: Copying server state to local state
const { data } = useQuery({ queryKey: ['user'], queryFn: fetchUser });
const [user, setUser] = useState(data); // Don't sync to local state!

// RIGHT: Use server state directly
const { data: user } = useQuery({ queryKey: ['user'], queryFn: fetchUser });
// Use `user` directly in your components
```

---

## Important Defaults to Understand

| Default | Value | Meaning |
|---------|-------|---------|
| `staleTime` | `0` | Data is stale immediately after fetch |
| `gcTime` | `5 min` | Inactive queries garbage collected |
| `retry` | `3` | Silent retries with exponential backoff |
| `refetchOnMount` | `true` | Refetch when component mounts |
| `refetchOnWindowFocus` | `true` | Refetch when app returns to foreground |
| `refetchOnReconnect` | `true` | Refetch when network reconnects |

### staleTime Strategy

```tsx
// Fresh for 5 minutes - no refetches during this period
const { data } = useQuery({
  queryKey: ['user', userId],
  queryFn: fetchUser,
  staleTime: 1000 * 60 * 5,
});

// Never refetch unless manually invalidated
staleTime: Infinity

// Never refetch, even with invalidation (truly static)
staleTime: 'static'
```

> **TkDodo:** "I think setting a global staleTime to at least 20 seconds is a good
> idea for most applications. I've even seen 5 minutes work well."

---

## Query Key Factories

Structure keys from **most generic** to **most specific**. Use factories for consistency:

```tsx
// features/todos/queries.ts
import { queryOptions } from '@tanstack/react-query';

export const todoKeys = {
  all: ['todos'] as const,
  lists: () => [...todoKeys.all, 'list'] as const,
  list: (filters: TodoFilters) => [...todoKeys.lists(), filters] as const,
  details: () => [...todoKeys.all, 'detail'] as const,
  detail: (id: string) => [...todoKeys.details(), id] as const,
};

// With queryOptions for type-safe prefetching
export const todoQueries = {
  detail: (id: string) =>
    queryOptions({
      queryKey: todoKeys.detail(id),
      queryFn: () => fetchTodo(id),
      staleTime: 1000 * 60 * 5,
    }),
  list: (filters: TodoFilters) =>
    queryOptions({
      queryKey: todoKeys.list(filters),
      queryFn: () => fetchTodos(filters),
    }),
};
```

**Usage:**

```tsx
// In components
const { data } = useQuery(todoQueries.detail(id));

// Invalidation (fuzzy matching)
queryClient.invalidateQueries({ queryKey: todoKeys.all });        // All todos
queryClient.invalidateQueries({ queryKey: todoKeys.lists() });    // All lists
queryClient.invalidateQueries({ queryKey: todoKeys.detail(id) }); // Specific detail
```

> **TkDodo:** "I keep my Query Keys next to their respective queries,
> co-located in a feature directory. The queries file will contain everything
> React Query related."

---

## Mutations

### Invalidation vs Direct Updates

```tsx
// Option 1: Invalidation (recommended for most cases)
const useUpdateTodo = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: updateTodo,
    onSuccess: () => {
      // Refetch all todo lists
      queryClient.invalidateQueries({ queryKey: todoKeys.lists() });
    },
  });
};

// Option 2: Direct update (when you have the complete response)
const useUpdateTodo = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: updateTodo,
    onSuccess: (updatedTodo) => {
      // Update cache directly
      queryClient.setQueryData(todoKeys.detail(updatedTodo.id), updatedTodo);
      // Still invalidate lists (position might have changed)
      queryClient.invalidateQueries({ queryKey: todoKeys.lists() });
    },
  });
};
```

> **TkDodo:** "I personally think that most of the time, invalidation should be
> preferred. For direct updates to work reliably, you need more code on the frontend,
> and to some extent duplicate logic from the backend."

### Await Invalidation for Loading States

```tsx
// WRONG: Mutation ends before invalidation completes
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: todoKeys.lists() });
}

// RIGHT: Mutation stays loading until refetch completes
onSuccess: async () => {
  await queryClient.invalidateQueries({ queryKey: todoKeys.lists() });
}
```

### mutate vs mutateAsync

```tsx
// PREFER: mutate - handles errors automatically
const onSubmit = () => {
  mutation.mutate(data, {
    onSuccess: (result) => navigation.navigate('Detail', { id: result.id }),
  });
};

// AVOID: mutateAsync - requires manual error handling
const onSubmit = async () => {
  try {
    const result = await mutation.mutateAsync(data);
    navigation.navigate('Detail', { id: result.id });
  } catch (error) {
    // Must handle errors manually!
  }
};
```

> **TkDodo:** "You should almost always use `mutate`. Since `mutateAsync` gives you
> control over the Promise, you also have to catch errors manually, or you might
> get an unhandled promise rejection."

---

## Optimistic Updates

Use sparingly - best for toggle buttons and instant feedback scenarios:

```tsx
const useToggleTodo = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: toggleTodo,
    onMutate: async (todoId) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: todoKeys.detail(todoId) });

      // Snapshot previous value
      const previousTodo = queryClient.getQueryData(todoKeys.detail(todoId));

      // Optimistically update
      queryClient.setQueryData(todoKeys.detail(todoId), (old) => ({
        ...old,
        completed: !old.completed,
      }));

      return { previousTodo };
    },
    onError: (err, todoId, context) => {
      // Rollback on error
      queryClient.setQueryData(todoKeys.detail(todoId), context?.previousTodo);
    },
    onSettled: (_, __, todoId) => {
      // Always refetch after
      queryClient.invalidateQueries({ queryKey: todoKeys.detail(todoId) });
    },
  });
};
```

> **TkDodo:** "Optimistic updates are a bit over-used. You should really be sure
> that it rarely fails, because the UX for a rollback is not great. Also be sure
> that instant feedback is really required."

**When to use optimistic updates:**
- Toggle buttons (like/unlike, complete/uncomplete)
- Counter increments
- Simple boolean flips

**When NOT to use:**
- Forms that close/redirect on submit
- Complex data transformations
- Operations that frequently fail

---

## Error Handling

### Three Approaches

```tsx
// 1. Local error property - handle in component
const { data, error, isError } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
});

if (isError) return <ErrorMessage error={error} />;

// 2. Error Boundaries with throwOnError
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  throwOnError: true, // Propagates to nearest Error Boundary
});

// 3. Selective throwOnError (5xx to boundary, 4xx local)
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  throwOnError: (error) => error.response?.status >= 500,
});
```

### Global Error Toast (React Native)

```tsx
// In your app setup
import Toast from 'react-native-toast-message';

const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error, query) => {
      // Only show toast if we have cached data (background error)
      if (query.state.data !== undefined) {
        Toast.show({
          type: 'error',
          text1: 'Background update failed',
          text2: error.message,
        });
      }
    },
  }),
  mutationCache: new MutationCache({
    onError: (error) => {
      Toast.show({
        type: 'error',
        text1: 'Operation failed',
        text2: error.message,
      });
    },
  }),
});
```

> **TkDodo:** "The global callbacks on QueryCache are likely the best place to put
> any sort of error tracking or monitoring, because it's guaranteed to run only
> once per request."

---

## Offline Support & Network Mode

### networkMode Options

| Mode | Behavior |
|------|----------|
| `online` (default) | Only fetch when online, pause when offline |
| `always` | Always fetch, ignore network state |
| `offlineFirst` | Try first request, pause retries if offline |

### Understanding fetchStatus

```tsx
const { status, fetchStatus, data } = useQuery({...});

// status: 'pending' | 'success' | 'error' - about DATA
// fetchStatus: 'fetching' | 'paused' | 'idle' - about NETWORK REQUEST
```

**Show offline indicator:**

```tsx
function TodoList() {
  const { data, fetchStatus } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  });

  return (
    <View>
      {fetchStatus === 'paused' && (
        <Banner>You're offline. Showing cached data.</Banner>
      )}
      <FlatList data={data} ... />
    </View>
  );
}
```

### Offline-First Pattern

```tsx
// For apps with service worker or HTTP cache
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  networkMode: 'offlineFirst', // Try request, pause retries if fails offline
});
```

---

## Infinite Queries for Mobile Lists

```tsx
function TodoList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['todos', 'infinite'],
    queryFn: ({ pageParam }) => fetchTodos({ cursor: pageParam }),
    initialPageParam: null,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
  });

  const todos = data?.pages.flatMap((page) => page.items) ?? [];

  return (
    <FlatList
      data={todos}
      renderItem={({ item }) => <TodoItem todo={item} />}
      keyExtractor={(item) => item.id}
      onEndReached={() => {
        if (hasNextPage && !isFetchingNextPage) {
          fetchNextPage();
        }
      }}
      onEndReachedThreshold={0.5}
      ListFooterComponent={
        isFetchingNextPage ? <ActivityIndicator /> : null
      }
    />
  );
}
```

---

## Data Transformations with select

Transform data at the subscriber level (not in cache):

```tsx
// Only subscribe to todo count (won't re-render for other changes)
const { data: todoCount } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  select: (data) => data.length,
});

// Transform for FlatList
const { data: sortedTodos } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  select: (data) => [...data].sort((a, b) => a.createdAt - b.createdAt),
});
```

> **TkDodo:** "The select option is great for partial subscriptions. The component
> will only re-render if the selected part changes."

---

## Cache Seeding (List → Detail Navigation)

```tsx
// In list screen: seed detail cache when navigating
function TodoList() {
  const queryClient = useQueryClient();
  const { data: todos } = useQuery(todoQueries.list());

  const handlePress = (todo: Todo) => {
    // Pre-populate detail cache with list data
    queryClient.setQueryData(todoKeys.detail(todo.id), todo);
    navigation.navigate('TodoDetail', { id: todo.id });
  };

  return (
    <FlatList
      data={todos}
      renderItem={({ item }) => (
        <TouchableOpacity onPress={() => handlePress(item)}>
          <TodoItem todo={item} />
        </TouchableOpacity>
      )}
    />
  );
}

// In detail screen: use seeded data, but refetch for freshness
function TodoDetail({ route }) {
  const { id } = route.params;

  const { data: todo } = useQuery({
    ...todoQueries.detail(id),
    // Data shows instantly from seed, then refetches
  });

  return <TodoView todo={todo} />;
}
```

---

## Essential Hooks for React Native

### useRefreshByUser - Pull-to-Refresh

```tsx
export function useRefreshByUser(refetch: () => Promise<unknown>) {
  const [isRefetchingByUser, setIsRefetchingByUser] = useState(false);

  async function refetchByUser() {
    setIsRefetchingByUser(true);
    try {
      await refetch();
    } finally {
      setIsRefetchingByUser(false);
    }
  }

  return { isRefetchingByUser, refetchByUser };
}

// Usage
function MyScreen() {
  const { data, refetch } = useQuery({ queryKey: ['items'], queryFn: fetchItems });
  const { isRefetchingByUser, refetchByUser } = useRefreshByUser(refetch);

  return (
    <ScrollView
      refreshControl={
        <RefreshControl refreshing={isRefetchingByUser} onRefresh={refetchByUser} />
      }
    >
      {/* content */}
    </ScrollView>
  );
}
```

### useRefreshOnFocus - Screen Navigation

```tsx
export function useRefreshOnFocus(refetch: () => void) {
  const firstTimeRef = useRef(true);

  useFocusEffect(
    useCallback(() => {
      if (firstTimeRef.current) {
        firstTimeRef.current = false;
        return; // Skip first mount (already fetches)
      }
      refetch();
    }, [refetch])
  );
}
```

### subscribed - Pause Background Updates

```tsx
import { useIsFocused } from '@react-navigation/native';

function ExpensiveScreen() {
  const isFocused = useIsFocused();

  const { data } = useQuery({
    queryKey: ['expensive-data'],
    queryFn: fetchExpensiveData,
    subscribed: isFocused, // Unsubscribe when screen loses focus
  });

  return <View>{/* render data */}</View>;
}
```

---

## Decision Trees

### When to Use Optimistic Updates?

```
Is instant feedback critical?
├── No → Use standard mutation with loading state
└── Yes
    └── Does it rarely fail?
        ├── No → Use standard mutation (rollback UX is bad)
        └── Yes
            └── Is the update simple (toggle, increment)?
                ├── Yes → Use optimistic update
                └── No → Consider if complexity is worth it
```

### staleTime Selection

```
How often does data change?
├── Never (static) → staleTime: Infinity
├── Rarely (config, user profile) → staleTime: 5-30 minutes
├── Sometimes (lists) → staleTime: 1-5 minutes
└── Frequently (chat, live data) → staleTime: 0 or use WebSockets
```

### Invalidation vs Direct Update

```
After mutation success:
├── Do you have the complete updated entity?
│   ├── No → Invalidate
│   └── Yes
│       └── Could the position in lists change?
│           ├── Yes → Direct update detail + Invalidate lists
│           └── No → Direct update only
└── Is it complex data transformation?
    └── Yes → Just invalidate (let server handle it)
```

---

## Common Mistakes

### 1. Using refetch() to Change Parameters

```tsx
// WRONG
const { refetch } = useQuery({ queryKey: ['todos'], queryFn: fetchTodos });
onClick={() => refetch(newFilters)} // refetch doesn't take parameters!

// RIGHT
const [filters, setFilters] = useState();
const { data } = useQuery({
  queryKey: ['todos', filters],
  queryFn: () => fetchTodos(filters)
});
onClick={() => setFilters(newFilters)} // Key change triggers refetch
```

### 2. Not Awaiting invalidateQueries

```tsx
// WRONG: Mutation ends, then invalidation runs async
onSuccess: () => {
  queryClient.invalidateQueries({ queryKey: ['todos'] });
  navigation.goBack(); // Might navigate before refetch!
}

// RIGHT: Wait for invalidation
onSuccess: async () => {
  await queryClient.invalidateQueries({ queryKey: ['todos'] });
  navigation.goBack();
}
```

### 3. Copying Server State to Local State

```tsx
// WRONG: Syncing creates stale data bugs
const { data } = useQuery({ queryKey: ['user'], queryFn: fetchUser });
const [user, setUser] = useState(data);
useEffect(() => { setUser(data) }, [data]); // Don't do this!

// RIGHT: Use server state directly, or use initialData
const { data: user } = useQuery({ queryKey: ['user'], queryFn: fetchUser });
// For forms, copy once with initialData on useMutation
```

### 4. staleTime: 0 Causing Excessive Refetches

```tsx
// Problem: Every mount/focus triggers refetch
const { data } = useQuery({ queryKey: ['todos'], queryFn: fetchTodos });
// With default staleTime: 0, data is immediately stale

// Solution: Set reasonable staleTime
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  staleTime: 1000 * 60 * 5, // 5 minutes
});
```

### 5. Using mutateAsync Without try/catch

```tsx
// WRONG: Unhandled promise rejection
const onSubmit = async () => {
  const result = await mutation.mutateAsync(data); // Error not caught!
  doSomething(result);
};

// RIGHT: Use mutate with callbacks
const onSubmit = () => {
  mutation.mutate(data, {
    onSuccess: (result) => doSomething(result),
  });
};
```

### 6. No Online/Focus Manager Setup

```tsx
// Without these, RN apps miss automatic refetches

// Add to app root:
onlineManager.setEventListener((setOnline) => {
  return NetInfo.addEventListener((state) => {
    setOnline(!!state.isConnected);
  });
});

AppState.addEventListener('change', (status) => {
  if (Platform.OS !== 'web') {
    focusManager.setFocused(status === 'active');
  }
});
```

---

## Testing React Query

### Test Setup

```tsx
// test-utils.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { render } from '@testing-library/react-native';

const createTestQueryClient = () =>
  new QueryClient({
    defaultOptions: {
      queries: {
        retry: false, // Important! Prevents retries in tests
        gcTime: Infinity, // Keep cache during test
      },
    },
  });

export function renderWithClient(ui: React.ReactElement) {
  const testQueryClient = createTestQueryClient();
  return {
    ...render(
      <QueryClientProvider client={testQueryClient}>{ui}</QueryClientProvider>
    ),
    queryClient: testQueryClient,
  };
}

// For testing hooks
export function createWrapper() {
  const queryClient = createTestQueryClient();
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
}
```

### Testing with MSW (Recommended)

```tsx
// Use MSW to mock network requests - React Query doesn't know the difference
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

const server = setupServer(
  http.get('/api/todos', () => {
    return HttpResponse.json([{ id: 1, title: 'Test Todo' }]);
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('loads todos', async () => {
  const { findByText } = renderWithClient(<TodoList />);
  expect(await findByText('Test Todo')).toBeTruthy();
});
```

> **TkDodo:** "Turn off retries. If you have retries on and a test fails, it will
> wait for all retries before failing - making your tests slower for no reason."

### Pre-seeding Cache for Tests

```tsx
const createWrapper = (initialData?: Todo[]) => {
  const queryClient = createTestQueryClient();
  if (initialData) {
    queryClient.setQueryData(['todos'], initialData);
  }
  return ({ children }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
};
```

---

## Dependent Mutations (Chaining)

Three approaches for sequential mutations:

```tsx
// 1. Multiple calls in one mutationFn (one loading state)
const useCreateAndAssign = () =>
  useMutation({
    mutationFn: async (data) => {
      const todo = await createTodo(data);
      return assignTodo(todo.id, data.userId);
    },
  });

// 2. Chained callbacks (cleanest for 2 mutations)
const useChainedMutations = () => {
  const mutation1 = useMutation({ mutationFn: createTodo });
  const mutation2 = useMutation({ mutationFn: assignTodo });

  return {
    mutate: (data) => {
      mutation1.mutate(data.todo, {
        onSuccess: (todo) => mutation2.mutate({ todoId: todo.id, userId: data.userId }),
      });
    },
  };
};

// 3. mutateAsync when you need the result (handle errors!)
const onSubmit = async () => {
  try {
    const todo = await createMutation.mutateAsync(data);
    await assignMutation.mutateAsync({ todoId: todo.id, userId });
    navigation.goBack();
  } catch (error) {
    // Must handle errors manually with mutateAsync!
  }
};
```

> **TkDodo:** "Dependent mutations can be solved in 3 ways. Option 1 gives you
> one loading state. Option 3 requires manual error handling. Option 2 with
> callbacks is concise but gets messy with more than 2 mutations."

---

## React Query and Forms

### The Component Split Pattern (Recommended)

```tsx
// Separate data fetching from form state
function EditTodoQuery({ todoId }: { todoId: string }) {
  const { data: todo, isPending, isError } = useQuery(todoQueries.detail(todoId));

  if (isPending) return <LoadingSpinner />;
  if (isError) return <ErrorMessage />;

  return <EditTodoForm initialTodo={todo} />;
}

function EditTodoForm({ initialTodo }: { initialTodo: Todo }) {
  // Form owns its own state, initialized once from server data
  const [title, setTitle] = useState(initialTodo.title);
  const mutation = useUpdateTodo();

  const handleSubmit = () => {
    mutation.mutate({ id: initialTodo.id, title });
  };

  return (
    <View>
      <TextInput value={title} onChangeText={setTitle} />
      <Button title="Save" onPress={handleSubmit} disabled={mutation.isPending} />
    </View>
  );
}
```

> **TkDodo:** "Server State and Client State are different. Once you copy server
> state into form state, React Query's job is done. The form now owns that copy."

### Disable Background Refetches While Editing

```tsx
function EditTodoForm({ todoId }) {
  const [isEditing, setIsEditing] = useState(false);

  const { data } = useQuery({
    ...todoQueries.detail(todoId),
    refetchOnWindowFocus: !isEditing, // Don't refetch while user is editing
  });

  return (
    <TextInput
      onFocus={() => setIsEditing(true)}
      onBlur={() => setIsEditing(false)}
      defaultValue={data?.title}
    />
  );
}
```

---

## DevTools

| Tool | URL |
|------|-----|
| **Native macOS App** | https://github.com/LovesWorking/rn-better-dev-tools |
| **Flipper Plugin** | https://github.com/bgaleotti/react-query-native-devtools |
| **Reactotron Plugin** | https://github.com/hsndmr/reactotron-react-query |

---

## Example Invocations

- "Set up React Query in my Expo app" → Full setup with online/focus managers
- "Add pull-to-refresh to my list" → useRefreshByUser + RefreshControl pattern
- "Why is my query refetching so much?" → Check staleTime, explain defaults
- "How should I structure query keys?" → Query key factory pattern
- "Add optimistic update to like button" → Optimistic mutation with rollback
- "Handle errors globally" → QueryCache.onError with toast
- "Make my app work offline" → networkMode + fetchStatus UI
- "Infinite scroll for my list" → useInfiniteQuery + FlatList pattern
- "Mutation not updating my list" → Invalidation pattern, await explanation
- "Navigate from list to detail fast" → Cache seeding pattern

---

*Based on TkDodo's "Practical React Query" blog series. TkDodo is a core maintainer of TanStack Query.*
*Source: https://tkdodo.eu/blog*
