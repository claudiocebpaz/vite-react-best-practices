---
title: Route-Level Code Splitting
impact: HIGH
tags: code-splitting, react-router, lazy-loading, suspense, performance
---

# Route-Level Code Splitting

Load code only when it is needed. If the router references page components with static imports, the whole application (every page, every modal, every rarely visited admin screen) ends up in the entry chunk that every visitor must download before seeing anything.

## Why it matters

A massive initial bundle delays when the app becomes usable: the browser must download, parse, and execute code for pages the visitor may never open. Splitting by route keeps the first payload small, improves LCP for the entry screen, and reduces main-thread work that would otherwise compete with user interactions (INP).

## Incorrect

Importing every page component at the top level of the route configuration.

```tsx
// App.tsx (declarative mode)
import Dashboard from './pages/Dashboard';
import Settings from './pages/Settings';
import Profile from './pages/Profile';

function App() {
  return (
    <Routes>
      <Route path="/dashboard" element={<Dashboard />} />
      <Route path="/settings" element={<Settings />} />
      <Route path="/profile" element={<Profile />} />
    </Routes>
  );
}
```

All three modules, plus everything they import, land in the same chunk as `App`.

## Correct

### React Router 8 data router: granular `route.lazy`

With `createBrowserRouter`, keep the route *definition* (path, nesting, ids) in the entry bundle and load each *implementation* property on demand. Import `createBrowserRouter` from `react-router` and `RouterProvider` from `react-router/dom` (the DOM entry wires up `flushSync` for you).

```tsx
// src/router.tsx
import { createBrowserRouter } from 'react-router';
import { RootLayout } from './layouts/RootLayout';

export const router = createBrowserRouter([
  {
    path: '/',
    Component: RootLayout,
    children: [
      {
        index: true,
        lazy: {
          Component: async () => (await import('./pages/Home')).Home,
        },
      },
      {
        path: 'dashboard',
        lazy: {
          Component: async () => (await import('./pages/Dashboard')).Dashboard,
          loader: async () => (await import('./pages/dashboard.loader')).dashboardLoader,
        },
      },
      {
        path: 'settings',
        lazy: {
          Component: async () => (await import('./pages/Settings')).Settings,
        },
      },
    ],
  },
]);
```

```tsx
// src/main.tsx
import { createRoot } from 'react-dom/client';
import { RouterProvider } from 'react-router/dom';
import { router } from './router';

createRoot(document.getElementById('root')!).render(<RouterProvider router={router} />);
```

Each `lazy` property is a function that returns the value (or a promise of it), so `Component` and `loader` can live in separate chunks that are fetched in parallel. If you prefer one module per route, return the whole implementation at once:

```tsx
{
  path: 'dashboard',
  lazy: async () => {
    const [page, loaderModule] = await Promise.all([
      import('./pages/Dashboard'),
      import('./pages/dashboard.loader'),
    ]);

    return { Component: page.Dashboard, loader: loaderModule.dashboardLoader };
  },
}
```

Because the definition stays up front, React Router can match the URL synchronously and start loading before rendering, instead of waiting for a module to reveal that a route exists.

### Lazy route discovery

In framework mode (`@react-router/dev`), route discovery is lazy by default: only routes needed for the initial render are in the manifest, and more are fetched from the manifest endpoint as the visitor navigates. `<Link>` and `<NavLink>` targets rendered on the current page are discovered in a batch ahead of time, so the click itself does not pay for the manifest round trip. Configure or disable it in `react-router.config.ts`, and opt individual links out with `<Link to="/admin" discover="none">`.

In data mode the equivalent primitive is the `patchRoutesOnNavigation` option of `createBrowserRouter`, called whenever no route matches a path so you can `patch` the missing subtree into the tree at runtime.

### Transition-friendly navigation

Router state updates are wrapped in `React.startTransition`, so a pending navigation does not block typing or scrolling. Render pending UI from `useNavigation()` and lower the priority of imperative navigations:

```tsx
import { startTransition } from 'react';
import { useNavigate, useNavigation } from 'react-router';

function QueuedSave() {
  const navigate = useNavigate();
  const navigation = useNavigation();

  return (
    <button
      disabled={navigation.state !== 'idle'}
      onClick={() => {
        // The arrow returns the navigation promise, so the Transition
        // lasts until the destination has been rendered.
        startTransition(() => navigate('/dashboard'));
      }}
    >
      {navigation.state === 'idle' ? 'Open dashboard' : 'Loading...'}
    </button>
  );
}
```

`<Link>` and `<Form>` navigations are already Transition-aware; `useNavigate` and `useSubmit` are not, so wrap those yourself and always return or await the promise. `RouterProvider` accepts a `useTransitions` prop: when left `undefined` (the default), router state updates are wrapped in `React.startTransition`; when set to `true`, `Link`/`Form` navigations are also wrapped and in-flight state goes through `useOptimistic` (requires React 19); when `false`, the router does not use Transitions at all. Do not stack your own `startTransition` on top of navigations the router already wraps; the docs warn this can produce buggy behavior, so pick one layer.

### Fallback without a data router: `React.lazy` + `Suspense`

When the app uses declarative routing (or no router at all), split with `React.lazy` and a `Suspense` boundary.

```tsx
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router';

// Module scope only: a lazy component created during render is a new
// component type every render, so React remounts it and resets state.
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Suspense fallback={<div className="spinner">Loading...</div>}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

`React.lazy` resolves the module's `default` export, so pages loaded this way need a default export (the granular `route.lazy` form can pick any named export). A rejected import is thrown to the nearest error boundary, so pair route-level `Suspense` with an error boundary.

Declaring `lazy` inside a component is still the most common mistake: it recreates the component type on every render, resetting state and re-triggering the import. Always declare lazy components at module scope.

React 19 keeps `lazy` and `Suspense` behavior unchanged. The only new lever is prioritization: when a route change can suspend, wrap the update that switches routes in `startTransition` (or `useTransition` when you need an `isPending` flag) so React keeps already revealed content on screen instead of replacing it with the fallback.

## References

- [React Router route objects (`lazy`)](https://reactrouter.com/start/data/route-object#lazy)
- [React Router data mode / custom framework](https://reactrouter.com/start/data/custom)
- [Lazy route discovery](https://reactrouter.com/explanation/lazy-route-discovery)
- [React `lazy` reference](https://react.dev/reference/react/lazy)
