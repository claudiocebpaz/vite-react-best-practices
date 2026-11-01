---
title: Common Vite Troubleshooting
impact: MEDIUM
tags: vite, troubleshooting, hmr, react, hydration
---

# Common Vite Troubleshooting

Quick fixes for the most frequent issues encountered in Vite + React projects. Applies to Vite 8 (Rolldown + Oxc) and React 19.

## 1. A `default` import from a CJS package returns an object

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

## 2. "Module is external" or "Cannot find module"

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

## 3. A plugin breaks after the Rolldown / Oxc switch

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

## 4. HMR Not Working (Full Reload on Save)

**Cause:**
1. **Circular Dependency:** A imports B, B imports A. HMR is handled but the runtime cannot recover execution order, so Vite falls back to a full reload.
2. **Mixed exports:** a module that exports both components and non-component values can defeat React Fast Refresh and force a full reload.
3. **Case Sensitivity:** importing `File.tsx` as `file.tsx` creates a second module ID. It works on macOS and Windows, then fails on Linux with `ENOENT` or `Module not found`.

**Fix:** log the actual cycle, then break it.

```bash
vite --debug hmr
```

The debug output prints the circular dependency path. Move the shared value into a third module instead of importing it back, and keep component files exporting only components.

## 5. React 19 hydration mismatch (`#418` / `#423`)

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

## 6. Styles Missing in Production

**Cause:** CSS imported through a dynamically constructed path, or a stylesheet that only exists in a lazily loaded chunk the bundler cannot trace statically.

**Fix:** keep CSS imports static and at module top level, so they join the module graph and are emitted with the chunk that uses them. Use `?inline` only when you actually want the CSS as a string instead of an injected style.

## 7. 404 on Refresh

**Cause:** missing client-side routing fallback.
**Fix:** configure rewrites on your static host (see `vite-spa-rewrites.md`).

## References
- [Vite: Troubleshooting](https://vite.dev/guide/troubleshooting)
- [Vite: Migration from v7 (Vite 8 / Rolldown)](https://vite.dev/guide/migration)
- [React: hydrateRoot](https://react.dev/reference/react-dom/client/hydrateRoot)
- [React: Minified error #418](https://react.dev/errors/418)
