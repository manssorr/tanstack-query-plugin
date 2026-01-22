# TanStack Query Plugin for Claude Code

Expert-level TanStack Query (React Query) patterns and official API documentation.

**Two skills included:**
1. **Patterns Skill** - TkDodo's "Practical React Query" best practices (975 lines)
2. **Docs Skill** - Official v5 API reference (450+ lines)

## Installation

```bash
/plugin install tanstack-query@manssorr/tanstack-query-plugin
```

## Usage

```bash
/tanstack-query how do I set up infinite scroll for FlatList?
/tanstack-query fix my mutation that's not updating the cache
/tanstack-query what's the best staleTime for user profiles?
```

## What's Included

### Skill 1: Patterns (975 lines)

Based on TkDodo's "Practical React Query" blog series.

| Topic | Coverage |
|-------|----------|
| **Setup** | React Native/Expo setup with online/focus managers |
| **Query Keys** | Factory pattern with `queryOptions` |
| **Mutations** | Invalidation, direct updates, `mutate` vs `mutateAsync` |
| **Optimistic Updates** | When to use, rollback patterns |
| **Error Handling** | Local, Error Boundaries, global toast |
| **Offline Support** | `networkMode`, `fetchStatus` UI |
| **Infinite Queries** | `useInfiniteQuery` + FlatList |
| **Testing** | MSW setup, test utilities |
| **Forms** | Component split pattern |
| **Decision Trees** | When to use which pattern |

### Skill 2: API Docs (450+ lines)

Official TanStack Query v5 API reference.

| Topic | Coverage |
|-------|----------|
| **useQuery** | All options, return values, examples |
| **useMutation** | Lifecycle callbacks, per-call options |
| **useInfiniteQuery** | Pagination, cursor-based fetching |
| **QueryClient** | invalidateQueries, setQueryData, prefetchQuery |
| **Optimistic Updates** | Full pattern with rollback |
| **TypeScript** | Typed queries, mutations, context |
| **Network Mode** | online, always, offlineFirst |
| **Global Callbacks** | QueryCache, MutationCache handlers |

### TkDodo Author Insights

The skill includes 14+ direct quotes from TkDodo explaining the "why" behind patterns:

> **TkDodo:** "Queries are declarative. If you have some state that changes your data,
> all you need to do is put it in the Query Key, because React Query will trigger
> a refetch automatically whenever the key changes."

## Example Patterns

### Query Key Factory

```tsx
export const todoKeys = {
  all: ['todos'] as const,
  lists: () => [...todoKeys.all, 'list'] as const,
  list: (filters: TodoFilters) => [...todoKeys.lists(), filters] as const,
  details: () => [...todoKeys.all, 'detail'] as const,
  detail: (id: string) => [...todoKeys.details(), id] as const,
};
```

### Pull-to-Refresh Hook

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
```

## Source

Content scraped and curated from:
- [TkDodo's Blog](https://tkdodo.eu/blog) - 38 posts
- [GitHub Discussions](https://github.com/TkDodo/blog-comments/discussions) - 2,970 comments

## License

MIT
