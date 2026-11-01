---
title: Anti-Pattern: Importing from Dist
impact: MEDIUM
tags: bundling, package-json, esm, dependencies
---

# Anti-Pattern: Importing from Dist

Never import files from the `dist/`, `build/`, or generated output folders of your libraries or your own project.

## Why it matters
- **Bundling double:** You might include the same library twice (once source, once bundled).
- **Optimization loss:** Pre-bundled code cannot be effectively tree-shaken by your Vite build.
- **Dev-Experience:** Source maps often break, making debugging impossible.
- **Brittle paths:** `dist/` layouts change between releases, and an `exports` map can make the path unresolvable.

## Incorrect
```tsx
import { Button } from 'expensive-library/dist/bundle.js';
import { Helper } from '../dist/utils.js';
```

## Correct
Import the source modules or the main package entry point.

```tsx
import { Button } from 'expensive-library'; // Resolved through package.json "exports"
import { Helper } from '../utils'; // Import source TS/JS
```

## Use `exports` maps, not file paths

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

## ESM vs CJS interop

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

## References
- [Node.js: Package entry points](https://nodejs.org/api/packages.html#package-entry-points)
- [Node.js: Subpath exports](https://nodejs.org/api/packages.html#subpath-exports)
- [Vite: Dependency Pre-Bundling](https://vite.dev/guide/dep-pre-bundling)
- [Rolldown: Bundling CJS](https://rolldown.rs/in-depth/bundling-cjs)
