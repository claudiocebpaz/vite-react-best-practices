---
title: Validate Production Builds Locally
impact: HIGH
tags: vite, build, ci, typescript, deployment
---

# Validate Production Builds Locally

Vite 8 builds with Rolldown and transforms JavaScript with Oxc, replacing Rollup and esbuild. The dev server serves source modules on demand and pre-bundles dependencies with Rolldown, while `vite build` runs the full bundling, transformation, and minification pipeline. Development and production are therefore different programs operating on the same source.

## Why it matters
Problems that survive `npm run dev` but break the deployed app include:
- Case-sensitive import paths that resolve on a case-insensitive filesystem but not on Linux CI.
- Files referenced from `public/` with a wrong base path, or assets missing from `dist/`.
- Code that only fails after minification or tree-shaking (side-effect imports removed, `process.env` left untouched).
- Builds that ran in a different mode, and therefore inlined a different set of `import.meta.env` values.

`vite build` does not typecheck. Without a separate `tsc` run, type errors ship to production.

## Incorrect
Verifying only the dev server, then pushing.

```bash
# Don't do this only
npm run dev
git commit -m "It works on my machine"
git push
```

## Correct
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

### Minimal CI workflow
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

## References
- [Vite: Building for Production](https://vite.dev/guide/build)
- [Vite: Command Line Interface](https://vite.dev/guide/cli)
- [Vite: Migration from v7 (Rolldown and Oxc)](https://vite.dev/guide/migration)
- [GitHub Actions: Workflow syntax](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions)
