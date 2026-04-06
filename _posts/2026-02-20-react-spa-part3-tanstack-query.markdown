---
layout: post
title: "Server State with TanStack Query"
date: 2026-02-20 00:00:00 +0000
categories: [React]
tags:
  - react
  - tanstack-query
  - server-state
  - typescript
series: "Building Modern React SPAs"
series_part: 3
---


## Series Overview

1. [TypeScript Patterns for React](/posts/react-spa-part1-typescript-patterns/)
2. [Type-Safe Routing with TanStack Router](/posts/react-spa-part2-tanstack-router/)
3. **Server State with TanStack Query** (this post)
4. [Styling with Tailwind CSS v4 & shadcn/ui](/posts/react-spa-part4-tailwind-shadcn/)
5. [Modular Feature Architecture](/posts/react-spa-part5-architecture/)
6. [Deploy to AWS with S3, CloudFront & CDK](/posts/react-spa-part6-deploy-aws/)

## Introduction

Traditional React data fetching uses `useEffect` + `useState` + loading/error flags. This leads to duplicated fetch logic, no caching, no background updates, manual state management, and race conditions on rapid navigations. TanStack Query replaces all of this with a declarative, cache-first approach.

The core mental model is straightforward: TanStack Query is an **async state manager**, not a data fetcher. You provide the Promise; it manages caching, deduplication, background refetching, garbage collection, retry logic, and reactive state updates automatically.

This post covers installation, the critical `staleTime` vs `gcTime` distinction, query keys, the `queryOptions` pattern, reading data, writing data with mutations, optimistic updates, router integration, infinite queries, and testing.

## Installation and Setup

```bash
npm install @tanstack/react-query
npm install -D @tanstack/react-query-devtools
```

### QueryClient and Provider

Create a `QueryClient` and wrap your app in `QueryClientProvider`:

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60,
      gcTime: 1000 * 60 * 5,
      refetchOnWindowFocus: false,
      retry: 3,
      retryDelay: (attempt) =>
        Math.min(1000 * 2 ** attempt, 30000),
    },
  },
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <YourApp />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

Set `staleTime` and `gcTime` globally, then override per-query where needed. Common recommended defaults for SPAs:

| Setting | Library Default | Recommended SPA Default | Why |
|---------|----------------|------------------------|-----|
| `staleTime` | `0` | `60_000` (1 min) | Prevents unnecessary refetches on mount |
| `gcTime` | `300_000` (5 min) | `300_000` (5 min) | Keeps inactive cache for back-navigation |
| `refetchOnWindowFocus` | `true` | `false` | SPAs do not need aggressive refetching |
| `retry` | `3` | `3` | Handles transient network failures |

## Core Concepts: staleTime vs gcTime

These two settings control how TanStack Query manages cached data. Understanding them is essential.

### staleTime: How Long Is This Data Fresh?

While data is **fresh**, TanStack Query serves it from cache without any network request -- even if the component remounts, the user navigates away and back, or the window regains focus.

Once `staleTime` elapses, data becomes **stale**. Stale data is still served from cache instantly, but a background refetch is triggered under certain conditions (component mount, window focus, network reconnect, or interval).

### gcTime: How Long to Keep Unused Data in Memory?

When a query has no active observers (no component is using it), a timer starts. After `gcTime` elapses, the cached entry is garbage collected.

If the component remounts before `gcTime` expires, the cached data is available instantly (and a background refetch may occur if data is stale).

### The Relationship

```
fetch --- fresh (staleTime) --- stale --- component unmounts --- gcTime expires --- deleted
                                  |                                  |
                                  +-- background refetch             +-- next mount = fresh fetch
```

**Rule of thumb:** Keep `gcTime >= staleTime`. This ensures cached data is still available when it goes stale, enabling the "instant cache + background refetch" pattern.

### Per-Query Configuration Examples

```tsx
// Static data (rarely changes)
useQuery({
  queryKey: ['countries'],
  queryFn: fetchCountries,
  staleTime: Infinity,
  gcTime: Infinity,
});

// User profile (moderate freshness)
useQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
  staleTime: 30_000,
  gcTime: 300_000,
});

// Live stock price (very short staleTime)
useQuery({
  queryKey: ['stock', ticker],
  queryFn: () => fetchStockPrice(ticker),
  staleTime: 5_000,
  refetchInterval: 10_000,
});
```

## Query Keys: Cache Identity

Query keys are the cache identity for your data. Two queries with the same key share the same cache entry. Keys are serialized deterministically, so object property order does not matter.

### Query Key Factory Pattern (Recommended)

Create a factory object that centralizes all your query keys. This prevents key typos and makes cache invalidation predictable:

```tsx
export const userKeys = {
  all:     ['users'] as const,
  lists:   () => [...userKeys.all, 'list'] as const,
  list:    (filters: UserFilters) => [...userKeys.lists(), filters] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail:  (id: string) => [...userKeys.details(), id] as const,
};

export const postKeys = {
  all:     ['posts'] as const,
  lists:   () => [...postKeys.all, 'list'] as const,
  list:    (filters: PostFilters) => [...postKeys.lists(), filters] as const,
  details: () => [...postKeys.all, 'detail'] as const,
  detail:  (id: string) => [...postKeys.details(), id] as const,
};
```

Usage in queries and invalidation:

```tsx
// Fetch a single user
useQuery({ queryKey: userKeys.detail(userId), queryFn: ... });

// Invalidate all user queries (lists + details)
queryClient.invalidateQueries({ queryKey: userKeys.all });

// Invalidate only user lists (not details)
queryClient.invalidateQueries({ queryKey: userKeys.lists() });
```

## The queryOptions Pattern

TanStack Query v5 introduced `queryOptions()` -- a helper that bundles `queryKey`, `queryFn`, and options into a single reusable object. This is the recommended way to define queries.

```tsx
import { queryOptions } from '@tanstack/react-query';

export function userDetailOptions(userId: string) {
  return queryOptions({
    queryKey: userKeys.detail(userId),
    queryFn: () => fetchUser(userId),
    staleTime: 30_000,
  });
}

export function userListOptions(filters: UserFilters) {
  return queryOptions({
    queryKey: userKeys.list(filters),
    queryFn: () => fetchUsers(filters),
    staleTime: 60_000,
  });
}
```

### Why This Matters

The same `queryOptions` object can be used in three different places:

```tsx
// 1. In a TanStack Router loader (prefetch)
export const Route = createFileRoute('/users/$userId')({
  loader: ({ params, context: { queryClient } }) =>
    queryClient.ensureQueryData(userDetailOptions(params.userId)),
  component: UserPage,
});

// 2. In a component (reactive cache read + background refetch)
function UserPage() {
  const { userId } = Route.useParams();
  const { data: user } = useQuery(userDetailOptions(userId));
}

// 3. In a prefetch-on-hover handler
<Link
  to="/users/$userId"
  params={{ userId: '42' }}
  onMouseEnter={() =>
    queryClient.prefetchQuery(userDetailOptions('42'))
  }
>
  View User
</Link>
```

Inline `useQuery({ queryKey, queryFn })` works, but duplicates key-function pairs. `queryOptions` creates a single source of truth that ensures the `queryKey` and `queryFn` are always paired correctly.

## Queries: Reading Data

### Basic Query

```tsx
import { useQuery } from '@tanstack/react-query';

function UserProfile({ userId }: { userId: string }) {
  const {
    data,        // The fetched data (typed from queryFn return)
    isPending,   // True on first load (no cached data)
    isError,     // True if queryFn threw
    error,       // The error object
    isStale,     // True after staleTime elapses
    isFetching,  // True during any fetch (including background)
    refetch,     // Manual refetch function
  } = useQuery(userDetailOptions(userId));

  if (isPending) return <Spinner />;
  if (isError) return <ErrorMessage error={error} />;

  return <div>{data.name}</div>;
}
```

### Dependent (Serial) Queries

When one query depends on another's result:

```tsx
const { data: user } = useQuery(userDetailOptions(userId));

const { data: projects } = useQuery({
  queryKey: ['projects', user?.orgId],
  queryFn: () => fetchProjectsByOrg(user!.orgId),
  enabled: !!user?.orgId,
});
```

### Parallel Queries

Multiple independent queries run in parallel automatically:

```tsx
const userQuery = useQuery(userDetailOptions(userId));
const postsQuery = useQuery(postListOptions({ authorId: userId }));
// Both fetch simultaneously
```

## Mutations: Writing Data

Mutations handle create, update, and delete operations. They are not cached -- they run only when you call `mutate()` or `mutateAsync()`.

```tsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

function CreatePostForm() {
  const queryClient = useQueryClient();

  const createPost = useMutation({
    mutationFn: (newPost: NewPost) =>
      fetch('/api/posts', {
        method: 'POST',
        body: JSON.stringify(newPost),
      }).then((res) => res.json()),

    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: postKeys.lists() });
    },
    onError: (error) => {
      toast.error(`Failed: ${error.message}`);
    },
  });

  return (
    <button
      onClick={() => createPost.mutate({ title: 'New Post', body: '...' })}
      disabled={createPost.isPending}
    >
      {createPost.isPending ? 'Creating...' : 'Create Post'}
    </button>
  );
}
```

### mutate() vs mutateAsync()

- `mutate()` -- Fire-and-forget. Errors go to the mutation's `error` state. Use this for most form submissions.
- `mutateAsync()` -- Returns a Promise. Use when you need to `await` the result before doing something else (e.g., navigate after creation).

```tsx
const handleSubmit = async (data: NewPost) => {
  const created = await createPost.mutateAsync(data);
  navigate({ to: '/posts/$postId', params: { postId: created.id } });
};
```

## Optimistic Updates

Show the result immediately, then reconcile when the server responds:

```tsx
const updateTodo = useMutation({
  mutationFn: (updatedTodo: Todo) => patchTodo(updatedTodo),
  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({ queryKey: todoKeys.detail(newTodo.id) });
    const previousTodo = queryClient.getQueryData(todoKeys.detail(newTodo.id));
    queryClient.setQueryData(todoKeys.detail(newTodo.id), newTodo);
    return { previousTodo };
  },
  onError: (_err, _newTodo, context) => {
    if (context?.previousTodo) {
      queryClient.setQueryData(
        todoKeys.detail(context.previousTodo.id),
        context.previousTodo
      );
    }
  },
  onSettled: (_data, _err, variables) => {
    queryClient.invalidateQueries({ queryKey: todoKeys.detail(variables.id) });
  },
});
```

Use optimistic updates for actions where the user expects instant feedback: toggling a favorite, marking a task complete, updating a name. Avoid them for complex operations where server-side validation might reject the change.

## Prefetching and Router Integration

### Prefetch in TanStack Router Loaders

The recommended pattern: prefetch in the route loader, read in the component.

```tsx
// Route definition
export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params, context: { queryClient } }) =>
    queryClient.ensureQueryData(postDetailOptions(params.postId)),
  component: PostPage,
});

// Component
function PostPage() {
  const { postId } = Route.useParams();
  const { data } = useQuery(postDetailOptions(postId));
  // data is available INSTANTLY from the loader's cache
}
```

**`ensureQueryData` vs `prefetchQuery`:**

- `ensureQueryData` -- Returns the data. If stale, fetches fresh data. If already fresh in cache, returns cached data without a network call.
- `prefetchQuery` -- Returns `void`. Fires a fetch if stale, but does not wait. Useful for fire-and-forget prefetching on hover.

### Prefetch on Hover

With `defaultPreload: 'intent'` on the router, all `<Link>` components automatically prefetch route data and code on hover/focus.

## Infinite Queries (Pagination)

For load-more or infinite scroll patterns:

```tsx
import { useInfiniteQuery } from '@tanstack/react-query';

function PostsFeed() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['posts', 'infinite'],
    queryFn: ({ pageParam }) =>
      fetch(`/api/posts?cursor=${pageParam}`).then(r => r.json()),
    initialPageParam: '',
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
  });

  const allPosts = data?.pages.flatMap((page) => page.posts) ?? [];

  return (
    <div>
      {allPosts.map((post) => (
        <PostCard key={post.id} post={post} />
      ))}
      {hasNextPage && (
        <button
          onClick={() => fetchNextPage()}
          disabled={isFetchingNextPage}
        >
          {isFetchingNextPage ? 'Loading...' : 'Load More'}
        </button>
      )}
    </div>
  );
}
```

## Testing

Use Mock Service Worker (MSW) to intercept network requests in tests. Always create a new `QueryClient` for each test to avoid shared cache state.

```tsx
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

function createWrapper() {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
    },
  });
  return ({ children }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}

test('fetches user data', async () => {
  const { result } = renderHook(
    () => useQuery(userDetailOptions('1')),
    { wrapper: createWrapper() }
  );

  await waitFor(() => expect(result.current.isSuccess).toBe(true));
  expect(result.current.data?.name).toBe('Alice');
});
```

## Common Anti-Patterns to Avoid

### Using useEffect to Fetch Data

```tsx
// Don't do this
const [data, setData] = useState(null);
useEffect(() => {
  fetch('/api/users').then(r => r.json()).then(setData);
}, []);

// Do this instead
const { data } = useQuery({
  queryKey: ['users'],
  queryFn: () => fetch('/api/users').then(r => r.json()),
});
```

### Putting Server State in Redux/Zustand

TanStack Query IS your server-state manager. Use Zustand or Redux only for client state (UI toggles, form state, theme preferences).

### Forgetting to Set staleTime

With the default `staleTime: 0`, every component mount triggers a background refetch. For most SPAs, setting `staleTime` to 30-60 seconds globally prevents excessive network requests.

### Using the Same Query Key for Different Data

```tsx
// Bad: both return different data but share a cache entry
useQuery({ queryKey: ['data'], queryFn: fetchUsers });
useQuery({ queryKey: ['data'], queryFn: fetchPosts });

// Correct: distinct keys for distinct data
useQuery({ queryKey: ['users'], queryFn: fetchUsers });
useQuery({ queryKey: ['posts'], queryFn: fetchPosts });
```

## Conclusion

TanStack Query transforms how you think about server data in React. Instead of imperatively fetching and managing loading states, you declare what data you need and let the library handle the rest: caching, deduplication, background updates, retries, and garbage collection. Combined with the `queryOptions` pattern and TanStack Router's loader integration, you get instant navigations with always-fresh data.

In the next post, we will build the visual layer of our SPA using Tailwind CSS v4 and shadcn/ui -- a design system that gives you beautiful, accessible components with full theme control.

## References

- [TanStack Query Documentation](https://tanstack.com/query/latest/docs/framework/react/overview)
- [Important Defaults](https://tanstack.com/query/latest/docs/framework/react/guides/important-defaults)
- [Query Keys Guide](https://tanstack.com/query/latest/docs/framework/react/guides/query-keys)
- [Caching Guide](https://tanstack.com/query/latest/docs/framework/react/guides/caching)
- [Mutations Guide](https://tanstack.com/query/latest/docs/framework/react/guides/mutations)
- [Optimistic Updates](https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates)
- [Prefetching Guide](https://tanstack.com/query/latest/docs/framework/react/guides/prefetching)
- [Testing Guide](https://tanstack.com/query/latest/docs/framework/react/guides/testing)
