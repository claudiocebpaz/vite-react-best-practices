---
title: Component Colocation and Architecture
impact: MEDIUM
tags: react, architecture, react-router, colocation
---

# Component Colocation and Architecture

Organize code by **Feature**, not by **Technology**. Avoid monolithic `components/` or `hooks/` folders that become dumping grounds.

## Why it matters
- **Discoverability:** Related code is easier to find.
- **Maintainability:** Deleting a feature is safe (delete one folder) vs hunting files across the codebase.
- **Reviewability:** A change that touches one feature folder is easy to scope, review, and revert.
- **Routing boundary:** Route entries stay thin — they map a URL to a feature and contain no business logic.

## Incorrect
Grouping by file type.

```text
src/
  components/
    Header.tsx
    Button.tsx
    LoginForm.tsx
    UserProfile.tsx
  hooks/
    useLogin.ts
    useProfile.ts
  utils/
    authHelpers.ts
```

## Correct
Group by Feature (Domain). Shared UI goes in `components/`. A `routes/` entry owns the URL and renders a feature.

```text
src/
  routes/
    router.tsx          # createBrowserRouter config
    login.tsx           # route entry -> features/Auth
    profile.tsx         # route entry -> features/User
  features/
    Auth/
      LoginForm.tsx
      useLogin.ts
      authApi.ts
      index.ts          # public entry point of the feature
    User/
      UserProfile.tsx
      useProfile.ts
      userApi.ts
      index.ts
  components/ (Shared UI)
    Button.tsx
    Header.tsx
```

A route entry is a React Router route object, not a second implementation of the feature:

```tsx
// src/routes/login.tsx
import type { RouteObject } from 'react-router';
import { LoginForm } from '../features/Auth';

export const loginRoute = {
  path: '/login',
  Component: LoginForm,
} satisfies RouteObject;
```

```tsx
// src/routes/router.tsx
import { createBrowserRouter } from 'react-router';
import { loginRoute } from './login';
import { profileRoute } from './profile';

export const router = createBrowserRouter([loginRoute, profileRoute]);
```

## Rules of thumb
- `features/*/index.ts` is the only entry other folders may import. Everything else inside a feature stays private.
- A feature must not import another feature's internals. Promote the shared piece to `components/` or to an explicit shared feature.
- A route entry contains at most a `loader` / `clientLoader`, an `ErrorBoundary`, and the feature component import.
- Route-level code splitting stays in the route entry (see `react-route-splitting.md`).
- If deleting a folder requires editing more than the router config, the boundary is wrong.

## References
- [React Router: Routing (Data mode)](https://reactrouter.com/start/data/routing)
- [React Router: Route Module](https://reactrouter.com/start/framework/route-module)
- [React: Thinking in React](https://react.dev/learn/thinking-in-react)
- [Vite: Features](https://vite.dev/guide/features)
