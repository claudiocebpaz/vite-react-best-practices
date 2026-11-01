---
title: Separation of Server State
impact: HIGH
tags: server-state, tanstack-query, caching, data-fetching
---

# Separation of Server State

Server state (data that lives in an API) behaves differently from client state (UI interactions). It needs caching, deduplication, background revalidation, and a lifecycle tied to the cache rather than to a component tree.

The examples below use `@tanstack/react-query` (the current 5.x line). SWR covers similar ground with a smaller API; either way, the point is to stop hand-rolling request lifecycles.

## Why it matters

Managing server data with `useEffect` and `useState` leads to:

- **Race conditions:** Responses arriving out of order, so a slow request for the previous id overwrites the fresh one.
- **Waterfalls:** Child components waiting for parents to finish fetching before they can even start.
- **Complexity:** Manually handling loading, error, empty, retry, abort, and refetch states, then repeating it in every component that needs the same data.

## Incorrect

Manual fetching in `useEffect`.

```tsx
function UserProfile() {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/user')
      .then((res) => res.json())
      .then((d) => {
        setData(d);
        setLoading(false);
      });
  }, []); // Only runs once, but what if we need to refetch?

  if (loading) return <div>Loading...</div>;
  return <div>{data.name}</div>;
}
```

There is no abort and no deduplication: two components mounting together issue two requests, a newer response can be overwritten by an older one, and nothing revalidates when the user comes back to the screen. Errors are never represented at all.

## Correct

### 1. Declare the query once with `queryOptions`

```ts
// src/queries/user.ts
import { queryOptions } from '@tanstack/react-query';

export type User = { id: string; name: string; email: string };

export const userOptions = (userId: string) =>
  queryOptions({
    queryKey: ['user', userId],
    queryFn: async ({ signal }) => {
      // Query functions receive an AbortSignal; pass it to fetch so the
      // request is cancelled when the query becomes unused.
      const response = await fetch(`/api/users/${userId}`, { signal });
      if (!response.ok) throw new Error(`Request failed: ${response.status}`);
      return (await response.json()) as User;
    },
    staleTime: 30_000,
    gcTime: 5 * 60_000,
  });
```

`queryOptions` ties the key to the query function and its inferred data type, so the same object is reused by hooks and by imperative APIs such as `queryClient.query`.

### 2. Read it with the options-object `useQuery`

```tsx
import { useQuery } from '@tanstack/react-query';
import { userOptions } from '../queries/user';

function UserProfile({ userId }: { userId: string }) {
  const { data, isPending, isError, error } = useQuery(userOptions(userId));

  if (isPending) return <div>Loading...</div>;
  if (isError) return <div>Error: {error.message}</div>;

  return <div>{data.name}</div>;
}
```

`useQuery` takes a single options object; the older multi-argument signature is gone. `isPending` means "no data yet", `isFetching` means a request is in flight (including background refetches), and `isLoading` is `isPending && isFetching`.

### 3. Suspense: `useSuspenseQuery` behind a boundary

```tsx
import { Suspense } from 'react';
import { useSuspenseQuery } from '@tanstack/react-query';

function UserName({ userId }: { userId: string }) {
  const { data } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  return <h1>{data.name}</h1>; // data is never undefined on the type level
}

export function UserHeader({ userId }: { userId: string }) {
  return (
    <Suspense fallback={<UserSkeleton />}>
      <UserName userId={userId} />
    </Suspense>
  );
}
```

The loading state becomes the boundary's `fallback` and errors go to the nearest error boundary. Use `QueryErrorResetBoundary` with `useQueryErrorResetBoundary` so a "Try again" button resets the failed queries. Two constraints to remember: suspense hooks have no `placeholderData`, and cancellation does not apply to them. To avoid replacing visible data with the fallback when the key changes, wrap the update in `startTransition`.

### 4. Keep the previous page visible while the next one loads

```tsx
function ProjectList({ page }: { page: number }) {
  const { data, isPlaceholderData } = useQuery({
    queryKey: ['projects', page],
    queryFn: () => fetchProjects(page),
    placeholderData: (previousData) => previousData,
  });

  return (
    <ul style={{ opacity: isPlaceholderData ? 0.5 : 1 }}>
      {data.map((project) => (
        <li key={project.id}>{project.name}</li>
      ))}
    </ul>
  );
}
```

Returning the previous data from `placeholderData` keeps the list rendered while the next page loads, which removes the spinner flash on pagination. The old option that carried the previous page's data, and its `isPreviousData` flag, no longer exist; this identity function plus `isPlaceholderData` is the current API.

### 5. Know what the two timers do

- `staleTime`: how long data is considered fresh. While fresh, no refetch happens on mount, window focus, or reconnect.
- `gcTime`: how long an unused query stays in memory before it is garbage collected. It only starts counting once no component observes the query.

This is the current name for the old cache-time setting. After a mutation, invalidate the affected keys instead of editing the cache by hand:

```ts
queryClient.invalidateQueries({ queryKey: ['user', userId] });
```

## References

- [TanStack Query overview](https://tanstack.com/query/latest/docs/framework/react/overview)
- [Suspense guide](https://tanstack.com/query/latest/docs/framework/react/guides/suspense)
- [Query cancellation](https://tanstack.com/query/latest/docs/framework/react/guides/query-cancellation)
- [`queryOptions` reference](https://tanstack.com/query/latest/docs/framework/react/reference/queryOptions)
