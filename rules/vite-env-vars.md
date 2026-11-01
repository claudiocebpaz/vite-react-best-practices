---
title: Environment Variables in Vite
impact: HIGH
tags: vite, env, configuration, security, typescript
---

# Environment Variables in Vite

Vite exposes configuration on `import.meta.env`. Values from `.env` files are parsed at build time and statically replaced in the output, so the browser bundle contains literals, not a lookup table. Only `VITE_`-prefixed variables from those files reach client code.

## Why it matters
- **Secret leakage.** Anything prefixed with `VITE_` is bundled into JavaScript that every visitor can read. An API key placed there is published.
- **Stale expectations.** Values are inlined when the bundle is built. Changing a variable in the hosting dashboard does nothing until the app is rebuilt, which surprises teams used to server-side runtimes.

## Incorrect
Using `process.env`, or expecting an unprefixed variable to be available in the browser.

```tsx
// Incorrect: Vite does not polyfill process.env for client code
console.log(process.env.API_KEY)

// Incorrect: not exposed, because it lacks the VITE_ prefix
console.log(import.meta.env.SECRET_KEY)
```

## Correct
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

### `.env` files and modes
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

### Built-in flags
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

### Runtime configuration
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

### Type safety
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

## References
- [Vite: Env Variables and Modes](https://vite.dev/guide/env-and-mode)
- [Vite: Shared Options (`envPrefix`)](https://vite.dev/config/shared-options)
- [MDN: import.meta](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import.meta)
