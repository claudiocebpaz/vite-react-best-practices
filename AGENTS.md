# Vite React Best Practices

Comprehensive guide for building production-ready React Single Page Applications (SPAs) with Vite. This document is the compiled form of the rules in `rules/`; each section mirrors one rule file and keeps its frontmatter impact level.

## Stack Baseline

The guidance targets the current stable toolchain:

| Tool | Version |
| --- | --- |
| React | 19.3.1 |
| Vite | 8.3.5 (Rolldown + Oxc) |
| React Router | 8.5.1 |
| TanStack Query | 5.105.1 (v5 line) |
| React Compiler | `babel-plugin-react-compiler` 1.x |
| Node (required by Vite 8) | `^20.19.0 || >=22.12.0` |

## Table of Contents

1. [Vite SPA Deployment (CRITICAL)](#vite-spa-deployment-critical)
   - [Configure Static Host Rewrites for SPA Routing](#configure-static-host-rewrites-for-spa-routing) — CRITICAL
   - [Correct Caching Strategy for Vite Assets](#correct-caching-strategy-for-vite-assets) — CRITICAL
   - [Validate Production Builds Locally](#validate-production-builds-locally) — HIGH
   - [Environment Variables in Vite](#environment-variables-in-vite) — HIGH
2. [React Core Performance](#react-core-performance)
   - [Route-Level Code Splitting](#route-level-code-splitting) — HIGH
   - [Separation of Server State](#separation-of-server-state) — HIGH
   - [Strategic Memoization](#strategic-memoization) — MEDIUM
   - [Image Optimization for CLS and Performance](#image-optimization-for-cls-and-performance) — MEDIUM
3. [Architecture & Cleanup](#architecture--cleanup)
   - [Component Colocation and Architecture](#component-colocation-and-architecture) — MEDIUM
   - [Anti-Pattern: Importing from Dist](#anti-pattern-importing-from-dist) — MEDIUM
   - [Common Vite Troubleshooting](#common-vite-troubleshooting) — MEDIUM

---

# Vite SPA Deployment (CRITICAL)

The host, not the application code, decides whether a Vite SPA survives a page refresh, a new deployment, or a change of environment. These four rules cover the deployment surface: routing fallbacks, cache headers, build verification, and environment variables.

## Configure Static Host Rewrites for SPA Routing

**Impact:** CRITICAL · **Tags:** vite, spa, routing, deployment, static-hosting

A Vite SPA is a folder of static files. Every route below `/` (for example `/dashboard`) exists only in the browser, handled by React Router. When a user opens or refreshes that URL, the browser asks the host for that path, and the host looks for a file that does not exist.

### Why it matters
Failing to configure rewrites is the most common cause of "404 Not Found" (or "403 Forbidden" on S3) after a page refresh. It breaks deep linking, shared URLs, bookmarks, and any crawler that follows an internal link. The application code is fine; the host simply never hands `index.html` to the browser.

### Incorrect
Deploying the `dist/` output to a static host with its default behavior: serve the requested path, return an error when no file matches.

```nginx
# Nginx default (Incorrect for SPA)
location / {
  root /var/www/html;
  index index.html;
  # Returns 404 if the requested file does not exist
}
```

### Correct
Configure the host to serve `index.html` with a `200` status for any path that does not map to a real file. Real files (JS, CSS, images, fonts) must still be served directly, otherwise the app cannot boot.

#### Nginx
`try_files` checks the filesystem first and falls back to the entry point.

```nginx
location / {
  root /var/www/html;
  try_files $uri $uri/ /index.html;
}
```

#### Netlify (`netlify.toml`)
A rule with `status = 200` is a rewrite, not a redirect: the URL in the address bar does not change.

```toml
[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

#### Vercel (`vercel.json`)
```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}
```

#### Firebase (`firebase.json`)
```json
{
  "hosting": {
    "public": "dist",
    "rewrites": [{ "source": "**", "destination": "/index.html" }]
  }
}
```

#### Cloudflare Pages (`_redirects`)
Create `public/_redirects` so Vite copies it to the root of `dist/`. The trailing `200` turns the rule into a proxy/rewrite.

```bash
# public/_redirects
/* /index.html 200
```

#### AWS S3 + CloudFront
Upload the contents of `dist/` to the bucket, set the distribution's default root object to `index.html`, and add **custom error responses** for both `403` and `404` that return `/index.html` with an HTTP response code of `200`. CloudFront caches error responses for the error caching minimum TTL (10 seconds by default), so keep that value low while iterating.

```yaml
# CloudFormation fragment: AWS::CloudFront::Distribution DistributionConfig
CustomErrorResponses:
  - ErrorCode: 403
    ResponseCode: 200
    ResponsePagePath: /index.html
    ErrorCachingMinTTL: 10
  - ErrorCode: 404
    ResponseCode: 200
    ResponsePagePath: /index.html
    ErrorCachingMinTTL: 10
```

Mapping both codes matters: depending on the origin type and bucket permissions, S3 can answer a missing key with `403` instead of `404`.

#### GitHub Pages
GitHub Pages serves [`404.html`](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-custom-404-page-for-your-github-pages-site) for unknown paths, so publish a copy of the built entry point under that name. Add the copy step after the build:

```bash
vite build
cp dist/index.html dist/404.html
```

Also set `base` to `/<REPO>/` when the site is served from a project subpath such as `https://<USERNAME>.github.io/<REPO>/`. The response still carries a `404` status code, which is acceptable for client-rendered apps but not for crawlers that require a `200`.

#### Azure Static Web Apps (`staticwebapp.config.json`)
`navigationFallback` returns the entry point for requests that do not match a deployed file. Route rules are not applied to requests handled by the fallback.

```json
{
  "navigationFallback": {
    "rewrite": "/index.html",
    "exclude": ["/assets/*"]
  }
}
```

### References
- [Vite: Deploying a Static Site](https://vite.dev/guide/static-deploy)
- [Cloudflare Pages: Redirects](https://developers.cloudflare.com/pages/configuration/redirects/)
- [Amazon CloudFront: Change response codes returned by CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/custom-error-pages-response-code.html)
- [Azure Static Web Apps: Configuration](https://learn.microsoft.com/en-us/azure/static-web-apps/configuration)

---

## Correct Caching Strategy for Vite Assets

**Impact:** CRITICAL · **Tags:** vite, caching, cdn, performance, deployment

Vite 8 emits content-hashed file names by default: `assets/[name]-[hash].js` for entry chunks and `assets/[name]-[hash].[ext]` for other assets. The hash is derived from the file contents, so a deployment that changes a file also changes its URL. That property is what makes an aggressive cache policy safe.

### Why it matters
Incorrect caching produces two opposite failures:
1. **Users see old code** after a deployment, because `index.html` was cached and still points to the previous build.
2. **Slow loading**, because hashed assets that can never be reused under the same URL are revalidated on every visit.

A single policy applied to both file types cannot solve both problems.

### Incorrect
Applying the same cache policy to every file.

```http
# Incorrect: caching everything for a week
Cache-Control: public, max-age=604800
```

### Correct
Split the files into two categories with different policies.

#### 1. The entry point (`index.html`)
This file is the pointer to the current build. It must not be cached, or must be revalidated on every request.

```http
Cache-Control: no-cache, no-store, must-revalidate
```

At most, use `Cache-Control: public, max-age=0, must-revalidate`. Vite's build guide makes the same point for the missing-chunk case below: without `Cache-Control: no-cache` on the HTML, clients keep referencing deleted assets.

#### 2. Hashed assets (`/assets/*.js`, `/assets/*.css`, images, fonts)
The file name changes whenever the content changes, so these can be cached for a year and marked immutable.

```http
Cache-Control: public, max-age=31536000, immutable
```

If you change `build.assetsDir` (default `assets`), update the host and CDN rules to match the new directory.

#### Make the CDN cache key include the hashed file name
The policy above only works if the cache key distinguishes one hash from another. Keep the full request path in the cache key. A rule that normalizes `/assets/*` to one key, strips the file name, or versions assets only through a query string will serve a previous build's JavaScript under the current HTML, which fails at runtime with a chunk that no longer exists.

Do not rewrite hashed file names on upload either. The HTML references the exact names produced by the build.

#### Keep the modulepreload tags for the entry chunk
Vite injects `<link rel="modulepreload">` tags for the entry chunk and its static imports, and adds the module preload polyfill by default (`build.modulePreload`). If you post-process `index.html` through an HTML minifier or a CDN transform, do not strip those tags: they start fetching the entry chunk before the parser reaches the module script. The preloaded URLs are hashed files and need the same immutable header as any other asset.

#### Recover from chunks removed by a new deployment
Even with correct headers, a user with an open tab can request a chunk that the latest deployment deleted. Vite emits a `vite:preloadError` event for this case.

```ts
window.addEventListener('vite:preloadError', (event) => {
  event.preventDefault()
  window.location.reload()
})
```

### References
- [Vite: Building for Production](https://vite.dev/guide/build)
- [Vite: Build Options](https://vite.dev/config/build-options)
- [MDN: Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)

---

## Validate Production Builds Locally

**Impact:** HIGH · **Tags:** vite, build, ci, typescript, deployment

Vite 8 builds with Rolldown and transforms JavaScript with Oxc, replacing Rollup and esbuild. The dev server serves source modules on demand and pre-bundles dependencies with Rolldown, while `vite build` runs the full bundling, transformation, and minification pipeline. Development and production are therefore different programs operating on the same source.

### Why it matters
Problems that survive `npm run dev` but break the deployed app include:
- Case-sensitive import paths that resolve on a case-insensitive filesystem but not on Linux CI.
- Files referenced from `public/` with a wrong base path, or assets missing from `dist/`.
- Code that only fails after minification or tree-shaking (side-effect imports removed, `process.env` left untouched).
- Builds that ran in a different mode, and therefore inlined a different set of `import.meta.env` values.

`vite build` does not typecheck. Without a separate `tsc` run, type errors ship to production.

### Incorrect
Verifying only the dev server, then pushing.

```bash
# Don't do this only
npm run dev
git commit -m "It works on my machine"
git push
```

### Correct
Run typecheck, build, and preview before pushing. Wire the same commands into CI.

```json
{
  "scripts": {
    "dev": "vite",
    "typecheck": "tsc --noEmit",
    "build": "vite build",
    "preview": "vite preview --port 8080"
  }
}
```

1. **Typecheck first.** `tsc --noEmit` catches type errors with `strict` enabled and fails faster than a full bundle.
2. **Build.** `vite build` writes the production output to `dist/` (Rolldown, Oxc transform, Oxc minify, Lightning CSS).
3. **Preview.** `vite preview` serves `dist/` locally. The default port is `4173`; `vite preview --port 8080` pins a different one. This server is for verification only and is not intended to serve production traffic.
4. **Exercise the built app.** Navigate between routes, refresh on a deep link, and open DevTools to confirm that the hashed assets in `dist/` load without 404s.

`vite preview` applies Vite's own SPA fallback when `appType` is `spa`, so it cannot prove that the production host has rewrite rules. Check at least one deep link against the deployed host as well.

#### Minimal CI workflow
The workflow installs from the lockfile, typechecks, and builds. It intentionally does not deploy.

```yaml
name: build
on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with:
          node-version: lts/*
          cache: npm
      - run: npm ci
      - run: npx tsc --noEmit
      - run: npx vite build
```

Run CI on Linux even if the team develops on macOS or Windows: that is where case-sensitive imports and missing files surface.

### References
- [Vite: Building for Production](https://vite.dev/guide/build)
- [Vite: Command Line Interface](https://vite.dev/guide/cli)
- [Vite: Migration from v7 (Rolldown and Oxc)](https://vite.dev/guide/migration)
- [GitHub Actions: Workflow syntax](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions)

---

## Environment Variables in Vite

**Impact:** HIGH · **Tags:** vite, env, configuration, security, typescript

Vite exposes configuration on `import.meta.env`. Values from `.env` files are parsed at build time and statically replaced in the output, so the browser bundle contains literals, not a lookup table. Only `VITE_`-prefixed variables from those files reach client code.

### Why it matters
- **Secret leakage.** Anything prefixed with `VITE_` is bundled into JavaScript that every visitor can read. An API key placed there is published.
- **Stale expectations.** Values are inlined when the bundle is built. Changing a variable in the hosting dashboard does nothing until the app is rebuilt, which surprises teams used to server-side runtimes.

### Incorrect
Using `process.env`, or expecting an unprefixed variable to be available in the browser.

```tsx
// Incorrect: Vite does not polyfill process.env for client code
console.log(process.env.API_KEY)

// Incorrect: not exposed, because it lacks the VITE_ prefix
console.log(import.meta.env.SECRET_KEY)
```

### Correct
Prefix public values with `VITE_` and read them from `import.meta.env`.

```env
# .env
VITE_API_URL=https://api.example.com
VITE_FEATURE_FLAG=true

# Not exposed to the client bundle
DB_PASSWORD=supersecret
```

```tsx
// App.tsx
const apiUrl: string = import.meta.env.VITE_API_URL
const featureEnabled: boolean = import.meta.env.VITE_FEATURE_FLAG === 'true'
```

Values are always strings, including numbers and booleans, so convert them explicitly. If you need a different prefix, set `envPrefix` in `vite.config.ts`; the `VITE_` default exists to make accidental exposure unlikely, so changing it weakens that guard.

#### `.env` files and modes
Vite loads these files from the environment directory, in increasing priority:

```bash
.env                # loaded in all cases
.env.local          # loaded in all cases, ignored by git
.env.[mode]         # only loaded in the specified mode
.env.[mode].local   # only loaded in the specified mode, ignored by git
```

Mode-specific files override generic ones, and variables already present in the process environment (for example `VITE_API_URL=... vite build`) override all files. The dev command runs in `development` mode and the build command in `production` mode. Build for another mode explicitly:

```bash
vite build --mode staging
```

That command loads `.env.staging`. `.env` files are read when Vite starts, so restart the dev server after editing one. Add `*.local` to `.gitignore`.

#### Built-in flags
These constants are available in every app and do not need a `VITE_` prefix:

```ts
// The mode the app was started with: 'development', 'production', 'staging', ...
const mode: string = import.meta.env.MODE

// NODE_ENV-derived booleans, not the mode string
if (import.meta.env.DEV) {
  console.log(`starting in ${mode} mode`)
}

const logLevel = import.meta.env.PROD ? 'error' : 'debug'

// True only in a server environment; always false in a browser-only SPA
if (import.meta.env.SSR) {
  // server-only branch
}
```

`import.meta.env.BASE_URL` returns the configured `base`, which is useful when building URLs at runtime.

#### Runtime configuration
Because values are inlined, per-environment settings that must change without a rebuild need an external source. A common pattern is a JSON file fetched at boot, before the app renders:

```json
{
  "apiUrl": "https://api.example.com",
  "sentryDsn": "https://example.invalid/1"
}
```

```ts
// src/config.ts
export type RuntimeConfig = {
  apiUrl: string
  sentryDsn?: string
}

let cached: RuntimeConfig | undefined

export async function loadRuntimeConfig(): Promise<RuntimeConfig> {
  if (!cached) {
    // BASE_URL keeps this correct for subpath deployments
    const response = await fetch(`${import.meta.env.BASE_URL}config.json`, {
      cache: 'no-cache',
    })
    if (!response.ok) {
      throw new Error(`Failed to load runtime config: ${response.status}`)
    }
    cached = (await response.json()) as RuntimeConfig
  }
  return cached
}
```

Place `config.json` in `public/` so it is copied to `dist/` unchanged and is not hashed, and serve it with revalidation rather than a long-lived cache. Validate the parsed shape before trusting it; a plain cast does not check anything at runtime.

#### Type safety
Augment `ImportMetaEnv` in `src/vite-env.d.ts` so `strict` TypeScript knows which variables exist.

```ts
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string
  readonly VITE_FEATURE_FLAG: string
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

Keep the file free of `import` statements, otherwise the interfaces become module-scoped and the augmentation stops applying. Vite also supports making `ImportMetaEnv` strict, so unknown keys are rejected, through the `ViteTypeOptions` interface documented on the env page.

### References
- [Vite: Env Variables and Modes](https://vite.dev/guide/env-and-mode)
- [Vite: Shared Options (`envPrefix`)](https://vite.dev/config/shared-options)
- [MDN: import.meta](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import.meta)

---

# React Core Performance

These rules cover the rendering and data-fetching decisions that dominate the initial load and the responsiveness of a React 19 application: how code is split per route, how server state is cached, when memoization is worth it, and how images are delivered.

## Route-Level Code Splitting

**Impact:** HIGH · **Tags:** code-splitting, react-router, lazy-loading, suspense, performance

Load code only when it is needed. If the router references page components with static imports, the whole application (every page, every modal, every rarely visited admin screen) ends up in the entry chunk that every visitor must download before seeing anything.

### Why it matters

A massive initial bundle delays when the app becomes usable: the browser must download, parse, and execute code for pages the visitor may never open. Splitting by route keeps the first payload small, improves LCP for the entry screen, and reduces main-thread work that would otherwise compete with user interactions (INP).

### Incorrect

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

### Correct

#### React Router 8 data router: granular `route.lazy`

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

#### Lazy route discovery

In framework mode (`@react-router/dev`), route discovery is lazy by default: only routes needed for the initial render are in the manifest, and more are fetched from the manifest endpoint as the visitor navigates. `<Link>` and `<NavLink>` targets rendered on the current page are discovered in a batch ahead of time, so the click itself does not pay for the manifest round trip. Configure or disable it in `react-router.config.ts`, and opt individual links out with `<Link to="/admin" discover="none">`.

In data mode the equivalent primitive is the `patchRoutesOnNavigation` option of `createBrowserRouter`, called whenever no route matches a path so you can `patch` the missing subtree into the tree at runtime.

#### Transition-friendly navigation

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

#### Fallback without a data router: `React.lazy` + `Suspense`

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

### References

- [React Router route objects (`lazy`)](https://reactrouter.com/start/data/route-object#lazy)
- [React Router data mode / custom framework](https://reactrouter.com/start/data/custom)
- [Lazy route discovery](https://reactrouter.com/explanation/lazy-route-discovery)
- [React `lazy` reference](https://react.dev/reference/react/lazy)

---

## Separation of Server State

**Impact:** HIGH · **Tags:** server-state, tanstack-query, caching, data-fetching

Server state (data that lives in an API) behaves differently from client state (UI interactions). It needs caching, deduplication, background revalidation, and a lifecycle tied to the cache rather than to a component tree.

The examples below use `@tanstack/react-query` (the current 5.x line). SWR covers similar ground with a smaller API; either way, the point is to stop hand-rolling request lifecycles.

### Why it matters

Managing server data with `useEffect` and `useState` leads to:

- **Race conditions:** Responses arriving out of order, so a slow request for the previous id overwrites the fresh one.
- **Waterfalls:** Child components waiting for parents to finish fetching before they can even start.
- **Complexity:** Manually handling loading, error, empty, retry, abort, and refetch states, then repeating it in every component that needs the same data.

### Incorrect

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

### Correct

#### 1. Declare the query once with `queryOptions`

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

#### 2. Read it with the options-object `useQuery`

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

#### 3. Suspense: `useSuspenseQuery` behind a boundary

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

#### 4. Keep the previous page visible while the next one loads

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

#### 5. Know what the two timers do

- `staleTime`: how long data is considered fresh. While fresh, no refetch happens on mount, window focus, or reconnect.
- `gcTime`: how long an unused query stays in memory before it is garbage collected. It only starts counting once no component observes the query.

This is the current name for the old cache-time setting. After a mutation, invalidate the affected keys instead of editing the cache by hand:

```ts
queryClient.invalidateQueries({ queryKey: ['user', userId] });
```

### References

- [TanStack Query overview](https://tanstack.com/query/latest/docs/framework/react/overview)
- [Suspense guide](https://tanstack.com/query/latest/docs/framework/react/guides/suspense)
- [Query cancellation](https://tanstack.com/query/latest/docs/framework/react/guides/query-cancellation)
- [`queryOptions` reference](https://tanstack.com/query/latest/docs/framework/react/reference/queryOptions)

---

## Strategic Memoization

**Impact:** MEDIUM · **Tags:** memoization, react-compiler, performance, rendering

React is fast by default. Over-memoizing (`useMemo`, `useCallback`, `memo`) can hurt performance more than it helps, because every memoized value costs a dependency comparison and a cache entry, and because the comparison itself is easy to get wrong.

### Why it matters

- **Too little:** Expensive calculations and wide re-render cascades block the main thread and make interactions feel slow.
- **Too much:** Code becomes harder to read (`useCallback` hell) and the overhead of memoization outweighs the re-render cost of simple components.

### React Compiler changes the default

With React Compiler enabled (stable, `babel-plugin-react-compiler` 1.x), manual `memo`, `useMemo`, and `useCallback` are usually unnecessary. The compiler analyzes components and hooks at build time and inserts memoization itself: it skips cascading re-renders of children and caches expensive calculations that run during render.

In a Vite project, the compiler runs as a Babel preset in front of the pipeline. `@vitejs/plugin-react` 6+ exposes `reactCompilerPreset`, which you pass to `@rolldown/plugin-babel`:

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react, { reactCompilerPreset } from '@vitejs/plugin-react';
import babel from '@rolldown/plugin-babel';

export default defineConfig({
  plugins: [react(), babel({ presets: [reactCompilerPreset()] })],
});
```

What the compiler does *not* let you skip:

- **Effect dependency stability.** The React docs still recommend `useMemo`/`useCallback` as the escape hatch when a memoized value feeds an effect dependency array and the effect must not re-fire.
- **Existing code.** Leave existing memoization in place unless you test the change; removing it can change the compiler's output.
- **Work outside React.** The compiler only memoizes components and hooks, and its memoization is not shared between them. A genuinely expensive pure function called from many components still deserves its own cache (or a `useMemo` per call site that was profiled as hot).
- **Opt-outs.** `"use no memo"` at the top of a component skips compilation for it while you debug.

Projects without the compiler keep the guidance below.

### Incorrect

Memoizing everything blindly.

```tsx
// Incorrect: Primitive values and simple objects don't need memoization
const value = useMemo(() => 42, []);
const style = useMemo(() => ({ color: 'red' }), []);

// Incorrect: Inline functions on lightweight HTML elements are fine
const handleClick = useCallback(() => {
  console.log('clicked');
}, []);

return <button onClick={handleClick}>Click me</button>;
```

### Correct

Memoize only where a measurement or a hard requirement justifies it.

#### 1. Referential equality for dependencies

When a value or function is used in a `useEffect` dependency array, stability matters.

```tsx
const params = useMemo(() => ({ id: userId }), [userId]);

useEffect(() => {
  api.fetch(params);
}, [params]); // params must be stable to avoid re-running on every render
```

#### 2. Expensive calculations

Heavy synchronous work that runs on every render, such as filtering a 5,000 item list.

```tsx
const sortedList = useMemo(() => {
  return largeArray.filter((item) => item.active).sort((a, b) => a.value - b.value);
}, [largeArray]);
```

#### 3. Passing stable props to optimized children

If a child is wrapped in `memo`, a new function reference on every parent render breaks the optimization.

```tsx
import { memo, useCallback, useState } from 'react';

const HeavyComponent = memo(function HeavyComponent({
  onParamChange,
}: {
  onParamChange: (value: string) => void;
}) {
  return <input onChange={(event) => onParamChange(event.target.value)} />;
});

function Parent() {
  const [value, setValue] = useState('');

  const handleChange = useCallback((next: string) => {
    setValue(next);
  }, []);

  // memo() on HeavyComponent is only useful because handleChange is stable
  return <HeavyComponent onParamChange={handleChange} />;
}
```

Profile first. React DevTools' Profiler tells you whether a component re-renders often enough to matter, and the compiler's `Memo` badge tells you whether that component was already optimized automatically. Without either signal, plain code is the better default.

### References

- [React Compiler introduction](https://react.dev/learn/react-compiler/introduction)
- [React Compiler installation](https://react.dev/learn/react-compiler/installation)
- [`useMemo` reference](https://react.dev/reference/react/useMemo)
- [`memo` reference](https://react.dev/reference/react/memo)

---

## Image Optimization for CLS and Performance

**Impact:** MEDIUM · **Tags:** images, lcp, cls, responsive-images, performance

Images are usually the largest assets on a page. Handled carelessly they shift the layout when they arrive, they delay the Largest Contentful Paint because the browser discovers them late or at low priority, and they consume bandwidth that interactions need.

### Why it matters

- **CLS (Cumulative Layout Shift):** Without reserved space the page "jumps" when an image arrives, and users lose their place or click the wrong control.
- **LCP (Largest Contentful Paint):** The hero image is the LCP element on most content pages. Late discovery, lazy loading, or an oversized file pushes LCP past the 2.5s target.
- **INP (Interaction to Next Paint):** Every byte spent on images that are too large competes with the work an interaction needs in order to paint its update, especially on slow connections.

### Incorrect

An `img` with no dimensions, no priority hint, and no format or size alternatives. Lazy loading the hero image makes it worse.

```tsx
// Incorrect: the browser cannot reserve space, and the LCP image
// is discovered late and fetched at whatever priority it guesses
<img src="/hero.jpg" alt="Hero" loading="lazy" />
```

### Correct

#### The LCP image: high priority, sized, modern formats

```tsx
function Hero() {
  return (
    <picture>
      <source
        type="image/avif"
        srcSet="/hero-640.avif 640w, /hero-1280.avif 1280w, /hero-1920.avif 1920w"
        sizes="100vw"
      />
      <source
        type="image/webp"
        srcSet="/hero-640.webp 640w, /hero-1280.webp 1280w, /hero-1920.webp 1920w"
        sizes="100vw"
      />
      <img
        src="/hero-1280.jpg"
        srcSet="/hero-640.jpg 640w, /hero-1280.jpg 1280w, /hero-1920.jpg 1920w"
        sizes="100vw"
        alt="Team reviewing designs in the studio"
        width={1920}
        height={1080}
        fetchPriority="high"
        decoding="async"
        className="h-auto w-full"
      />
    </picture>
  );
}
```

- `fetchPriority="high"` (React's camelCase prop for `fetchpriority`) tells the browser that this image is the one that matters. Set it on one or two images at most; if everything is high priority, nothing is.
- Never put `loading="lazy"` on the LCP image: the browser only starts fetching it after layout decides it is near the viewport, which adds avoidable delay.
- `width` and `height` give the browser the intrinsic aspect ratio so it reserves space before the bytes arrive, which is what prevents CLS. `h-auto w-full` lets CSS keep that ratio while scaling.
- `<picture>` with AVIF and WebP sources is a fallback chain: modern browsers take the smallest format they support, and everyone else gets the JPEG in `<img>`. Keep the `<img>` `srcSet`/`sizes` so the fallback is also responsive.
- `sizes` must describe the real rendered width. A wrong `sizes` value defeats the whole `srcSet` negotiation and downloads the largest candidate.

React does not generate an automatic preload hint for an `<img>` inside `<picture>`, so if that hero is the LCP element, add the hint to the HTML shell yourself:

```html
<link
  rel="preload"
  as="image"
  href="/hero-1280.avif"
  type="image/avif"
  fetchpriority="high"
/>
```

#### Below-the-fold images: lazy, but still sized

```tsx
function ProductCard({ product }: { product: Product }) {
  return (
    <img
      src={product.image}
      srcSet={`${product.imageSmall} 640w, ${product.imageLarge} 1280w`}
      sizes="(min-width: 768px) 320px, 100vw"
      width={640}
      height={480}
      loading="lazy"
      decoding="async"
      alt={product.name}
      className="h-auto w-full"
    />
  );
}
```

`loading="lazy"` defers the request until the image approaches the viewport, which saves bandwidth and keeps early requests free for the critical path. It only pays off when the element has explicit dimensions: without them the browser treats the image as zero-sized, can believe everything fits on screen, and loads all of it anyway.

### References

- [Optimize Largest Contentful Paint](https://web.dev/articles/optimize-lcp)
- [Browser-level image lazy loading](https://web.dev/articles/browser-level-image-lazy-loading)
- [Serve responsive images](https://web.dev/articles/serve-responsive-images)
- [React `<img>` reference](https://react.dev/reference/react-dom/components/img)

---

# Architecture & Cleanup

Structure decides how expensive a change is. These rules cover the folder boundaries that keep a feature shippable on its own, the import paths that keep a bundle from containing the same code twice, and the recurring Vite failures that have a known fix.

## Component Colocation and Architecture

**Impact:** MEDIUM · **Tags:** react, architecture, react-router, colocation

Organize code by **Feature**, not by **Technology**. Avoid monolithic `components/` or `hooks/` folders that become dumping grounds.

### Why it matters
- **Discoverability:** Related code is easier to find.
- **Maintainability:** Deleting a feature is safe (delete one folder) vs hunting files across the codebase.
- **Reviewability:** A change that touches one feature folder is easy to scope, review, and revert.
- **Routing boundary:** Route entries stay thin — they map a URL to a feature and contain no business logic.

### Incorrect
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

### Correct
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

### Rules of thumb
- `features/*/index.ts` is the only entry other folders may import. Everything else inside a feature stays private.
- A feature must not import another feature's internals. Promote the shared piece to `components/` or to an explicit shared feature.
- A route entry contains at most a `loader` / `clientLoader`, an `ErrorBoundary`, and the feature component import.
- Route-level code splitting stays in the route entry (see [`rules/react-route-splitting.md`](rules/react-route-splitting.md)).
- If deleting a folder requires editing more than the router config, the boundary is wrong.

### References
- [React Router: Routing (Data mode)](https://reactrouter.com/start/data/routing)
- [React Router: Route Module](https://reactrouter.com/start/framework/route-module)
- [React: Thinking in React](https://react.dev/learn/thinking-in-react)
- [Vite: Features](https://vite.dev/guide/features)

---

## Anti-Pattern: Importing from Dist

**Impact:** MEDIUM · **Tags:** bundling, package-json, esm, dependencies

Never import files from the `dist/`, `build/`, or generated output folders of your libraries or your own project.

### Why it matters
- **Bundling double:** You might include the same library twice (once source, once bundled).
- **Optimization loss:** Pre-bundled code cannot be effectively tree-shaken by your Vite build.
- **Dev-Experience:** Source maps often break, making debugging impossible.
- **Brittle paths:** `dist/` layouts change between releases, and an `exports` map can make the path unresolvable.

### Incorrect
```tsx
import { Button } from 'expensive-library/dist/bundle.js';
import { Helper } from '../dist/utils.js';
```

### Correct
Import the source modules or the main package entry point.

```tsx
import { Button } from 'expensive-library'; // Resolved through package.json "exports"
import { Helper } from '../utils'; // Import source TS/JS
```

### Use `exports` maps, not file paths

The `"exports"` field is a package's declared public surface. When it exists, every subpath that is not listed is private: `import 'pkg/dist/index.js'` throws `ERR_PACKAGE_PATH_NOT_EXPORTED` in Node, and Vite cannot resolve it either.

```json
{
  "name": "expensive-library",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    },
    "./icons": "./dist/icons.js"
  }
}
```

- Import the package root for the public API: `import { Button } from 'expensive-library'`.
- Use a declared subpath for a smaller entry: `import { Icon } from 'expensive-library/icons'`.
- Never assemble a path into `dist/` by hand. If a subpath is missing, the library did not export it.
- Your own workspace code must import its source (`../utils`), never its own build output.
- In a monorepo, point at the source entry through `exports` or `resolve.alias` instead of `../lib/dist`.

### ESM vs CJS interop

Vite 8 (Rolldown) applies one consistent rule for the `default` import of a CommonJS module. It resolves to `module.exports` when the importer is `.mjs` / `.mts`, when the importer's nearest `package.json` has `"type": "module"`, or when the CJS module does not set `__esModule`. Otherwise it resolves to `module.exports.default`.

A mismatch shows up as `Element type is invalid: expected a string ... but got: object` or `foo is not a function`:

```tsx
// Incorrect: `legacy-chart` is CJS and does not set __esModule,
// so the default import is the whole module.exports object.
import Chart from 'legacy-chart';
```

```tsx
// Correct: use named exports, which are detected from the CJS module.
import { Chart } from 'legacy-chart';

// Or read `default` explicitly when the package provides module.exports.default.
import * as pkg from 'legacy-chart';
const ChartFromDefault = pkg.default;
```

- Prefer packages that ship an `"import"` condition. Vite resolves that branch for browser code.
- Do not deep-import a package's `.cjs` file to work around interop. That bypasses the `exports` map and re-introduces double bundling.
- Do not add `.cjs` files to browser code. Keep CommonJS for Node-only tooling.
- If a dependency is genuinely broken, the temporary escape hatch is `legacy.inconsistentCjsInterop`, which is deprecated and should not ship:

```ts
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  legacy: {
    inconsistentCjsInterop: true, // temporary; report the package instead
  },
});
```

### References
- [Node.js: Package entry points](https://nodejs.org/api/packages.html#package-entry-points)
- [Node.js: Subpath exports](https://nodejs.org/api/packages.html#subpath-exports)
- [Vite: Dependency Pre-Bundling](https://vite.dev/guide/dep-pre-bundling)
- [Rolldown: Bundling CJS](https://rolldown.rs/in-depth/bundling-cjs)

---

## Common Vite Troubleshooting

**Impact:** MEDIUM · **Tags:** vite, troubleshooting, hmr, react, hydration

Quick fixes for the most frequent issues encountered in Vite + React projects. Applies to Vite 8 (Rolldown + Oxc) and React 19.

### 1. A `default` import from a CJS package returns an object

**Cause:** CommonJS interop. Vite 8 resolves the `default` import of a CJS module to `module.exports` when the importer is ESM (`"type": "module"`, `.mjs`, `.mts`) or when the CJS module does not set `__esModule`. Otherwise it resolves to `module.exports.default`. The failing code usually renders `Element type is invalid: expected a string ... but got: object` or throws `foo is not a function`.

**Fix:** use named exports, a namespace import, or pre-bundle the dependency.

```tsx
// Incorrect: `legacy-chart` exposes module.exports, not module.exports.default
import Chart from 'legacy-chart';

// Correct: named exports are detected from the CJS module
import { Chart } from 'legacy-chart';

// Correct: read default explicitly
import * as pkg from 'legacy-chart';
const ChartFromDefault = pkg.default;
```

If the package itself is at fault, `legacy.inconsistentCjsInterop: true` restores the previous behavior. It is deprecated, so treat it as a temporary unblock and report the package.

### 2. "Module is external" or "Cannot find module"

**Cause:** the dependency ships CJS only, has no usable entry in its `exports` map, or is a linked local package whose pre-bundled output is stale.

**Fix:** add the entry to `optimizeDeps.include` and force re-optimization.

```ts
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  optimizeDeps: {
    include: ['broken-lib', 'broken-lib/deep-entry'],
    // Vite 8 pre-bundles with Rolldown.
    // optimizeDeps.esbuildOptions is deprecated; migrate to rolldownOptions.
    rolldownOptions: {
      // rolldown input/output options, only if you need them
    },
  },
});
```

```bash
# Linked packages (npm link, workspace symlink) are not invalidated automatically
npm run dev -- --force
```

Note that `build.commonjsOptions` is a no-op in Vite 8, so it is never the fix.

### 3. A plugin breaks after the Rolldown / Oxc switch

**Cause:** Vite 8 replaced Rollup with Rolldown and esbuild with Oxc. Plugins that depend on Rollup-specific hooks or on esbuild being installed can fail at config load or at build time.

**Fix:** check the plugin for a Vite 8 / Rolldown release, then migrate the parts you own:

- `build.rollupOptions` is now `build.rolldownOptions` (the old key is deprecated).
- `transformWithEsbuild` is deprecated; use `transformWithOxc`, which is exported from `vite`.
- `esbuild` is now an optional dependency rather than a bundled one. If a plugin calls `transformWithEsbuild`, install `esbuild` yourself as a `devDependency`.
- `load` / `transform` hooks that turn a non-JS file into JS must return `moduleType: 'js'`, because Rolldown infers the module type from the resolved file extension.
- The object form of `output.manualChunks` was removed. Use Rolldown's `codeSplitting`, or the deprecated function form as an interim step.
- These hooks are not supported by Rolldown: `shouldTransformCachedModule`, `resolveImportMeta`, `renderDynamicImport`, `resolveFileUrl`.
- Parallel Rollup hooks execute sequentially under Rolldown. A plugin that assumed concurrency can deadlock or produce different output.

```ts
// A load hook that emits JavaScript from a non-JS file must declare the module type.
const txtLoader = {
  name: 'txt-loader',
  load(id: string) {
    if (!id.endsWith('.txt')) return;
    return {
      code: `export default ${JSON.stringify('...')}`,
      moduleType: 'js',
    };
  },
};
```

### 4. HMR Not Working (Full Reload on Save)

**Cause:**
1. **Circular Dependency:** A imports B, B imports A. HMR is handled but the runtime cannot recover execution order, so Vite falls back to a full reload.
2. **Mixed exports:** a module that exports both components and non-component values can defeat React Fast Refresh and force a full reload.
3. **Case Sensitivity:** importing `File.tsx` as `file.tsx` creates a second module ID. It works on macOS and Windows, then fails on Linux with `ENOENT` or `Module not found`.

**Fix:** log the actual cycle, then break it.

```bash
vite --debug hmr
```

The debug output prints the circular dependency path. Move the shared value into a third module instead of importing it back, and keep component files exporting only components.

### 5. React 19 hydration mismatch (`#418` / `#423`)

**Cause:** the HTML sent to the browser does not match what React renders on the first client pass. In production React prints the minified code:

- `#418` — "Hydration failed because the server rendered HTML didn't match the client. As a result this tree will be regenerated on the client."
- `#423` — "There was an error while hydrating but React was able to recover by instead client rendering the entire root."

`#423` is the same root problem, but the error escaped the component that caused it, so React discards the whole root instead of one subtree.

Common triggers:

1. **Invalid HTML nesting.** `<div>` inside `<p>`, `<a>` inside `<a>`, `<tr>` outside `<tbody>`. The browser rewrites the DOM before React hydrates, so the trees differ.
2. **Browser-only APIs during render.** `window`, `localStorage`, `matchMedia`, `Date.now()`, `Math.random()`.
3. **Mismatched text, date, or number rendering.** `toLocaleString()`, timezone-dependent dates, and currency formatting that differ between build machine and browser.
4. **External data that changed between render passes**, such as a cached API response replayed during hydration.
5. **Browser extensions** that mutate the DOM before React loads.

```tsx
// Incorrect: browser-only value during render, plus invalid <div> inside <p>
export function Price({ cents }: { cents: number }) {
  return (
    <p>
      <div>{new Date().toLocaleDateString()} — {(cents / 100).toFixed(2)} USD</div>
    </p>
  );
}
```

```tsx
// Correct: deterministic first render, browser-only value after mount
import { useEffect, useState } from 'react';

export function Price({ cents }: { cents: number }) {
  const [today, setToday] = useState<string | null>(null);

  useEffect(() => {
    setToday(new Date().toLocaleDateString());
  }, []);

  return (
    <p>
      <span>{today ?? '—'}</span> {(cents / 100).toFixed(2)} USD
    </p>
  );
}
```

**Fix:** render the same deterministic output on both sides, move browser reads into `useEffect`, pass an explicit `locale` and `timeZone` to formatters, and repair the nesting. Use `suppressHydrationWarning` only on a leaf text node that is legitimately allowed to differ, such as a relative timestamp.

This only applies when something is hydrated: React Router framework mode with SSR, or a prerender / SSG step. A purely client-rendered Vite SPA has no server HTML to mismatch.

### 6. Styles Missing in Production

**Cause:** CSS imported through a dynamically constructed path, or a stylesheet that only exists in a lazily loaded chunk the bundler cannot trace statically.

**Fix:** keep CSS imports static and at module top level, so they join the module graph and are emitted with the chunk that uses them. Use `?inline` only when you actually want the CSS as a string instead of an injected style.

### 7. 404 on Refresh

**Cause:** missing client-side routing fallback.
**Fix:** configure rewrites on your static host (see [`rules/vite-spa-rewrites.md`](rules/vite-spa-rewrites.md)).

### References
- [Vite: Troubleshooting](https://vite.dev/guide/troubleshooting)
- [Vite: Migration from v7 (Vite 8 / Rolldown)](https://vite.dev/guide/migration)
- [React: hydrateRoot](https://react.dev/reference/react-dom/client/hydrateRoot)
- [React: Minified error #418](https://react.dev/errors/418)
